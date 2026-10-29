# Option B duty map: can Snowflake do each Mailbox ingestion duty?

Research for issue #33 (part of #31). It extends `docs/research/mailbox-ingestion-options.md` (issue #7) and does not repeat it. The decision is made later by a human.

Question: for Option B (Snowflake external access integration + Snowpark Python procedure + serverless task polling Graph delta query), which Snowflake mechanism covers each Mailbox ingestion duty, with what verdict, and how does that compare with the Azure Function (Option A)?

Date of research: 2026-10-29. Sources are Microsoft Learn, Snowflake documentation, and read-only SQL against `JBSUSA-PILGRIMS_EUROPE_PROD`. Nothing was changed in the account and no call was made to Graph.

Verdict key: **Works** = documented and has a precedent in this account. **Works, with care** = documented, but a limit or gotcha applies. **Unproven** = no document rules it out, but nothing here shows it working.

## Summary

- Every duty has a documented Snowflake mechanism. **No duty is ruled out.** Three need design care (large Attachments, the stage path, and the memory ceiling) and two are unproven in this account (serverless task running a procedure that uses an external access integration, and OAuth via a security integration against Entra).
- **Token:** the two existing Graph precedents do **not** use a security integration. They store tenant ID, client ID and client secret as `GENERIC_STRING` secrets and the procedure calls the Entra token endpoint itself with `requests`. A security integration plus `OAUTH2` secret is the documented alternative, but there are no `API_AUTHENTICATION` integrations in the account today. [S1][S2][A1]
- **Large Attachments** are the main risk. Graph does not limit the size of a read, but Exchange allows messages and file attachments up to 150 MB. Snowflake publishes **no memory number** for procedure handlers, only "returns an error if it consumes too much memory". A DEV proof with a near-150 MB Attachment would settle it. [M3][M4][S6]
- **Stage path gotcha:** `internetMessageId` looks like `<abc@host>`. Stage folder names allow only English letters, numbers and a limited set of special characters, so the folder must be a hash or sanitised form of the id. [S5]
- **`put_stream` gotcha:** it compresses with gzip by default (`auto_compress=True`). Set `auto_compress=False` to keep the original bytes of an Attachment and of the `.eml`. [S4]
- **Timeouts:** a task run times out at the lowest non-zero of `USER_TASK_TIMEOUT_MS` (default 1 hour, maximum 7 days) and `STATEMENT_TIMEOUT_IN_SECONDS` (2 days in this account). Not a constraint at Mailbox volumes. [S3][A3]
- Compared with the Azure Function, Option B loses push latency and certificate or managed-identity auth, and gains having no hosted compute. It also gains the same pattern IT already approved. Azure Function runtime limits were **not** researched here, so that column is partial.

## Account facts checked (read-only)

| Fact | Evidence |
|---|---|
| Two EAIs call Graph with app credentials | `INTUNE_API_ACCESS` and `SHAREPOINT_HC_EAI` (`SHOW EXTERNAL ACCESS INTEGRATIONS`, `DESC EXTERNAL ACCESS INTEGRATION`) |
| Their secrets are all `GENERIC_STRING`, not `OAUTH2` | `SHOW SECRETS IN ACCOUNT`: `INTUNE_TENANT_ID`, `INTUNE_CLIENT_ID`, `INTUNE_CLIENT_SECRET`, `SHAREPOINT_CLIENT_SECRET`, all `GENERIC_STRING` |
| Both EAIs have `ALLOWED_API_AUTHENTICATION_INTEGRATIONS = []` | `DESC EXTERNAL ACCESS INTEGRATION` |
| No `API_AUTHENTICATION` security integration exists | `SHOW SECURITY INTEGRATIONS` returned only `AAD_PROVISIONING`, `AZUREAD_SSO`, `SNOWFLAKE$LOCAL_APPLICATION` |
| Precedent procedures are Python 3.11 with `snowflake-snowpark-python` and `requests` | `SNOWFLAKE.ACCOUNT_USAGE.PROCEDURES`: `INTUNE_DATA.RAW.INGEST_DEVICES`, `PROD_RAW.SHAREPOINT.CATEGORYDB_LASTMODIFIED_SHAREPOINT_RAW_SP` and others |
| `INGEST_DEVICES` gets the token itself: `requests.post(".../oauth2/v2.0/token", data={grant_type: client_credentials, scope: https://graph.microsoft.com/.default})`, secrets read with `_snowflake.get_generic_secret_string` | `INFORMATION_SCHEMA.PROCEDURES.PROCEDURE_DEFINITION` |
| The precedent pagination follows `@odata.nextLink` in a loop with `timeout=` on each call | same definition |
| Precedent tasks run on **user-managed warehouses** (`ANALYTICS_WH`, `RAW_WH`, `PILGRIMS_EU`), once or twice a day, not serverless | `SHOW TASKS IN ACCOUNT`: `DAILY_DEVICE_SYNC`, `CATEGORYDB_SHAREPOINT_TASK` |
| `CATEGORYDB_SHAREPOINT_TASK` keeps a watermark (file `lastModifiedDateTime`) in a table and reloads only on change | task comment |
| Client secret expiry is tracked in a secret comment ("expires 2028-10-08") | `SHOW SECRETS` |
| Account timeouts | `STATEMENT_TIMEOUT_IN_SECONDS` = 172800, `USER_TASK_TIMEOUT_MS` = 3600000 (`SHOW PARAMETERS ... IN ACCOUNT`) |

