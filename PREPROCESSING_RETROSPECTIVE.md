# Preprocessing Pipeline Retrospective

This project used LangGraph and LangChain to route construction-site questions to domain-specific agents. I was responsible for the **process** and **cost** domains, and both depended on data stored in a relational database.

That meant there was work to finish before the agents themselves: parsing the Excel reports, structuring the data, designing the schema, and loading everything into the database.

**The RDB came before the agents.**

## 1. First approach: the whole sheet at once

Construction daily-report Excel files were far from ordinary tabular data. Table positions were not fixed, and a single sheet could contain several different tables:

- Process status
- Work details
- Workforce
- Equipment
- Materials
- Workforce by subcontractor
- Equipment by subcontractor

My first approach was simple:

```
Excel → Markdown → LLM (with_structured_output) → JSON
```

**It did not scale.**

- Some sheets were too large to fit into the context window.
- Splitting sheets into left and right halves didn't help much, because each half could still contain several tables.
- Converting sheets into DataFrames and splitting rows using openpyxl formatting information still failed on long tables.

At that point I realized that reducing the input size was not enough.

> **The unit of processing had to be a table, not a sheet.**



## 2. Extracting individual tables

Most reports had visible cell borders around their tables. I used those borders to build a cell mask, then grouped connected cells into individual blocks.

```
Excel sheet
    ↓
Border mask
    ↓
Connected cells
    ↓
Table blocks
```

Once the blocks were extracted, the next question was: **what kind of table is this?** The target schema depended on whether a block represented process status, workforce, equipment, materials, or something else.


| Case                           | Approach                                                                                                           |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| Obvious tables                 | Keyword rules                                                                                                      |
| Ambiguous tables               | LLM classification                                                                                                 |
| Tables mixing several concepts | Combined categories, e.g. *process + work details*, *workforce + equipment*, *subcontractor workforce + equipment* |


Applying deterministic rules first and using the LLM only for ambiguous cases was **much more stable** than asking the model to classify everything.

## 3. Headers that didn't work as headers

With the table type known, the next step was mapping Excel columns to database fields. This was harder than expected, because the headers often didn't carry enough information.

For example, several columns could share exactly the same label:


| Process Status       | Process Status | Process Status                       | Process Status |
| -------------------- | -------------- | ------------------------------------ | -------------- |
| Yeongdeungpo Station | Exit #1        | Temporary Structure / Vertical Shaft | Excavation     |


The values underneath formed a hierarchy. A person can infer it from context, but **the model could not reliably tell what each repeated column represented.**

My first schema split them into separate fields:

- `site_level_1`
- `site_level_2`
- `process`
- `detailed_process`

For some combined tables, I simplified this further to just `site` and `process`.

**Looking back,** I would model it as a single, meaningful label:

```
Yeongdeungpo Station / Exit #1 / Temporary Structure / Excavation
```

After deduplication, the number of distinct process names was not that large. Combining the hierarchy into one label would likely have simplified both the database and the Text-to-SQL layer.

## 4. A flexible schema became a burden for Text-to-SQL

I initially designed a fairly normalized schema:

```
Project · File · Daily Report · Site · Process · Progress · Work Detail
Workforce · Equipment · Company · Company Workforce · Company Equipment · Material
```

I also kept additional combined schemas for mixed Excel layouts.

At first this felt right: responsibilities were cleanly separated and the structure was flexible. The downside became obvious once I started on Text-to-SQL. More tables meant:

- more join paths
- more ambiguity about which table to use
- more possible interpretations of the same question

> **A schema designed for extensibility is not necessarily a schema designed for an LLM to query.**

If Text-to-SQL is a major access pattern, fewer tables and clearer query paths may be the better trade-off.

## 5. Precomputing anomaly flags

I added an anomaly flag (`0` / `1`) to work-detail records. Day and night work descriptions were processed in batches, and a record was flagged if it mentioned situations such as:

- delays
- waiting
- rain
- missing materials
- other work disruptions

Already-processed rows were tracked so they would not be analyzed again.

**I would keep this design.** When users repeatedly ask questions like *"Were there any delayed activities?"*, classifying records once during preprocessing is far more efficient than sending every work description to an LLM at query time.

## 6. What I missed: hidden rows

The extraction logic mainly looked at **cell values** and **cell borders**. As a result, rows hidden in the workbook were still extracted if they had values and formatting.

Some processes that had been intentionally hidden in the original report ended up being treated as valid progress or work-detail records.

It was a reminder that Excel parsing is not only about visible values. All of these can change what a document means:

- hidden rows
- merged cells
- formatted but empty cells
- formula cells



## 7. What I would do differently

If I rebuilt the pipeline today, I would keep the overall direction but make each layer's responsibilities explicit.


| Layer      | Responsibilities                                                                                                       |
| ---------- | ---------------------------------------------------------------------------------------------------------------------- |
| **Python** | Table detection · hidden-row filtering · merged-cell handling · table splitting · data normalization                   |
| **Rules**  | Obvious table classification · straightforward header matching                                                         |
| **LLM**    | Ambiguous table classification · irregular header interpretation · semantic mappings that are hard to express as rules |


At the beginning, I expected the LLM to handle most of the messy Excel structure for me. What worked better was almost the opposite:

> **Handle everything deterministic with code first, and use the LLM only where semantic interpretation is actually needed.**

That was probably the most useful lesson from this part of the project.