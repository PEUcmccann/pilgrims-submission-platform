# Ingestion latency options: can a Snowflake-native design beat polling?

Research for issue #37 (part of map #31). The decision is made later, in #40 and then #10. This note does not decide.

Question: can any Snowflake-native route receive Microsoft Graph change notifications instead of polling, or is delta-query polling the only fully Snowflake-contained option? What is the practical minimum polling interval, what does it cost, and what latency do the pilot Data Products need?

Date of research: 2026-10-29. Sources are Microsoft Learn and Snowflake documentation only (listed at the end). Nothing was run against the account.

## Summary

- **No Snowflake-native route can receive Graph change notifications.** Every receiving path I checked either needs a component outside Snowflake that is reachable by Graph (a public endpoint, an Event Hub, an Event Grid topic) or needs a client to push into Snowflake. Delta-query polling from a serverless task is the only design where every running part is inside Snowflake. [1][2][3][4][5][6][7]
- **SPCS public endpoints cannot be a Graph webhook.** "Public access to an endpoint requires Snowflake authentication." Graph sends no Snowflake credential, and the webhook must be reachable by Graph without one. [1][8]
- **Event Grid or Event Hubs deliver Graph events, but Snowflake has no generic consumer for them.** The inbound Azure Storage Queue integration is documented only for Snowpipe file-arrival events, and Snowpipe ignores other event schemas. [9][10][11]
- **Practical minimum polling interval: about 10 seconds.** The Snowflake docs show `SCHEDULE = '10 SECONDS'` and `'30 SECONDS'` examples. I found no stated lower bound for `SCHEDULE`, but the docs show nothing below 10 seconds. [12][13]
- **Cost grows linearly as the interval shrinks.** Under my assumptions (below) 5 minutes is about 11 credits a month, 1 minute about 54, 30 seconds about 108, 10 seconds about 324. [14]
- **Graph's own notification latency is already about a minute.** Outlook `message` notifications are "less than 1 minute" typical, 3 minutes maximum. A 30 to 60 second poll is the same order of latency as the push design. [3]
- **Graph throttling for Outlook is not the limit.** 10,000 requests per 10 minutes and 4 concurrent requests, per app and mailbox. A 10 second poll uses about 0.6% of the request budget when idle. [15]
- **The pilot documents state no latency requirement.** I assume minutes are acceptable because a human works the Review Queue after extraction. This is an assumption to confirm with the Data Product Owners.

## Can a Snowflake-native route receive Graph change notifications?

Graph delivers change notifications to one of three targets: a webhook, Azure Event Hubs, or Azure Event Grid. [3][4]

| Route | Can it receive Graph notifications? | Why |
|---|---|---|
| SPCS service, public endpoint, **with** Snowflake auth | **No** | A public endpoint "requires Snowflake authentication" and the caller needs a service role. Callers present a PAT, a key-pair-derived OAuth token or a session token in the `Authorization` header. Graph does not send those. The webhook must also answer a validation request with a plain-text token and answer each notification within 3 seconds, so Graph has to call it directly. [1][2][8] |
| SPCS service, public endpoint, **without** Snowflake auth | **No such option** | The specification reference says public access always requires Snowflake authentication. I found no setting to turn it off. [8] |
| Graph to Event Hubs, then into Snowflake | **Not natively** | Event Hubs removes the public-URL need on the Graph side, but it is an Azure resource that has to be provisioned (plus a Key Vault or a role grant for the Microsoft Graph Change Tracking service principal). Event Hubs exposes a Kafka endpoint, so something must read it and write to Snowflake: the Openflow Kafka connector (always-on cost, see `mailbox-ingestion-options.md` Option C) or a Kafka Connect or custom reader outside Snowflake. [4][16][17] |
| Graph to Event Grid, then into Snowflake | **Not natively** | Graph creates a partner topic in an Azure subscription. Event Grid can route to a Storage Queue, an Event Hub or a webhook. Snowflake's inbound Azure Storage Queue integration is documented for Snowpipe file-arrival (blob-created) events in the Event Grid event schema. Graph events use CloudEvents, and "if an event subscription uses the CloudEvents schema, Snowpipe ignores its messages without reporting an error". I found no Snowflake feature that reads arbitrary queue messages into a table. [5][9][10] |
| Azure Storage Queue notification integration | **No** | Same as above. It is a Snowpipe and directory-table file-event mechanism, one queue per integration. [9][10] |
| Snowpipe Streaming (SDK or REST) | **No, it is push-only** | It ingests rows that a client sends. The client must run somewhere that Graph can reach, so it only moves the problem. Latency "as low as 5 seconds" refers to ingest-to-queryable, not to receiving Graph events. [18] |
| Snowflake notification integrations and webhooks | **No, outbound only** | The webhook integration sends notifications from Snowflake to an HTTP endpoint. It does not receive. [19] |
| Triggered tasks and streams | **Not a receiver** | A triggered task reacts to new rows in a stream. Something still has to put the row in the table. Triggered tasks run at most every 30 seconds by default, or every 10 seconds with `USER_TASK_MINIMUM_TRIGGER_INTERVAL_IN_SECONDS`. [13][20] |

