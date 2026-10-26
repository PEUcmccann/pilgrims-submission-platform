# Snowflake AI extraction for Attachments (AWS eu-west-2)

Research for issue #8. The question: what can the Snowflake AI functions (AI_EXTRACT, AI_PARSE_DOCUMENT, AI_COMPLETE) do for the pilot's Attachments (feed planning documents and third-party logistics documents) in account `JBSUSA-PILGRIMS_EUROPE_PROD` (region `AWS_EU_WEST_2`, role `ENGINEER_ROLE`)?

Researched 2026-10-26. Sources: Snowflake documentation (linked inline), read-only SQL in the account, and two small AI_EXTRACT calls on a text string. No Snowflake objects were created or changed.

## Summary

- **Confidence Score: yes, per field.** AI_EXTRACT returns a score between 0 and 1 for each extracted field when called with `scores => TRUE`. This costs nothing extra. Lists and tables get one aggregate score each, not one per item or cell. The smoke test confirmed this works in this account.
- **Cross-region inference is required and already on.** AI_EXTRACT does not run natively in London, so it needs cross-region inference. The account has `CORTEX_ENABLED_CROSS_REGION = AWS_EU`, set explicitly at ACCOUNT level. AI_EXTRACT calls from this account succeed under that setting.
- **Excel and CSV are not supported as files** by AI_EXTRACT, AI_PARSE_DOCUMENT or AI_COMPLETE. PDF (native and scanned), images, DOCX, PPTX, HTML, TXT and EML are supported. Excel and CSV Attachments need a different route (see [Formats outside the AI functions](#formats-outside-the-ai-functions)).
- **Fixed schema per Document Type:** AI_EXTRACT takes a JSON-schema `responseFormat` that supports string, list and table fields. AI_COMPLETE supports full JSON-schema structured output, but returns no confidence value.
- **Cost:** AI_EXTRACT bills about 970 tokens per page. This account was billed 5.55 credits per million tokens, which is about 0.0054 credits per page. A pilot of 500 Attachments a month averaging 3 pages comes to roughly 9 credits a month for AI_EXTRACT, plus warehouse time.

## 1. Supported file types, page and size limits

| | AI_EXTRACT | AI_PARSE_DOCUMENT | AI_COMPLETE (single image) |
|---|---|---|---|
| File types | PDF, PNG, JPEG/JPG, TIF/TIFF, BMP, GIF, WEBP, DOC/DOCX, PPT/PPTX, EML, HTM/HTML, TXT, MD | PDF, PPTX, DOCX, JPEG/JPG, PNG, TIFF/TIF, HTML, TXT | JPG/JPEG, PNG, GIF, WEBP (BMP for pixtral/llama4 only) |
| Excel / CSV | Not supported | Not supported | Not supported |
| Max file size | 100 MB | 100 MB | 10 MB for most models; 3.75 MB for Claude models |
| Max pages | 125 per document (fewer if `scale_factor` > 1, e.g. 62 at 2.0) | 2,000 per document | One image per call |
| Max page size | 10000x10000 px; 50x50 in | 10000x10000 px; 50x50 in | Claude: 8000x8000 px max |
| Question limit | 100 entity questions per call; a table question counts as 10, and at most 10 table questions | n/a | n/a |
| Output limit | 512 tokens per entity answer; 4,096 tokens per table | n/a | `max_tokens` (default 4,096) |
| Scanned docs | Yes ("optimized for both digital-born and scanned documents"); handwriting, checkboxes and signatures | Yes; OCR mode or LAYOUT mode (Markdown with tables) | Image understanding by model |
| Stage encryption | Client- or server-side | Client- or server-side | Server-side only |

Sources: [AI_EXTRACT](https://docs.snowflake.com/en/sql-reference/functions/ai_extract), [AI_PARSE_DOCUMENT user guide](https://docs.snowflake.com/en/user-guide/snowflake-cortex/parse-document), [AI_COMPLETE (Single image)](https://docs.snowflake.com/en/sql-reference/functions/ai_complete-single-file).

Other notes:

- AI_EXTRACT has a `text =>` input as well as `file =>`. Text input is how any content that is not a supported file (CSV text, Excel cells turned into text, email body) can still be extracted with scores. The AI_EXTRACT context window is 128,000 tokens ([regional availability / model limits](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql-regional-availability)).
- For large tables, Snowflake advises splitting multi-page tables into one-page documents, because of the 4,096-token table output cap ([document extraction guide](https://docs.snowflake.com/en/user-guide/snowflake-cortex/document-extraction)).
- `scale_factor` (1.0–4.0) improves OCR on small text or large pages. It multiplies the token cost and divides the page limit.
- Snowflake recommends a warehouse no larger than MEDIUM. Larger warehouses do not run these functions faster.

### Formats outside the AI functions

None of the three functions accepts XLSX/XLS or CSV as a file. Options for those Attachments, not tested here:

- **CSV:** load it as structured data (COPY INTO / a schema per Document Type). If the layout varies, pass the text to `AI_EXTRACT(text => ...)` for scored Extraction.
- **Excel:** convert it first, for example with a Python UDF/procedure that reads the staged file (openpyxl) and either produces rows directly or produces text for `AI_EXTRACT(text => ...)`. Converting it to PDF outside Snowflake is another route.
- Structured CSV/Excel loaded directly has no model confidence. The platform would need a rule-based Confidence Score for those Document Types, such as schema validation pass/fail.

## 2. Confidence value (the platform's Confidence Score)

**AI_EXTRACT returns a per-field score.** Use the named-argument form with `scores => TRUE`. The result gains a `scoring.scores` object with one `score` (0–1) per field in `response` ([AI_EXTRACT: Extraction scores](https://docs.snowflake.com/en/sql-reference/functions/ai_extract#extraction-scores)):

```json
{
  "response": { "date": "2022-04-01", "name": "Johnson" },
  "scoring":  { "scores": { "date": {"score": 0.96}, "name": {"score": 0.99} } },
  "error": null
}
```

From the docs:

- "A higher score indicates a higher likelihood that the extracted value is correct". Snowflake suggests using the scores for "thresholds ... fallback mechanisms, and human-in-the-loop workflows". That is the Confidence Threshold and Review Queue design.
- "Requesting scores does not incur additional cost."
- "Per-element scores for individual list items and table cells are not available." Lists and tables get one aggregate score each.
- Scores are also returned by fine-tuned arctic-extract models.
- There is **no per-document score**. The platform must derive one from the per-field scores, for example the minimum over required fields. That rule is a design decision, not something Snowflake provides.

**Verified in this account** (query `01c7fd62-0004-9203-0001-023e1aa0ad22`):

```sql
SELECT AI_EXTRACT(
  text => 'Feed plan for Farm 12, week 44: 18.5 tonnes of grower feed delivered by Hauliers Ltd on 2026-10-28. Loads: L1 9.0t, L2 9.5t.',
  responseFormat => {'schema': {'type':'object','properties': {
     'farm': {'type':'string','description':'Farm name'},
     'tonnes': {'type':'string','description':'Total tonnes of feed'},
     'delivery_date': {'type':'string','description':'Delivery date'},
     'hauliers_po': {'type':'string','description':'What is the purchase order number?'},
     'loads': {'type':'array','description':'Load identifiers'}}}},
  scores => TRUE);
```

Result:

```json
{"error":null,
 "response":{"delivery_date":"2026-10-28","farm":"Farm 12","hauliers_po":"None","loads":["L1","L2"],"tonnes":"18.5"},
 "scoring":{"scores":{"delivery_date":{"score":0.966},"farm":{"score":0.830},"hauliers_po":{"score":0.966},"loads":{"score":0.918},"tonnes":{"score":0.915}}}}
```

**Caveat found in testing:** a field that is not in the source comes back as the *string* `"None"`, and it can still score high (0.97 above). The score measures how certain the model is of its answer, and "not present" counts as an answer. The platform must treat `"None"` as missing. A missing required field should send the Version to the Review Queue whatever its score.

**AI_COMPLETE and AI_PARSE_DOCUMENT return no confidence value.** AI_COMPLETE with `show_details => TRUE` returns token usage only ([AI_COMPLETE structured outputs](https://docs.snowflake.com/en/sql-reference/functions/ai_complete-structured-outputs)). AI_PARSE_DOCUMENT returns content and `metadata.pageCount` ([AI_PARSE_DOCUMENT](https://docs.snowflake.com/en/sql-reference/functions/ai_parse_document)). If AI_COMPLETE is used, other ways to get a Confidence Score are:

- route those fields through AI_EXTRACT (on the file, or on parsed text with `text =>`) to get scores
- rule-based checks (schema validation, required fields present, cross-field consistency such as load tonnages summing to the total, lookups against reference data)
- run two models and score by agreement
- ask the LLM for a self-rated confidence. Snowflake does not document this as calibrated, so it is the weakest option.

## 3. Model availability in AWS eu-west-2 and cross-region inference

**Account setting (read-only SQL, run 2026-10-26):**

```
SHOW PARAMETERS LIKE 'CORTEX_ENABLED_CROSS_REGION' IN ACCOUNT;
key=CORTEX_ENABLED_CROSS_REGION  value=AWS_EU  default=DISABLED  level=ACCOUNT
```

`level=ACCOUNT` means someone set the value on this account. It is not inherited. A value set explicitly on the account is not changed by the 2026_06 behaviour change that defaults London accounts to `AWS_EU` ([BCR-2376](https://docs.snowflake.com/en/release-notes/bcr-bundles/2026_06/bcr-2376)). Only ACCOUNTADMIN can change the parameter ([cross-region inference](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cross-region-inference)).

`AWS_EU` keeps inference "within designated cloud provider regions" (AWS, Europe). Data stays on the AWS private backbone, the payload is not persisted in the processing region, and credits are billed in the account's region with no egress charge ([cross-region inference](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cross-region-inference)).

**Native availability in AWS Europe West 2 (London):**

| Function / model | Native in London? | Under `AWS_EU` |
|---|---|---|
| AI_EXTRACT (`arctic-extract`) | **No.** Native AWS EU regions are Ireland and Frankfurt | Works (verified in this account: two AI_EXTRACT calls ran under this setting) |
| AI_PARSE_DOCUMENT | **Yes**, London is listed | Works (account usage shows 433 pages in LAYOUT mode over the last 90 days) |
| AI_COMPLETE `claude-sonnet-4-6`, `claude-opus-4-6` | **Yes**, the only native London models in the Europe table | n/a |
| AI_COMPLETE other models (e.g. `llama3.1-8b`, `mistral-large2`) | No, they are native in Frankfurt/Ireland | Available through `AWS_EU` (not tested here) |

Sources: [AI_EXTRACT regional availability](https://docs.snowflake.com/en/sql-reference/functions/ai_extract#regional-availability), [AI_PARSE_DOCUMENT regional availability](https://docs.snowflake.com/en/user-guide/snowflake-cortex/parse-document#regional-availability), [Models and regional availability](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql-regional-availability).

**Consequence:** Extraction with AI_EXTRACT depends on `CORTEX_ENABLED_CROSS_REGION` staying at `AWS_EU` or something broader. Setting it to `DISABLED` would break AI_EXTRACT in this account. Attachment content is processed transiently in another AWS EU region (probably Ireland or Frankfurt); it is not stored there.

## 4. Cost per page / token and pilot estimate

Billing rules ([cost considerations](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql-cost)):

- **AI_EXTRACT:** input tokens plus output tokens, and the `responseFormat` counts as input. Each page of a paged file (PDF, DOCX, TIF) is 970 tokens, and each image file counts as one page (970 tokens).
- **AI_PARSE_DOCUMENT:** billed per page. One image file is one page. HTML/TXT is billed per 3,000-character chunk.
- **AI_COMPLETE:** input plus output tokens, at a rate that depends on the model. A larger JSON schema means more tokens.
- Requesting AI_EXTRACT scores costs nothing extra. Warehouse time is billed separately.
- Snowflake publishes the rates in the [Snowflake Service Consumption Table](https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf).

**Rates observed in this account** (`SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY`, last 90 days, read-only):

| Function | Model | Volume | Credits | Rate |
|---|---|---|---|---|
| AI_EXTRACT | arctic-extract | 801,109 tokens (13 calls) | 4.446 | **5.55 credits / million tokens** (about 0.0054 credits per page) |
| AI_PARSE_DOCUMENT | layout | 433 pages (10 calls) | 1.585 | **3.66 credits / 1,000 pages** |

The two smoke-test calls on 2026-10-26 (199 and 233 tokens) were both billed at exactly 5.55 credits per million tokens.

**Pilot estimate.** Assume 500 Attachments a month, about 3 pages each, and a `responseFormat` plus answers of about 400 tokens per Attachment. All three numbers are guesses to replace once sample Attachments exist.

| Path | Tokens / pages per month | Credits per month |
|---|---|---|
| AI_EXTRACT on the file | 1,500 pages × 970 = 1.46M + 0.2M = about 1.66M tokens | **about 9.2** |
| Add AI_PARSE_DOCUMENT LAYOUT (e.g. archiving text) | 1,500 pages | about 5.5 |
| `scale_factor` 2.0 on scanned pages | doubles the page tokens | about 17 |

The cost grows linearly with pages, so the estimate scales directly: 1,000 Attachments at 3 pages is about 18 credits a month. To convert credits to currency, use the account's contract price per credit (not checked here). For tighter estimates, use `AI_COUNT_TOKENS` on representative text and check `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` after a trial batch.

## 5. Holding output to a fixed schema per Document Type

**AI_EXTRACT `responseFormat`** ([AI_EXTRACT arguments](https://docs.snowflake.com/en/sql-reference/functions/ai_extract#arguments)):

- Accepted forms: a simple `{'label': 'question'}` object, an array of questions, an array of `[label, question]` pairs, or a JSON schema under the key `schema`.
- JSON schema rules: the top level must be `type: object`. Each property is extracted independently and must be one of:
  - `type: string` (entity)
  - `type: array` (list of strings)
  - `type: object` with `column_ordering` and `type: array` columns (table)
- **String is the only scalar type.** `number`, `integer`, `boolean`, `date` and nested objects other than tables are rejected. Testing `'type':'number'` in this account returned `"Error parsing JSON Schema: Incorrect 2nd-level type"`.
- A schema cannot be combined with other formats in one call.
- Limits: 100 entity questions per call, with a table question counting as 10.
- Typing (dates, decimals), required-field checks and code lookups must therefore happen *after* Extraction, in SQL. One `responseFormat` (stored as an OBJECT) per Document Type fits this well, and Snowflake's guidance is to "use one schema for a specific type of document" ([document extraction guide](https://docs.snowflake.com/en/user-guide/snowflake-cortex/document-extraction#table-extraction-best-practices)).
- If base accuracy falls short for a Document Type, `arctic-extract` can be fine-tuned and called with `model => 'db.schema.my_tuned_model'`. Scores are still returned.

**AI_COMPLETE `response_format`** ([structured outputs](https://docs.snowflake.com/en/sql-reference/functions/ai_complete-structured-outputs)):

- Full JSON schema (`type: json`, `schema`, `required`, `enum`, nested objects/arrays, `$defs`/`$ref`) or a SQL type literal (`TYPE OBJECT(...)`).
- Every generated token is checked against the schema. If a `required` field cannot be produced, the call raises an error.
- Not supported: `minimum`/`maximum`, `minLength`/`maxLength`, `format`, `minItems`/`maxItems`, `pattern*`, and external `$ref`. Property names may contain only letters, digits, `-` and `_`, up to 64 characters. OpenAI models also need `additionalProperties: false` and a full `required` list.
- Snowflake recommends temperature 0, and TRY_COMPLETE to absorb errors.
- **No confidence value** (see section 2).

**Implication:** AI_EXTRACT with a per-Document-Type JSON schema plus `scores => TRUE` is the only one of the three that gives both a fixed schema and a Confidence Score. AI_COMPLETE gives richer typing but would need its Confidence Score built some other way.

## Could not verify

- **Whether this role can call every function.** `EXPLAIN_PRIVILEGES` failed with "Privilege analysis requires access on all objects in the statement". AI_EXTRACT calls succeeded as `ENGINEER_ROLE`. AI_PARSE_DOCUMENT and AI_COMPLETE access is inferred from account usage history, not tested with this role.
- **Which region AI_EXTRACT requests are actually processed in.** It is inside `AWS_EU` but not reported per query. The exact member regions of `AWS_EU` were not confirmed from the parameter reference page.
- **Published credit rates** in the Service Consumption Table PDF. Rates above are the ones observed in this account. Currency price per credit was not checked.
- **How well scores are calibrated** on real feed-planning and logistics Attachments, and how much accuracy suffers on poor scans. This needs sample Attachments and a labelled set.
- **The Excel/CSV conversion routes** in "Formats outside the AI functions" were not tested.
- **Whether AI_EXTRACT handles `.xlsx` saved as PDF, or multi-sheet workbooks, without loss.** Not tested.