## Duty table: Option B next to the Azure Function

| # | Duty | Option B mechanism | Verdict | Azure Function (Option A) | Source |
|---|---|---|---|---|---|
| 1 | Get a Graph token (client credentials) | **Precedent route:** three `GENERIC_STRING` secrets on the procedure (`SECRETS = (...)`), token fetched with `requests` from `login.microsoftonline.com/<tenant>/oauth2/v2.0/token`, scope `https://graph.microsoft.com/.default`. **Documented alternative:** `CREATE SECURITY INTEGRATION ... TYPE = API_AUTHENTICATION AUTH_TYPE = OAUTH2 OAUTH_GRANT = 'CLIENT_CREDENTIALS'`, then `CREATE SECRET ... TYPE = OAUTH2 API_AUTHENTICATION = <integration>`, then `_snowflake.get_oauth_access_token('<secret>')` in the handler. | **Works** (precedent route). Security-integration route: **Unproven** here (no such integration exists, and Snowflake's own SharePoint example is the only Graph evidence). | Entra app secret or certificate; Key Vault or managed identity possible | A1, S1, S2, S7 |
| 2 | Authenticate with a certificate instead of a secret | Security integration offers only `CLIENT_SECRET_BASIC` and `CLIENT_SECRET_POST`. A certificate would mean a `GENERIC_STRING` secret holding the key and building the client-assertion JWT in Python. | **Gap.** Secret only, unless custom JWT code is written (untested). Secrets expire; the account already tracks expiry in a comment. | Certificate or managed identity supported by Azure | S1 |
| 3 | Reach `graph.microsoft.com` and `login.microsoftonline.com` | Network rule `MODE = EGRESS`, `TYPE = HOST_PORT`, default port 443, in an EAI. Existing rule `INTUNE_DATA.RAW.MICROSOFT_GRAPH_NETWORK_RULE` covers both hosts. Private connectivity is optional and needs Business Critical plus extra billing. | **Works** | Outbound from Function App, no Snowflake rule | S2, prior doc |
| 4 | Graph delta query per Mailbox | `GET /users/{mailbox}/mailFolders/{id}/messages/delta`, one call chain per Mailbox and per folder. Follow `@odata.nextLink` until `@odata.deltaLink`. Use `changeType=created`, `$select`, and `Prefer: odata.maxpagesize`. Query options go on the **first** call only, because the tokens encode them. | **Works** | Same call from code, or change notifications instead | M1, M2 |
| 5 | Keep `deltaLink` state per Mailbox | A state table keyed by Mailbox and folder holding the last `@odata.deltaLink`. Write it **after** the batch is landed. The precedent already keeps a watermark in a table. If a run fails, the old link is replayed. | **Works, with care**: Graph says replays happen, so landing must be idempotent (duty 10). Tokens are opaque and should be treated as secret. | Needs its own store (blob, table or Cosmos); nothing built in | M2, A2 |
| 6 | Delta token expiry and reset | Outlook token life "isn't fixed; depends on the size of the internal delta token cache". On `410 Gone` or `syncStateNotFound` the procedure must restart with a full sync. The first run is also a full sync of the folder. | **Works, with care.** Needs a "full resync" path and a polling interval short enough that tokens do not age out. | Same | M2 |
| 7 | Message MIME via `$value` | `GET /users/{id}/messages/{id}/$value` returns the `.eml`. Stream the response to the stage (duty 12). | **Works** (documented in prior doc; page not re-read in this ticket). Size follows the message-size limits in duty 9. | Same | prior doc [11], M4 |
| 8 | File, item and reference Attachments | List `.../attachments?$select=id,name,contentType,size,isInline` (no `contentBytes`). `fileAttachment`: `/attachments/{id}/$value` returns raw bytes. `itemAttachment` (contact, event, message): `$value` returns vCard, iCal or MIME. `$expand` can read nested items up to 30 levels. `referenceAttachment`: `$value` returns **HTTP 405**; keep metadata only. | **Works for file and item.** **Gap for reference**: the linked OneDrive or SharePoint file is not fetched. Reading it would need further Graph permissions, and that is true for both options. | Same | M3 |
| 9 | Large Attachments and size limits | Reading is "not technically limited by attachment size", but base64 `contentBytes` hurts performance. Microsoft advises `$value` or a `$select` without `contentBytes`. Exchange Online allows file attachments up to 150 MB in Outlook clients, and admins can set the message limit between 1 MB and 150 MB. Stream with `requests.get(..., stream=True)` and pass `response.raw` to `put_stream` so the file is not held in memory. | **Works, with care** (streaming) but **Unproven** at 100 MB or more, because Snowflake gives no memory figure (duty 14). | Same Graph limits; Function memory and timeout limits **not researched here** | M3, M4, M5 |
| 10 | Landing metadata and de-duplication | `MERGE` into the Platform Schema table on (Mailbox, `internetMessageId`). Use a staging table per run. Remove duplicate keys from the source batch first. Delta can return the same entity more than once. | **Works.** `internetMessageId` and the `id` caveat come from the prior doc. MERGE semantics were **not** re-read here. | Same MERGE, or Snowpipe plus MERGE | M2, prior doc [10] |
| 11 | Mailbox scoping | Entra app `Mail.Read` application permission with RBAC for Applications limited to the Data Product Mailboxes. | **Same as every option.** IT work, outside Snowflake. | Same | prior doc [2] |
| 12 | Write to an internal stage | `session.file.put_stream(input_stream, "@stage/<path>/<file>", auto_compress=False, overwrite=False)`. Files under 64 MB go as one file; larger files are chunked and reassembled. `Session.sql("PUT ...")` is **not** allowed in stored procedures, but the `session.file` API is the supported route. | **Works, with care**: (a) set `auto_compress=False` or the original bytes are gzipped; (b) an unmodified file with the same name is ignored as a duplicate, so the name must be unique per Attachment; (c) stage folder names allow only English letters, numbers and a limited set of special characters, so hash or sanitise `internetMessageId` for the folder and keep the (Unicode-safe) file name as the leaf. | Blob external stage via storage integration (exists in account), or `PUT` with a service user and key pair | S4, S5, S6 |
| 13 | Scheduling and overlap | `CREATE TASK ... SCHEDULE = '<n> MINUTES'` calling the procedure. One instance of a scheduled task runs at a time; a run is skipped if the last is still going. Serverless tasks scale up to XXLARGE equivalent, set `TARGET_COMPLETION_INTERVAL`, and bill per use. | **Works, with care**: precedent tasks all use user-managed warehouses. A **serverless** task running an EAI procedure is not shown in this account and I found no document either way. Prove it in DEV. | Timer trigger in the Function App | S3, A4 |
| 14 | Procedure timeout and memory | Task run: lowest non-zero of `USER_TASK_TIMEOUT_MS` (default 1 h, 0 to 7 days) and `STATEMENT_TIMEOUT_IN_SECONDS` (2 days here). Handler: "returns an error if it consumes too much memory. The specific limit is subject to change." Python `multiprocessing` is unsupported; `joblib` is. EAI connections per handler are limited, so reuse one `requests.Session`. Fire-and-forget child queries are cancelled when the procedure ends. | **Works for time.** **Gap for memory**: no published number. Mitigate with streaming and one message at a time. | Function plan limits **not researched here** | S3, S6, S8, S9, A3 |
| 15 | Graph throttling | Outlook limits are per app and Mailbox: 10,000 requests per 10 minutes, **4 concurrent requests**, 150 MB upload per 5 minutes (writes only). Delta query is described as reducing throttling risk. | **Works.** A few Mailboxes polled every few minutes is far below these. Keep the procedure to 4 or fewer concurrent calls per Mailbox. | Same limits | M6, M2 |
| 16 | Observability | `EXTERNAL_ACCESS_HISTORY` view lists outbound requests. Task history and `LOG_LEVEL` events for runs. | **Works** | Application Insights (not researched) | S2, S3 |

## Gaps to call out

1. **Memory ceiling is unpublished.** The only statement is that handlers error when they use too much memory. A 150 MB Attachment is the case to test. [S6]
2. **Serverless task plus EAI procedure is unproven in this account.** All precedent tasks use warehouses. If serverless fails, the fallback is a small warehouse with a 60-second minimum per resume. [S3]
3. **Security-integration OAuth against Entra is unproven here.** The precedent route (secrets plus handler-side token) works today and needs no new integration type. Choosing the security integration would add a new object type to IT's approval path. [S1]
4. **No certificate or managed identity.** A client secret is the only first-class option. It expires and must be rotated by hand, as the existing secret comment shows. [S1]
5. **Reference Attachments are metadata only.** Fetching the linked file needs more Graph permissions and is the same in both options. [M3]
6. **Latency is polling latency.** Change notifications cannot reach a task (from prior doc). The interval is bounded by schedule and run time, not by Graph.
7. **Delta moves and ids.** I did not verify how a message moved between folders appears in a per-folder delta, or whether `Prefer: IdType="ImmutableId"` is accepted on delta calls. Keying on `internetMessageId` avoids the question for de-duplication but not for completeness.
8. **Azure Function column is partial.** Function runtime, memory, timeout and cost limits were not researched in this ticket.

## What a DEV proof would need to show

Only if #31 decides the choice hinges on it:

1. A serverless task calls a Python procedure with `EXTERNAL_ACCESS_INTEGRATIONS` and `SECRETS`, fetches a Graph token, and runs a delta query against a test Mailbox.
2. The procedure streams one Attachment of about 100 to 150 MB to an internal stage with `put_stream(auto_compress=False)` without a memory error.
3. A forced failure and rerun shows the same message is not landed twice.

This needs IT's Entra app (#6) or a borrowed test Mailbox.

## Not verified

- Snowflake memory limit for Python procedure handlers (not published).
- Behaviour of `get_oauth_access_token` with an Entra client-credentials secret in this account.
- Whether the MIME `$value` page adds limits beyond the prior doc (not re-read).
- MERGE behaviour with duplicate source keys (not re-read).
- Azure Function limits and prices.
- Exchange Online default message size in this tenant (`Get-TransportConfig`).

## Sources

Account (`JBSUSA-PILGRIMS_EUROPE_PROD`, read-only SQL):

- A1. `SHOW EXTERNAL ACCESS INTEGRATIONS`, `DESC EXTERNAL ACCESS INTEGRATION INTUNE_API_ACCESS` and `SHAREPOINT_HC_EAI`, `SHOW SECRETS IN ACCOUNT`, `SHOW SECURITY INTEGRATIONS`.
- A2. `INFORMATION_SCHEMA.PROCEDURES` definition of `INTUNE_DATA.RAW.INGEST_DEVICES`; `SNOWFLAKE.ACCOUNT_USAGE.PROCEDURES`.
- A3. `SHOW PARAMETERS LIKE '%TIMEOUT%' IN ACCOUNT`.
- A4. `SHOW TASKS IN ACCOUNT` (`DAILY_DEVICE_SYNC`, `INGEST_CROWDSTRIKE_SYNC`, `CATEGORYDB_SHAREPOINT_TASK`).

Snowflake documentation:

- S1. CREATE SECURITY INTEGRATION (External API Authentication). https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-api-auth
- S2. Creating and using an external access integration. https://docs.snowflake.com/en/developer-guide/external-network-access/creating-using-external-network-access
- S3. Introduction to tasks, and CREATE TASK. https://docs.snowflake.com/en/user-guide/tasks-intro and https://docs.snowflake.com/en/sql-reference/sql/create-task
- S4. `FileOperation.put_stream`. https://docs.snowflake.com/en/developer-guide/snowpark/reference/python/latest/snowpark/api/snowflake.snowpark.FileOperation.put_stream
- S5. PUT (stage path and duplicate-file rules). https://docs.snowflake.com/en/sql-reference/sql/put
- S6. Designing handlers that stay within Snowflake-imposed constraints. https://docs.snowflake.com/en/developer-guide/udf-stored-procedure-constraints
- S7. CREATE SECRET. https://docs.snowflake.com/en/sql-reference/sql/create-secret
- S8. Python stored procedure limitations. https://docs.snowflake.com/en/developer-guide/stored-procedure/python/procedure-python-limitations
- S9. Writing the Python handler for a stored procedure. https://docs.snowflake.com/en/developer-guide/stored-procedure/python/procedure-python-writing

Microsoft:

- M1. Get incremental changes to messages in a folder. https://learn.microsoft.com/en-us/graph/delta-query-messages
- M2. Use delta query to track changes in Microsoft Graph data (state tokens, replays, 410 Gone, token duration). https://learn.microsoft.com/en-us/graph/delta-query-overview
- M3. Get attachment (file, item, reference; `$value`; 405 for reference). https://learn.microsoft.com/en-us/graph/api/attachment-get?view=graph-rest-1.0
- M4. Attach large files to Outlook messages or events (3 MB and 150 MB; reading not size-limited; `$value` and `$select` advice). https://learn.microsoft.com/en-us/graph/outlook-large-attachments
- M5. Exchange Online limits (message and attachment limits). https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-limits
- M6. Microsoft Graph throttling limits, Outlook service limits (source text from the `microsoftgraph/microsoft-graph-docs-contrib` repository, `includes/throttling-outlook.md`, because the rendered page was truncated). https://learn.microsoft.com/en-us/graph/throttling-limits

Prior art: `docs/research/mailbox-ingestion-options.md` (sources [10], [11], [2]).