**Conclusion.** Graph push needs an Azure component that is reachable by Graph (the A1 Function, or an Event Hub or Event Grid partner topic plus a reader). That is Option A in the earlier research, not a Snowflake-contained design. The only fully Snowflake-contained design is Option B: a serverless task calling Graph delta query. Graph's own guidance agrees that polling is the alternative when a public endpoint is not available, and that delta query reduces polling cost and throttling risk. [4][21]

A hybrid exists in Microsoft's documentation: use change notifications as a trigger and delta query to fetch the changes. [21] It still needs the push-receiving component, so it does not change the conclusion.

## Polling interval for serverless tasks

- `SCHEDULE` accepts `'<num> { HOURS | MINUTES | SECONDS }'`. The tasks documentation shows a task "that runs every 10 seconds" and another every 30 seconds. [12][13] I did not find a documented minimum for `SCHEDULE`. The documented minimum for triggered tasks is 10 seconds. [20]
- Snowflake runs one instance of a scheduled task at a time and skips the next scheduled time if the previous run is still going. [12] So the real interval is the larger of the schedule and the run time.
- A task run includes queueing time as well as run time (the docs show an example of 5 s queue plus 7 s run for a 15 s schedule). [12]
- Serverless billing: "Snowflake bills your account based on the actual compute resource usage." The minimum serverless statement size defaults to XSMALL. The 60-second minimum is documented for user-managed warehouse resumes, not for serverless tasks. [12]
- Rate: Table 5 of the Service Consumption Table (effective 2026-10-15) lists Serverless Tasks with a 0.9 multiplier for Snowflake-managed compute and 1 for cloud services. An XSMALL standard warehouse is 1 credit per hour (Table 1(a)). Serverless Tasks Flex is listed at 0.5. [14]

**Practical minimum:** about 10 seconds, with an effective floor of the run time plus queueing. Below that, use a push design.

### Cost of each interval (estimate, not a measurement)

Assumptions, all mine: one task covers both pilot Mailboxes in one run; a run takes 5 or 20 seconds at the XSMALL floor; billed credits = 0.9 x run seconds / 3600 x 1 credit per XS-hour; cloud services and internal-stage storage are excluded; 30-day month; credit price in currency not looked up. The run times are guesses until the procedure exists.

| Interval | Runs per day | Credits per month, 5 s run | Credits per month, 20 s run |
|---|---|---|---|
| 5 minutes | 288 | about 11 | about 43 |
| 1 minute | 1,440 | about 54 | about 216 |
| 30 seconds | 2,880 | about 108 | about 432 |
| 10 seconds | 8,640 | about 324 | about 648 (task is effectively always running) |

Most polls find no new mail, so most of this spend buys nothing. For scale, the Openflow shape in the earlier research has a fixed cost of about 2 `CPU_X64_S` instances running all the time, with the per-instance rate not looked up. The Azure Function shape has Azure costs that were not priced.

## Graph throttling for Outlook (confirmed)

Per Microsoft Learn, the Outlook service applies these limits "to each app ID and mailbox combination": [15]

- 10,000 API requests in a 10 minute period, v1.0 and beta.
- 4 concurrent requests.
- 150 MB upload in a 5 minute period (not relevant to reads).

Impact on polling:

- A 10 second poll is 60 delta calls per 10 minutes per Mailbox, about 0.6% of the request budget when nothing is new. Even adding MIME and per-Attachment `$value` fetches for each new message, volume at pilot scale is unlikely to approach it. Message volume at pilot scale is not known to me.
- The 4 concurrent request limit matters more than the request count: the procedure should fetch Attachments serially or with at most 4 in parallel per Mailbox.
- Other relevant limits: up to 1,000 active change-notification subscriptions per Mailbox across all apps; Outlook subscription lifetime under 7 days (under 1 day with resource data), which a push design must renew. [3]

## Latency options per design

"Graph floor" is the latency of Graph itself. Microsoft publishes a figure for push notifications only (`message`: less than 1 minute typical, 3 minutes maximum). [3] I found no published figure for how soon delta query reflects a new message.

