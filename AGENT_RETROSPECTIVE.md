# Text-to-SQL Agent Retrospective



This project used LangGraph and LangChain to route construction-site questions to domain-specific agents such as **process**, **cost**, and **quality**.

Questions were first split and classified. The router sent process questions to the Process Agent and cost questions to the Cost Agent. When a question covered both domains, the agents ran in parallel and the final Assistant combined their results.

I was responsible for the **process** and **cost** domains.



## 1. Process and cost were not RAG problems

These agents did not answer questions by retrieving documents. They queried **daily construction report data stored in MySQL**.

```
User question
    ↓
BasicEntities extraction
    ↓
Structured frame
    ↓
Text-to-SQL
    ↓
run_sql()
    ↓
Query result
    ↓
Assistant
    ↓
Final answer
```

The first step used `BasicEntities` to extract a structured frame from the question:


| Category    | Fields                                                                                     |
| ----------- | ------------------------------------------------------------------------------------------ |
| **Target**  | `focus_table`                                                                              |
| **Filters** | site / section · subcontractor · work area / process · equipment / occupation · date range |
| **Output**  | chart type · report intent · delay-analysis intent · time-series unit                      |


The frame was used to select an SQL prompt, and the Text-to-SQL agent generated the query.

SQL execution was kept outside the agent and handled separately by `run_sql()`. Keeping generation and execution apart made execution easier to control and avoided unnecessary agent tool calls.

  


## 2. The frame did not actually enforce the SQL

The idea behind the frame was sound: if the system already knows the date range, site, company, and target table before generating SQL, the SQL should be more stable.

The problem was **how I used it**. The frame was passed to the model as system context, under something like *Structured intent info*. For example, these values might already be known:

```
date_from = 2026-09-01
date_to   = 2026-09-30
```

But nothing guaranteed they would appear in the final `WHERE` clause. The model still had to read the frame and translate it into `WHERE date BETWEEN ...` on its own.

> **The frame was reference information, not a constraint.**

That difference mattered more than I expected.

  


## 3. temperature=0 did not make the SQL reliable

The Text-to-SQL output still varied. Even with `temperature=0`, the model sometimes:

- selected the wrong table
- omitted filters
- produced aggregations that didn't match the question

My response was to add more prompt rules, and the prompts gradually split into cases:


| Domain      | Prompt variants                                                                                |
| ----------- | ---------------------------------------------------------------------------------------------- |
| **Process** | Gantt chart · S-curve · progress bar · delays · work details · reports · workforce · equipment |
| **Cost**    | workforce · equipment · subcontractor workforce · subcontractor equipment                      |


Eventually, even column names were heavily constrained in the prompt.

This made some outputs more consistent. But past a certain point, the model seemed less like it was reading the schema and more like it was **copying query templates**. `temperature=0` simply made that behavior more consistent.

  


## 4. I exposed too much of the schema

The database structure also contributed. Daily-report data was spread across several tables and views:

```
daily_report · workforce · workforce_by_company · equipment · site · process · cost-related views
```

`SQLDatabase` was configured with `view_support=True`, so both tables and views were visible to the agent.

The key mistake: **I extracted** `focus_table` **but never used it to restrict the schema** available to the SQL agent. The model still had to choose among many tables and views.

I had also added column comments to the database. However:

- Table-listing tools mostly returned **names only**.
- Comments became visible only when the model explicitly inspected a table and received its `CREATE TABLE` definition.
- Meanwhile, the prompts already contained strong rules such as *"prefer* `v_`* *views"* and *"use these columns."*

So the model wasn't really ignoring the comments. **The prompt had usually made the decision before the comments were ever seen.**

  


## 5. The token problem was not caused by long answers

At first, I assumed token usage was high because the Assistant's responses were too long. The bigger issue was **the SQL result itself**.

When a query returned detailed rows instead of an aggregation, a full month of records could come back. `prepare_result_contexts` converted those results into Markdown for the Assistant, and anything over 15,000 characters was truncated.

```
Detailed SQL
    ↓
Many rows
    ↓
Markdown table
    ↓
Over 15,000 characters
    ↓
Truncation
    ↓
Assistant summary
```

The Assistant was then asked to summarize an **incomplete table**, so weak answers were not surprising.

And by then, the system had already spent time and tokens on entity extraction, SQL agent reasoning, schema inspection, and multiple tool calls. **The cost was paid before the final answer was even generated.**

  


## 6. Separating detail and aggregate queries

For cost queries, I introduced two versions:


| Version | Purpose                            |
| ------- | ---------------------------------- |
| **A**   | Detailed data for frontend display |
| **B**   | Aggregated data for the Assistant  |


The idea was reasonable. But if Version B still produced a detailed query instead of a real `GROUP BY`, the separation didn't help.

> **The real bottleneck was not the A/B structure. It was the quality of SQL generation itself.**

  


## 7. What "good" Text-to-SQL means here

For this system, a good query was not just syntactically valid SQL. It needed to:

- use the date range extracted in the frame
- apply the correct site or project filter
- distinguish detailed frontend queries from summary queries
- avoid mixing quantities with unit prices
- avoid returning unnecessary detailed rows

When those conditions were met, a monthly summary fit into a few aggregated rows. There was no need to truncate a 15,000-character result or ask the Assistant to reason over a huge table.

If the SQL was wrong, everything after it had limited value:

- A formatter could only make the wrong data easier to read.
- The Assistant could only turn the wrong result into a nicer sentence.

  


## 8. What I would change

If I rebuilt the system, I would not start by adding more prompt rules. Instead, **values already known from the structured frame would be enforced in code**.


| Handled by | Responsibilities                                                      |
| ---------- | --------------------------------------------------------------------- |
| **Code**   | Allowed tables · `FROM` · `WHERE` · date range · project/site filters |
| **LLM**    | `SELECT` · aggregation · `GROUP BY` · required metrics                |


There is little reason to ask the model whether to apply a date or site filter when those values are already known.

I would also:

- **Restrict the schema by** `focus_table`**.** Rather than showing the whole database and hoping the model picks correctly, reduce the choices up front.
- **Provide column descriptions directly with the selected schema**, instead of relying on the model to decide whether to inspect metadata.

  


## 9. How I would test it differently

At the time, several things were changing at once: the model, prompts, templates, schema, comments, and views. That made it hard to know what actually improved or degraded the SQL.

If I repeated the experiment, I would **change one variable at a time**:

1. Model only
2. Prompt length only
3. Schema scope only
4. Comment visibility only

That would make it much easier to identify what actually drives Text-to-SQL quality.

  


## 10. Final thoughts

The main lesson: **trying to control everything through prompts makes the prompts grow indefinitely.**

If the application already knows something, there is little benefit in asking the model to decide it again.

> **Code should handle what can be determined reliably. The model should handle what actually requires interpretation.**

Before trying to improve the final Assistant response, I should have focused on making SQL generation reliable. The Assistant could never be more accurate than the data it was given.