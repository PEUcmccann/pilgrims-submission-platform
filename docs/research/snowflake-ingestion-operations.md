# Operating Mailbox ingestion: Snowflake task polling Graph delta versus an Azure Function

Research for issue #34 (part of map #31). The decision is made later by a human; this note only compares how the two designs fail and recover.

Question: compare reliability and operability of polling Microsoft Graph **delta query** from a **Snowflake serverless task** (Python procedure with an external access integration, "Option B") versus an **Azure Function** (timer-triggered, "Option A2" in `docs/research/mailbox-ingestion-options.md`). Scope is everything between the **Mailbox** in Exchange Online and the **Attachment** landing in the **Platform Schema**.

Date of research: 2026-10-29. Sources are Microsoft Learn and Snowflake documentation only (numbered list at the end). Nothing was built or run in the account; every behaviour below is from documentation unless it is marked **Inference** (my reasoning, not stated by a source) or **Not verified**.

## Prior art and what was not found

- `docs/research/mailbox-ingestion-options.md` (branch `research/mailbox-ingestion-options`, merged to `main`) already compares A, B and C on cost, latency and IT burden. This note does not redo that. [28]
- The map (#31) names the branch `research/openflow-operations` (`docs/research/openflow-operations.md`) as prior art. **That branch does not exist on `origin`** (checked with `git ls-remote --heads origin` on 2026-10-29: only `chore/agent-skills-setup`, `docs/domain-model-and-readme`, `main`, `research/mailbox-ingestion-options`, `research/sis-review-queue`, `research/snowflake-ai-extraction`). Nothing from it is used here. Openflow is out of scope for this ticket either way.

## Summary

- **Neither design is clearly more reliable.** The Graph-protocol failures (delta token loss, replays, delta lag, 429 throttling, per-mailbox concurrency) are the same in both and are handled in code, not by the host. The host differences are about overlap, retry, auto-suspend, alerting and deployment.
- **Snowflake task strengths:** the scheduler never runs two instances of a standalone task at once and skips the slot instead [1]; the run's timeout is a task parameter (default 1 hour) [2]; task history, error notifications, and an error-rate alert template are built in [3][4][5][6]; the cursor can sit in a Snowflake table next to the landing tables.
- **Snowflake task hazards unique to it:** `SUSPEND_TASK_AFTER_NUM_FAILURES` can turn a Graph outage into a stopped pipeline [2]; a task that is modified is only picked up when resumed, and dropping the owner role pauses it [1]; the built-in error-rate alert **never fires** unless `LOG_EVENT_LEVEL` is raised from its default `OFF` [4]; auto-retry is documented for task graphs, with the standalone-task case not spelled out [2]; the Snowpark local testing framework cannot run raw SQL or external calls [11].
- **Azure Function strengths:** mature GitHub Actions deployment with OIDC [25], deployment slots on some plans [26], a manual HTTP invoke for timer functions [21], `IsPastDue` for missed invocations [21], per-function retry policies [22].
- **Azure Function hazards unique to it:** the timer does not retry by default [21]; it depends on blob host storage for its single-instance lock [21]; `runOnStartup` and scale-out can cause surprise invocations [21]; Application Insights sampling can hide executions [24]; the cursor lives in a separate store from the Snowflake landing, so a crash can split the two (Inference).
- **Hardest shared problem:** silent loss. Delta tokens for Outlook have no fixed lifetime, can be evicted, and delta responses can lag [15]. Both designs need the same guard: a periodic reconciliation of the Mailbox against the landing table, keyed on `internetMessageId`.
- The failure-mode table below lists 20 modes and the guard each design needs.

## Facts that apply to both designs

### Graph delta query for messages

- Delta is per folder. The first call returns all current messages and a `@odata.nextLink` chain; the last page carries a `@odata.deltaLink` to save for the next round. State tokens are opaque and encode the query options given on the first call. [15][16]
- Supported options for messages: `$select`, `$top`, `$expand`, a limited `$filter` (`receivedDateTime ge|gt`, **max 5,000 messages when a filter is used**), a limited `$orderby`, and a custom `changeType=created|updated|deleted`. [16]
- **Replays:** "Your application must be prepared for replays" - the same change can appear in later responses. A single entity can appear several times in one round. [15]
- **Processing delays:** changes may not show up immediately because of replication delay; retry the link later. [15]
- **Token duration:** for Outlook entities (message, mailFolder, event, contact...) "the upper limit isn't fixed; it's dependent on the size of the internal delta token cache" and old tokens are deleted when the cache is full. An expired token should give a 40X error such as `syncStateNotFound`. [15]
- **Synchronization reset:** delta can return `410 Gone` with a `Location` header that holds an empty `$deltatoken`, meaning restart with a full sync. [15]

### Graph throttling

- On throttling Graph returns `429 Too Many Requests` with a `Retry-After` header; wait that long, then retry; avoid immediate retries because every request counts against usage while throttled. Without `Retry-After`, use exponential backoff. [17]
- Graph's own guidance prefers change tracking and change notifications to "continuously polling", because polling is more likely to be throttled. Delta "likely limit[s] the chances of the requests being throttled". [15][17]
- Outlook limits are per **app ID and mailbox**: 10,000 API requests per 10 minutes, **4 concurrent requests**, 150 MB upload per 5 minutes; exceeding the limit on one mailbox does not affect other mailboxes. [18]
- Status codes worth handling: 429, 503 and 504 (retry after a delay; 503 may carry `Retry-After`), 509 (bandwidth cap), 410 (gone), 401/403 (auth, including `insufficient_claims` from Conditional Access). [19]

### What the host cannot fix

Entra app credentials, RBAC for Applications scoping, `internetMessageId` as the dedupe key, MIME and attachment fetching, and the "Graph `id` changes when a message moves unless `ImmutableId` is requested" rule are identical in both designs. [28]

## Snowflake task with Python procedure: how it behaves

### Scheduling, overlap and time limits

- "Snowflake ensures only one instance of a task with a schedule is run at a time. If a task is still running when the next scheduled run time occurs, then that scheduled time is skipped." [1]
- A run that overruns the interval continues until it completes, times out or fails. `USER_TASK_TIMEOUT_MS` defaults to 3,600,000 (1 hour), range up to 7 days; the effective timeout is the lowest non-zero of it and `STATEMENT_TIMEOUT_IN_SECONDS`. [1][2]
- Serverless tasks scale compute to the observed runtime, and a `TARGET_COMPLETION_INTERVAL` can be set. Billing is actual compute used. [1]
- The docs define the task-history state `SKIPPED` only as "the optional WHEN parameter ... returned a FALSE value". They do not say whether a slot skipped because the previous run was still going is recorded as a row. **Not verified**; do not rely on task history to reveal overlap skips. [5]

### Retries and auto-suspend

- `TASK_AUTO_RETRY_ATTEMPTS`: default 0 (off), values 0-30, retries "from the last task in the graph that failed". The reference says to set it "on the root task of a task graph". Whether a standalone task counts as its own root is not stated. **Not verified.** Each retry appears in history with `SCHEDULED_FROM = AUTOMATIC RETRY` and an incremented `ATTEMPT_NUMBER`. [2][5]
- `SUSPEND_TASK_AFTER_NUM_FAILURES`: when above 0, a standalone task is suspended after that many consecutive runs that fail with a **user error or time out**. Runs that are skipped, cancelled, or fail with a **system error** are "indeterminate" and are not counted. It can be set on the task, schema, database or account. [1][2] The default value was not retrieved; set it explicitly.
- The docs recommend auto-suspend and auto-retry to avoid wasting credits on failing tasks. [1] For ingestion the cost of that saving is a stopped pipeline during any long Graph or network outage, so the setting needs a matching alert and a resume procedure (see table).
- Other ways a task stops without failing: dropping the owner role makes the dropping role the owner and the task is paused until the new owner resumes it; revoking `EXECUTE TASK` prevents all runs of tasks that role owns. [1]

### Monitoring and alerting

- `TASK_HISTORY` (table function and Account Usage view) gives state (`SUCCEEDED`, `FAILED`, `CANCELLED`, `SKIPPED`), error code and message, scheduled/start/complete times, `SCHEDULED_FROM`, `ATTEMPT_NUMBER`. Timed-out runs always show `FAILED`. The Account Usage view lists only completed runs. [5][6]
- `ERROR_INTEGRATION` on a task pushes one message per failed run (including each failed retry) to Amazon SNS, Google Pub/Sub or Azure Event Grid via a notification integration. Delivery is at-least-once, so duplicates are possible. [3][4]
- The built-in `TASKS_ERROR_RATE` alert template emails or calls a webhook when the cumulative error rate crosses a threshold. It reads task events from the event table, so `LOG_EVENT_LEVEL` must be `INFO` (or lower) on the tasks. At `OFF` (the default) "the alert never fires"; at `WARN` or `ERROR` the rate is overstated. [3][4]
- Custom alerts: "alert on a schedule" evaluates a condition on existing data; "alert on new data" watches new rows in a table or view (needs change tracking); both need a warehouse or serverless compute. [7] This is how a "no successful run in N minutes" freshness check is built, because nothing fires when nothing runs.
- Event tables collect log, trace and metric telemetry for handler code; telemetry collection has its own cost. [8]
- External calls are visible in the `EXTERNAL_ACCESS_HISTORY` view. [9]

### Versioning, testing and deployment from GitHub

- A standalone task runs the version that was current when it was last resumed or manually run; after it is suspended and modified, a new version is set only when it is resumed or run. `TASK_VERSIONS` records versions. A currently running task finishes on its old version. [1]
- Manual test run: `EXECUTE TASK`. [1]
- Procedure packaging: `snow snowpark build` and `snow snowpark deploy` upload code to a stage and create the procedure objects from `snowflake.yml`. [10]
- Declarative alternative: **DCM Projects** went generally available on 2026-08-07. Definition files (including tasks, in the GA notice) live in Git; `snow dcm plan` previews, `snow dcm deploy` applies; one DCM project object per environment. A deploy can in rare cases fail midway and leave some objects partly changed; re-running usually restores the target state. `TEST` / `PREVIEW`, and the **GitHub Actions for DCM, are still Preview**. [12]
- Pipeline shape Snowflake documents: validate on pull request (`snow dcm plan`), deploy on merge (`snow dcm deploy`, `snow snowpark deploy`, `snow sql -f`, `snow git execute`), verify (`snow dcm test`). [14]
- GitHub Action `snowflakedb/snowflake-actions` (GA) installs the CLI and supports secretless OIDC / workload identity federation through a service user whose subject matches the repo, branch or environment. If the account restricts inbound IPs, the managed network rule `SNOWFLAKE.NETWORK_SECURITY.GITHUBACTIONS_GLOBAL` can be added to the service user's network policy. [13][29]
- Local unit tests: the Snowpark local testing framework runs in memory. `Session.sql(...)` and raw SQL strings are not supported (mock them), and objects such as stored procedures are not persisted between sessions. So the Graph call and the stage/table writes can only be exercised in a real DEV account; the Graph client logic must be factored so it can be unit-tested without Snowflake (Inference). [11]
- Secrets and egress: the procedure uses an external access integration with a network rule and secret objects; the Snowflake security integration for OAuth client credentials supports client secrets only (no certificate). [9][28]

## Azure Function (timer, polling delta): how it behaves

### Scheduling, overlap and time limits

- Timer trigger schedule is an NCRONTAB expression (UTC unless `WEBSITE_TIME_ZONE` is set); a `TimeSpan` schedule is allowed only on App Service plans, and there "when a function completes after running longer than the specified interval, the timer immediately invokes the function again". [21]
- Scale-out: "only a single instance of a timer-triggered function is run across all instances. It will not trigger again if there is an outstanding invocation still running." The single-instance lock uses blob storage through `AzureWebJobsStorage`; the identity needs Storage Blob Data Owner. If two function apps share host storage without distinct host IDs, only one timer runs. [21]
- `runOnStartup = true` fires on every start, wake from idle and scale-out; Microsoft says not to set it in production. [21]
- `UseMonitor` persists schedule occurrences; `IsPastDue` is true when an invocation is late, for example after a restart. [21]
- `functionTimeout` in `host.json` bounds an execution; on timeout "a timeout error occurs and the language worker process restarts". Defaults in the plans table: 30 minutes on Flex Consumption (maximum unbounded) and Premium. [23]

### Retries

- "Unlike the queue trigger, the timer trigger doesn't retry after a function fails. When a function fails, it isn't called again until the next time on the schedule." [21]
- The error-handling page lists **Timer** among the triggers that support runtime-enforced retry policies (`fixedDelay` or `exponentialBackoff`, `maxRetryCount`, `-1` = retry forever), evaluated when a function raises an uncaught exception. The timer page's own example uses `[FixedDelayRetry(5, "00:00:10")]`. The two pages read as inconsistent; the retry policy is opt-in. [21][22]
- There is no equivalent of auto-suspend: a failing timer keeps firing on schedule until someone disables the function app or function.

### Monitoring and alerting

- Application Insights is built in and collects logs, performance and errors. Sampling is on by default and can drop completed-execution telemetry; there is a daily data cap, after which new logs do not appear. [24]
- Azure Monitor alert rules, action groups and the exact failure-count queries were **not retrieved** for this note. **Not verified.**

### Versioning, testing and deployment from GitHub

- `azure/login` with OIDC (a user-assigned managed identity with a federated credential and the Website Contributor role) plus `Azure/functions-action` deploys from a workflow in `.github/workflows/`. The portal can create this workflow. [25]
- Deployment slots: Consumption 2 (production + staging), Premium 3, Dedicated 1-20, **Flex Consumption none** (rolling updates instead). [26]
- A timer function can be invoked manually over an HTTP admin endpoint, listed for integration testing and slot-swap smoke tests. [21]
- Plan note: the Functions pages state that Linux Consumption apps should migrate to Flex Consumption. The hosting plan therefore decides whether slots exist. [23][26][27]

## Failure modes and the guard each design needs

"Task" = Snowflake serverless task + Python procedure. "Function" = Azure Function timer + Graph client. "Both" = identical logic in code. Sources in brackets.

| # | Failure mode | Task design needs | Function design needs |
|---|---|---|---|
| 1 | **Run overlaps the next slot** | Nothing for concurrency: scheduler runs one instance and skips the slot [1]. Guard against the *effect*: a freshness alert (see 16), because skipped slots may not be visible in history [5]. | Nothing on one app for NCRONTAB (no new run while one is outstanding) [21]. On an App Service plan with a `TimeSpan` schedule, runs restart immediately after an overrun [21]; prefer NCRONTAB. Same freshness alert. |
| 2 | **Hung or slow Graph call** | Per-request HTTP timeout in the handler; set `USER_TASK_TIMEOUT_MS` to a few multiples of the interval (default 1 h is too long for a 1-5 min poll) [2]. Cursor advances only after landing (7). | Per-request HTTP timeout; set `functionTimeout` explicitly (default 30 min on Flex/Premium) [23]. A timeout restarts the worker, so no in-memory state may be relied on [23]. |
| 3 | **Transient failure (5xx, DNS, egress)** | Retry with backoff inside the run for a bounded budget; fail the run if exceeded. Optionally `TASK_AUTO_RETRY_ATTEMPTS` (0-30) after confirming it works for a standalone task in DEV [2]. The next scheduled run is also a retry because the cursor did not advance. | Same in-run backoff. Timer does not retry by default [21]; the next tick is the retry, or add a fixed or exponential retry policy and avoid `-1` (infinite) [22]. |
| 4 | **Repeated failure, task or function stops** | Set `SUSPEND_TASK_AFTER_NUM_FAILURES` deliberately: low value saves credits but stops ingestion in any outage; `0` or a high value keeps retrying [1][2]. Whatever the value, alert on `SUSPENDED` state and on stale cursor (16), and document the resume step. System errors, cancels and skips do not count toward the limit [2]. | No auto-suspend exists; it keeps firing [21]. Guard is cost/noise: backoff in code and alert on error count. |
| 5 | **Task left suspended** (after deploy of a changed definition, owner role dropped, `EXECUTE TASK` revoked) | Deploy pipeline must end with `ALTER TASK ... RESUME` and a post-deploy check on `SHOW TASKS` state; a modified task only takes the new definition when resumed [1]. Task ownership should sit on a stable role, not a person [1]. Whether `snow dcm deploy` resumes tasks was **not verified**. | Function app can be stopped or disabled; post-deploy smoke test via the manual HTTP invoke [21]. |
| 6 | **Delta token expired / evicted / `410 Gone`** | Catch `410` and `syncStateNotFound`/40X on the stored link; start a new delta round, bounded by `receivedDateTime ge {watermark - overlap}` (max 5,000 messages with `$filter`) [15][16]; rely on dedupe (8). Alert whenever a resync happens. | Same code. Cursor store must be able to be reset without redeploy. |
| 7 | **Crash after landing but before cursor save (or the reverse)** | Order: write stage files and metadata, then save the cursor last, so a crash causes replays rather than gaps. Cursor and landing rows are in the same system, so a transaction can cover table writes; stage `PUT` is not transactional (Inference, not in sources reviewed). | Cursor lives in a separate store (blob/table) from the Snowflake landing, so there is no common transaction (Inference). Same order rule; dedupe is mandatory. |
| 8 | **Replays / same message several times** | `MERGE` on Mailbox + `internetMessageId` (not Graph `id`); idempotent stage path keyed by `internetMessageId` [15][28]. | Same. |
| 9 | **Delta lags or omits a recent message (eventual consistency)** | Do not trust a single empty delta as proof of no mail [15]. Run a reconciliation job every N minutes: list messages with `$filter=receivedDateTime ge {now - window}` and compare `internetMessageId` with the landing table; land the difference. If `$orderby` is used, it must appear in `$filter` or Graph returns `InefficientFilter` [20]. | Same job (a second timer function or the same one on a slower cadence). |
| 10 | **Message moved or deleted before it is polled** (a Mailbox rule, a user) | Delta is per folder and `changeType=created` returns only new messages [16]. How a message that was created and moved out between two polls appears is **not documented on the pages reviewed**; treat as a risk to prove in DEV. Mitigation: reconcile across the Mailbox, not just the Inbox (9); request `ImmutableId` if Graph ids are stored [28]. | Same. |
| 11 | **Graph throttling (429)** | Honour `Retry-After`; never retry immediately [17]. Bound the wait to the remaining run budget, then end the run cleanly with the cursor unchanged, so a sleep does not hold the single task slot indefinitely (1). Keep the poll interval modest and use delta rather than listing [15][17]. | Same. Sleeping consumes billable execution time on Consumption-style plans (plan-dependent; billing not verified here). |
| 12 | **Per-mailbox concurrency (4) and upload limits** | Cap parallel attachment fetches to 4 per Mailbox (limit is per app ID + Mailbox) [18]. No uploads to Graph are needed. | Same. Scale-out of Functions does not raise the limit because the key is app + mailbox [18]; a single-instance timer avoids it anyway [21]. |
| 13 | **Large message or attachment exceeds run budget / memory** | Process per message, commit per message, and stop at a time budget so the next run resumes; avoid loading a full MIME into memory where the handler allows. Memory limits for Python procedures were not looked up. **Not verified.** | `functionTimeout` and plan memory limits apply [23]; same per-message commit. |
| 14 | **Credential expiry or revoked consent** | Alert on `401`/`403` separately from generic failure; track Entra secret expiry date outside Snowflake. The Snowflake OAuth security integration supports client secrets only, so rotation is a manual secret update [28]. | Same alert; the Function can use certificates or a managed identity for its own Azure resources, but the Graph app secret expiry problem is the same [28]. |
| 15 | **Egress or policy block** (network rule, Conditional Access `insufficient_claims`, Entra policy) | Network rule must list `graph.microsoft.com` and `login.microsoftonline.com` (already exists in the account) [28]. Watch `EXTERNAL_ACCESS_HISTORY` for denied calls [9]. `403 insufficient_claims` is a Conditional Access response [19]. | Function app firewall/VNet rules; `403` as in [19]. |
| 16 | **Silent non-run** (no failure raised, nothing ingested) | Scheduled custom alert: "no successful run, and no new Submission cursor update, in 3x the interval" using `TASK_HISTORY` and a heartbeat table the procedure writes each run [5][7]. Required, because `TASKS_ERROR_RATE` only counts runs that happened [4]. | Same heartbeat alert from App Insights or Snowflake. `IsPastDue` and `UseMonitor` help detect missed timer ticks [21]. |
| 17 | **Alert pipeline is not actually armed** | Set `LOG_EVENT_LEVEL = INFO` on the task; at the default `OFF` the error-rate alert never fires [4]. Create an `ERROR_INTEGRATION` that has a working destination, remembering at-least-once duplicates [3][4]. Test with a deliberately failing run. | Check Application Insights sampling and the daily cap so failures are not dropped [24]. Test with a deliberately failing run. |
| 18 | **Bad deploy / rollback** | Plan before deploy; PR validation with `snow dcm plan`, deploy on merge; keep the previous procedure version in Git and redeploy it [14][12]. A DCM deploy can partially apply; re-run to converge [12]. Task version changes only on resume [1]. Test the Graph path in DEV with `EXECUTE TASK` [1]; local test mode cannot run SQL [11]. | Deploy via GitHub Actions + OIDC [25]; slots on Consumption/Premium for staging and swap, none on Flex [26]. Smoke test via HTTP-invoked timer [21]. |
| 19 | **Unexpected extra invocations** | Not applicable: no startup trigger. A manual `EXECUTE TASK` while the schedule is active is possible; the overlap rule covers scheduled instances [1]. | Do not set `runOnStartup` in production; scale-out and restarts otherwise fire extra runs [21]. Dedupe (8) is the backstop. |
| 20 | **Host dependency fails** | Snowflake account/task service only; no separate store. A Snowflake region or service incident stops ingestion and the freshness alert is the only signal. | Blob host storage and its identity role (Storage Blob Data Owner) are required for the timer lock [21]; storage firewall or identity changes can stop the timer. Add a storage health alert (Azure Monitor alert rules **not verified**). |

## Reading the table (not a decision)

- Rows 6, 7 (ordering), 8, 9, 10, 11, 12, 13, 14, 15 are **code guards that both designs need**. Choosing between them does not remove them.
- Rows where the **task design is stricter by construction:** 1 (no overlap, no TimeSpan trap), 19 (no startup trigger), 20 (no separate lock store), and the co-located cursor in 7.
- Rows where the **task design adds a hazard:** 4 and 5 (auto-suspend and "modified but not resumed"), 17 (`LOG_EVENT_LEVEL` default), and the testing gap in 18.
- Rows where the **Function adds a hazard:** 3 (no default retry), 19 (`runOnStartup`/scale-out), 20 (host storage), 17 (sampling).
- Cost of the shared guard in row 9 is one extra scheduled job either way.

## Not verified

- The default value of `SUSPEND_TASK_AFTER_NUM_FAILURES` on a new task.
- Whether `TASK_AUTO_RETRY_ATTEMPTS` takes effect on a standalone (non-graph) task.
- Whether an overlap-skipped schedule slot appears in `TASK_HISTORY` or `ACCOUNT_USAGE.TASK_HISTORY`.
- Whether `snow dcm deploy` resumes or suspends tasks, and whether `CREATE OR ALTER TASK` leaves a task suspended after change (the dbt tutorial suspends, alters, then resumes explicitly, which suggests the pattern but is not a statement).
- Memory limits and per-run time ceilings of a Snowflake Python stored procedure called from a task.
- How a message created and moved out of the Inbox between two delta polls is reported by delta query.
- Azure Monitor alert-rule configuration for Function failures; Azure plan prices.
- The Snowflake `EXTERNAL_ACCESS_HISTORY` retention and whether it records denied calls.
- Anything about Openflow operations (prior-art branch missing).

## Sources

1. Snowflake, *Introduction to tasks*. https://docs.snowflake.com/en/user-guide/tasks-intro
2. Snowflake, *CREATE TASK*. https://docs.snowflake.com/en/sql-reference/sql/create-task
3. Snowflake, *Configure error alerts and notifications for tasks*. https://docs.snowflake.com/en/user-guide/tasks-errors-integrate
4. Snowflake, *Set up notifications for tasks* (prerequisites for task alerts). https://docs.snowflake.com/en/user-guide/tasks-errors
5. Snowflake, *TASK_HISTORY* (table function). https://docs.snowflake.com/en/sql-reference/functions/task_history
6. Snowflake, *TASK_HISTORY* (Account Usage view). https://docs.snowflake.com/en/sql-reference/account-usage/task_history
7. Snowflake, *Setting up alerts based on data in Snowflake*. https://docs.snowflake.com/en/user-guide/alerts
8. Snowflake, *Event table setup and overview*. https://docs.snowflake.com/en/developer-guide/logging-tracing/event-table-setting-up
9. Snowflake, *Creating and using an external network access integration*. https://docs.snowflake.com/en/developer-guide/external-network-access/creating-using-external-network-access
10. Snowflake, *Snowflake CLI: Snowpark overview*. https://docs.snowflake.com/en/developer-guide/snowflake-cli/snowpark/overview
11. Snowflake, *Local testing framework for Snowpark Python*. https://docs.snowflake.com/en/developer-guide/snowpark/python/testing-locally
12. Snowflake, *DCM Projects* and *Create and deploy DCM Projects*; release note *DCM Projects (General availability)*, 2026-08-07. https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-overview , https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-use , https://docs.snowflake.com/en/release-notes/2026/other/2026-08-07-dcm-projects-ga
13. Snowflake, *Snowflake CLI GitHub Action*. https://docs.snowflake.com/en/developer-guide/snowflake-cli/cicd/github-action
14. Snowflake, *Integrating CI/CD with Snowflake CLI*. https://docs.snowflake.com/en/developer-guide/snowflake-cli/cicd/integrate-ci-cd
15. Microsoft Graph, *Use delta query to track changes in Microsoft Graph data*. https://learn.microsoft.com/en-us/graph/delta-query-overview
16. Microsoft Graph, *Get incremental changes to messages in a folder*. https://learn.microsoft.com/en-us/graph/delta-query-messages
17. Microsoft Graph, *Microsoft Graph throttling guidance*. https://learn.microsoft.com/en-us/graph/throttling
18. Microsoft Graph, *Service-specific throttling limits* (Outlook service limits). https://learn.microsoft.com/en-us/graph/throttling-limits
19. Microsoft Graph, *Error responses and resource types*. https://learn.microsoft.com/en-us/graph/errors
20. Microsoft Graph, *List messages*. https://learn.microsoft.com/en-us/graph/api/user-list-messages?view=graph-rest-1.0
21. Microsoft Learn, *Timer trigger for Azure Functions*. https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer
22. Microsoft Learn, *Azure Functions error handling and retries*. https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-error-pages
23. Microsoft Learn, *Azure Functions scale and hosting*. https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale
24. Microsoft Learn, *Monitor Azure Functions*. https://learn.microsoft.com/en-us/azure/azure-functions/functions-monitoring
25. Microsoft Learn, *Deploy to Azure Functions by using GitHub Actions*. https://learn.microsoft.com/en-us/azure/azure-functions/functions-how-to-github-actions
26. Microsoft Learn, *Azure Functions deployment slots*. https://learn.microsoft.com/en-us/azure/azure-functions/functions-deployment-slots
27. Microsoft Learn, *Flex Consumption plan*. https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan
28. This repo, `docs/research/mailbox-ingestion-options.md` (which cites the Graph, Exchange and Snowflake pages for scoping, security integration and `internetMessageId`).
29. Snowflake, *Tutorial: Set up CI/CD integrations on dbt Projects on Snowflake* (service user, network policy and `snow ... -x` pipeline pattern, and the `CREATE OR ALTER TASK` suspend/resume sequence). https://docs.snowflake.com/en/user-guide/tutorials/dbt-projects-on-snowflake-ci-cd-tutorial