| Design | Mechanism | Expected arrival-to-landed latency | Under a minute? | What it costs to get there |
|---|---|---|---|---|
| **B (Snowflake EAI + procedure + task)**, 5 min poll | Delta query | average about 2.5 min plus run time, worst case about 5 min plus run time | No | Cheapest, about 11 credits per month by the estimate above |
| **B**, 1 min poll | Delta query | average about 30 s plus run time, worst case about 60 s plus run time | Average yes, worst case borderline | About 54 credits per month |
| **B**, 30 s poll | Delta query | average about 15 s plus run time, worst case about 30 s plus run time | Yes | About 108 credits per month |
| **B**, 10 s poll | Delta query | average about 5 s plus run time, worst case about 10 s plus run time | Yes | About 324 credits per month, more if runs exceed 10 s. This is the floor of the task schedule. |
| **A1 (Azure Function, Graph change notifications)** | Push | Graph: typically under 1 min, up to 3 min. Plus Function fetch time and landing in Snowflake | **Not guaranteed**: the Graph maximum is 3 min | An Azure subscription, a public HTTPS endpoint answering within 3 s, subscription renewal every under 7 days, plus the Snowflake landing path (Snowpipe or `PUT`). Azure prices not checked. |
| **A1 via Event Hubs or Event Grid** | Push | Same Graph latency, plus Event Hub or Event Grid hop | Not guaranteed | Same Azure footprint without the public URL, plus a reader into Snowflake (Openflow Kafka connector or an external reader). |
| **A2 (Azure Function on a timer, delta query)** | Poll | Same as B at the same interval | Same as B | Azure hosting, with no cost saving over B. |
| **Hybrid (push as trigger, delta as fetch)** | Push plus poll | Graph notification latency, then one delta call | Not guaranteed | Needs the push receiver as well as the poller. |
| **Openflow (C1 IMAP or C2 Graph)** | Poll (IMAP or Graph) | Flow schedule | Possible | Fixed always-on cost already counted in the earlier research; no latency advantage over B. |

Observations:

1. **Push is not faster than a short poll.** Graph's own `message` figures (typical under 1 minute, up to 3 minutes) are slower than a 10 or 30 second poll. A push design only reduces Snowflake spend and request count, not latency, unless Graph delivers well inside its typical figure. [3]
2. **Getting under a minute is cheap in the sense that no new platform is needed.** A 30 second schedule on the existing Option B design gets average latency about 15 seconds plus run time. The cost is roughly 10 times the 5 minute schedule in credits, about 108 credits per month under the assumptions. Whether that is acceptable is a question for #36 and #40, which hold the cost comparison.
3. **There is no design with guaranteed sub-minute latency.** A1 is bounded by Graph at 3 minutes. B is bounded by schedule plus run time, and I could not confirm Graph delta's own freshness.

## What latency do the pilot Data Products plausibly need?

**Not stated anywhere I could read.** I checked `GLOSSARY.md`, ADR 0001 to 0004, map #4, tickets #10, #11, #12, and the three research notes. None gives a latency or freshness target. Ticket #10 only asks "push or poll".

What the documents do show:

- The two pilot Data Products are Agri Feed Planning and 3PL (#11). Their Submissions are business events sent by email with Attachments, and a Data Product Owner works a Review Queue. [GLOSSARY.md, #11]
- After ingestion come Extraction, Validation, Review and Routing (map #31 scope). Low-confidence Versions wait for a person in the Review Queue. A Late Version always waits for a person (ADR 0004). A human step makes minute-level ingestion latency a small part of the end-to-end time.
- The earlier research note already states the working assumption that "minute-level latency should be acceptable for business Submissions" (`mailbox-ingestion-options.md`, recommendation).
- ADR 0001 treats email as a transport and the Platform Schema as the system of record, so replay is possible. Nothing says a Submission must be visible within a set time.

**Assumption (to confirm with the Data Product Owners for Agri Feed Planning and 3PL):** a delay of up to 5 to 15 minutes between a sender emailing and the Submission appearing in the Review Queue is acceptable. If so, the 5 minute poll on Option B is enough and the cost of going below a minute buys nothing for the pilot. Suggested questions for the owners:

1. When someone sends a planning file or a 3PL document, how soon must it be visible to you for review, in the worst case?
2. Is there a cut-off time of day (for example a planning deadline) after which a late arrival matters?
3. Does anyone act on the data within minutes of receiving it, or only during a working session?

If the answer is "within a minute", the options are B at 30 or 10 seconds, or A1 with the caveat that Graph itself does not guarantee under a minute.

## Not verified

- A documented minimum for `SCHEDULE` below 10 seconds. I saw none, and did not test.
- Real run time and credit use of a Python procedure that calls Graph from a serverless task. The cost table uses assumed 5 and 20 second runs.
- Whether serverless tasks have a per-run minimum charge. I found none documented; the 60-second minimum is documented for user-managed warehouses only.
- Delta query freshness: how soon after delivery a new message appears in `messages/delta`. Microsoft publishes no figure.
- Typical message and Attachment volume per Mailbox at pilot scale.
- Azure prices (Functions, Event Hubs, Event Grid, Storage) and the credit price in currency.
- Whether a Graph subscription can send a Snowflake PAT through query-string parameters on `notificationUrl` (query-string parameters on the URL are included in notification posts [2]) and whether SPCS ingress would accept a token that way. The SPCS documentation specifies the `Authorization` header only, so I treated it as unsupported. A PAT in a URL would also be a poor security design.
- Whether the Openflow Kafka connector can read an Event Hubs Kafka endpoint from an SPCS runtime in this account.

## Sources

1. Snowflake, *Working with services* (Ingress: using a service from outside Snowflake; ingress authentication). https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-services
2. Microsoft Graph, *Receive change notifications through webhooks* (public HTTPS endpoint, validation within 10 s, 3 s response, retries up to 4 h, slow and drop states, query string parameters). https://learn.microsoft.com/en-us/graph/change-notifications-delivery-webhooks
3. Microsoft Graph, *Set up notifications for changes in resource data* (delivery channels; Outlook subscription limits and lifetime; latency table). https://learn.microsoft.com/en-us/graph/change-notifications-overview
4. Microsoft Graph, *Receive change notifications through Azure Event Hubs*. https://learn.microsoft.com/en-us/graph/change-notifications-delivery-event-hubs
5. Microsoft Learn, *Receive Microsoft Graph API change events through Azure Event Grid* (partner topic, CloudEvents). https://learn.microsoft.com/en-us/azure/event-grid/subscribe-to-graph-api-events
6. Snowflake, *Snowpipe Streaming overview* (SDKs and REST API, client-push model, latency). https://docs.snowflake.com/en/user-guide/snowpipe-streaming/data-load-snowpipe-streaming-overview
7. Snowflake, *Introduction to tasks*. https://docs.snowflake.com/en/user-guide/tasks-intro
8. Snowflake, *Snowpark Container Services specification reference*, `endpoints` field (`public`): "Public access to an endpoint requires Snowflake authentication." https://docs.snowflake.com/en/developer-guide/snowpark-container-services/specification-reference
9. Snowflake, *CREATE NOTIFICATION INTEGRATION (inbound from an Azure Event Grid topic)*. https://docs.snowflake.com/en/sql-reference/sql/create-notification-integration-queue-inbound-azure
10. Snowflake, *Automating Snowpipe for Microsoft Azure Blob Storage* (Event Grid schema only; CloudEvents ignored). https://docs.snowflake.com/en/user-guide/data-load-snowpipe-auto-azure
11. Same as 9 and 10: no generic queue-to-table consumer is documented there.
12. Snowflake, *Introduction to tasks*: serverless tasks, 10 and 30 second examples, one instance at a time, skipped runs, task duration with queueing, task costs.
13. Snowflake, *Introduction to tasks*, "Run a task on a fixed schedule" (`task_runs_every_10_seconds`) and the 30 second serverless example.
14. Snowflake, *Service Consumption Table*, effective 2026-10-15: Table 1(a) (XS = 1 credit per hour) and Table 5 (Serverless Tasks 0.9, Serverless Tasks Flex 0.5). https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf
15. Microsoft Graph, *Microsoft Graph service-specific throttling limits*, "Outlook service limits". https://learn.microsoft.com/en-us/graph/throttling-limits
16. Microsoft Learn, *Azure Event Hubs for Apache Kafka* (Kafka endpoint on an event hub). https://learn.microsoft.com/en-us/azure/event-hubs/azure-event-hubs-apache-kafka-overview
17. Snowflake, *Openflow connectors* (Kafka listed), and the earlier note `docs/research/mailbox-ingestion-options.md` sources 6 and 22. https://docs.snowflake.com/en/user-guide/data-integration/openflow/connectors/about-openflow-connectors
18. Same as 6.
19. Snowflake, *CREATE NOTIFICATION INTEGRATION (webhooks)* (sends HTTP requests from Snowflake). https://docs.snowflake.com/en/sql-reference/sql/create-notification-integration-webhooks
20. Snowflake, *Triggered tasks* and *CREATE TASK* (`USER_TASK_MINIMUM_TRIGGER_INTERVAL_IN_SECONDS`, minimum 10; triggered tasks run at most every 30 seconds by default). https://docs.snowflake.com/en/user-guide/tasks-triggered and https://docs.snowflake.com/en/sql-reference/sql/create-task
21. Microsoft Graph, *Use delta query to track changes in Microsoft Graph data* ("avoid constantly polling"; combining delta query and change notifications). https://learn.microsoft.com/en-us/graph/delta-query-overview
