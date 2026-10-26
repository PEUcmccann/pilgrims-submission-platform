# Mailbox ingestion options: Exchange Online to the Platform Schema

Research for issue #7 (part of #4). The decision is made later by a human in #10.

Question: which ways can read a **Mailbox** in Exchange Online and land the message, email body and **Attachments** in the **Platform Schema** (`SUBMISSION_PLATFORM`) in Snowflake account `JBSUSA-PILGRIMS_EUROPE_PROD` (region `AWS_EU_WEST_2`)?

Date of research: 2026-10-26. Sources are Microsoft Learn and Snowflake documentation, plus read-only SQL against the account.

## Summary

- All three options use the same Microsoft side: an Entra ID app registration with an application permission (`Mail.Read` for Graph, or `IMAP.AccessAsApp` for IMAP), limited to the Data Product Mailboxes. **IT must do this part in every option**, because it needs an Entra admin (consent) and an Exchange admin (mailbox scoping). [1][2][3]
- **Option B (Snowflake external access integration + Python procedure + task, using Graph delta query)** keeps all the running parts inside Snowflake. The account already has two IT-approved integrations that call Graph this way (`INTUNE_API_ACCESS`, `SHAREPOINT_HC_EAI`), and there is an existing network rule for `graph.microsoft.com` and `login.microsoftonline.com`. (Account facts below.)
- **Option A (Azure Function + Graph change notifications)** has the lowest latency, but it needs an Azure subscription and a public HTTPS endpoint. Azure is not on the list of approved tools (Snowflake, Microsoft 365, GitHub), so it needs IT approval and hosting. [4][5]
- **Option C (Openflow)** has **no** Exchange or Outlook connector. It does have generic NiFi processors (`ConsumeIMAP`, `ExtractEmailHeaders`, `ExtractEmailAttachments`, `PutSnowflakeInternalStageFile`, `InvokeHTTP`) that could be built into a custom flow. Openflow Snowflake Deployments are available in all AWS Commercial regions, so eu-west-2 is covered. A deployment costs money all the time (at least one always-on `CPU_X64_S` control-pool node), even when it is idle. [6][7][8][9]

## Facts that apply to every option

### Microsoft side: access to the Mailbox

- Graph `message` exposes `internetMessageId` ("The message ID in the format specified by RFC2822"), `body` (HTML or text), `uniqueBody`, `internetMessageHeaders`, `receivedDateTime`, `from`/`sender`, `hasAttachments` and an `attachments` relationship. [10]
- The Graph `id` of a message **changes when the message is moved to another folder** unless the request sends `Prefer: IdType="ImmutableId"`. So the Graph `id` is not a safe de-duplication key. `internetMessageId` is. [10]
- The full original email is available as MIME (`.eml`) from `GET /users/{id}/messages/{id}/$value`. [11] Storing this file on a stage keeps an exact original for each Version (see ADR 0001: the Platform Schema is the system of record).
- Attachments come in three types: `fileAttachment`, `itemAttachment` (an attached email, event or contact) and `referenceAttachment` (a link to a cloud file). Raw bytes for file and item attachments come from `.../attachments/{id}/$value`. A `referenceAttachment` returns HTTP 405, so a linked OneDrive or SharePoint file is **not** fetched by this call. [12]
- `hasAttachments` does not count inline attachments, so we cannot rely on it alone. [10]
- **Scoping access to only the Data Product Mailboxes:** *RBAC for Applications in Exchange Online* lets an admin give an app `Application Mail.Read` limited to a management scope or an administrative unit. It **replaces Application Access Policies**. Permissions add together: if the app also has tenant-wide `Mail.Read` consented in Entra ID, the scope has no effect, so the Entra grant must be removed. Permission changes can take 30 minutes to 2 hours to apply (cache). This needs the Organization Management role group or Exchange Administrator. [2]
- **IMAP as an app (client credentials):** add `IMAP.AccessAsApp` (Office 365 Exchange Online API), get admin consent, register the service principal in Exchange with `New-ServicePrincipal`, then run `Add-MailboxPermission -AccessRights FullAccess` for each Mailbox. The token scope is `https://outlook.office365.com/.default`. [3]

### Snowflake side: facts checked in the account (read-only)

| Fact | Evidence |
|---|---|
| Region `AWS_EU_WEST_2`, org `JBSUSA`, account `PILGRIMS_EUROPE_PROD` | `SELECT CURRENT_REGION(), CURRENT_ORGANIZATION_NAME(), CURRENT_ACCOUNT_NAME()` |
| An existing EAI calls Graph with app credentials held in Snowflake SECRETs | `INTUNE_API_ACCESS`: `ALLOWED_AUTHENTICATION_SECRETS = INTUNE_DATA.RAW.INTUNE_TENANT_ID, INTUNE_CLIENT_ID, INTUNE_CLIENT_SECRET` |
| A second EAI calls Graph (SharePoint) | `SHAREPOINT_HC_EAI`: secret `DEV_RAW.SHAREPOINT_HC.SHAREPOINT_CLIENT_SECRET`, comment "SharePoint headcount file sync via Graph API" |
| A network rule for Graph already exists | `INTUNE_DATA.RAW.MICROSOFT_GRAPH_NETWORK_RULE` = `graph.microsoft.com:443, login.microsoftonline.com:443` (EGRESS, HOST_PORT) |
| Azure Blob storage integrations and Azure Storage Queue notification integrations exist | `AZ_JTP_DB_EU_PROD`, `SI_JTP_BLOBSTORAGE`, `CLARITYAZUREBLOB` (EXTERNAL_STAGE); `AZURE_NOTIFY_INT`, `NI_JTP_BLOBSTORAGE` (AZURE_STORAGE_QUEUE). So the organisation does run Azure storage, owned elsewhere. |
| No Openflow deployment exists yet | `SHOW OPENFLOW DEPLOYMENTS` returned no rows |

## Option A: Azure Function calling Microsoft Graph

**How it works.** An Azure Function either (A1) receives Graph **change notifications** (webhook) for `users/{mailbox}/mailFolders('inbox')/messages`, or (A2) runs on a timer and calls **delta query**. For each new message it reads the message, its MIME and its Attachments, then writes files to a stage.

- **Change notifications:** Graph needs a **publicly reachable HTTPS endpoint**. It must answer the validation token within 10 seconds and each notification within 3 seconds. If it does not, Graph marks the endpoint "slow" or "drop", and dropped notifications "can't be recovered". Graph retries for up to 4 hours. Event Hubs or Event Grid can be used as delivery targets instead. [4]
- **Subscription lifetime for Outlook messages:** at most **10,080 minutes (under 7 days)**, or 1,440 minutes with resource data. The Function must renew it. [5]
- **Latency for `message` notifications:** average **under 1 minute**, maximum **3 minutes**. [5] Notifications only say *something changed*, so the Function still calls Graph to fetch the content.
- **Delta query (A2):** this is per folder. `changeType=created` returns only new messages. Supported query options are `$select`, `$top`, `$expand` and a limited `$filter` on `receivedDateTime`. Save `@odata.deltaLink` between runs. [13]

**How Attachments reach a stage.** Either (a) the Function writes to an Azure Blob container that Snowflake reads as an **external stage** through a storage integration (the account already uses this pattern, optionally with an Azure Storage Queue notification integration for Snowpipe auto-ingest), or (b) the Function uses the Snowflake Python connector `PUT` to an internal stage. Option (b) needs a Snowflake service user with key-pair auth, and that key is stored in Azure Key Vault.

**De-duplication.** Read `internetMessageId` from the Graph message, or `Message-ID` from the MIME. Use it as the file-path prefix and as the de-duplication key on the landing table. Use `ImmutableId` if the Graph id is stored at all. [10]

**Auth.** Entra app with client secret or certificate. Azure can also use managed identity or Key Vault for the Snowflake key. Snowflake SECRET objects are not used by the Function itself.

**What IT must provide or host.** Entra app registration, consent and RBAC-for-Applications scope (as in every option), **plus** an Azure subscription and resource group, a Function App (public HTTPS endpoint for webhooks), a storage account and container, Key Vault, monitoring, and possibly a firewall or network review for a public endpoint.

**Rough cost.** Azure Functions on a consumption plan plus storage. For a few hundred emails a day this should be small, but **I did not verify Azure prices in this research**. On the Snowflake side there is only Snowpipe or COPY cost.

**Fit with the "existing stack only" rule.** **Poor.** Azure compute is not Snowflake, Microsoft 365 or GitHub, so it needs IT approval and an owner to run it.

## Option B: Snowflake external access integration + Snowpark Python procedure on a task

**How it works.** A scheduled **task** calls a Python **stored procedure**. The procedure uses `EXTERNAL_ACCESS_INTEGRATIONS` and `SECRETS` to get a token and call Graph **delta query** for each Mailbox's inbox. For each new message it fetches `$value` (MIME) and `attachments/{id}/$value`, uploads the bytes to an **internal stage**, and inserts metadata rows into the Platform Schema.

- **External access:** a network rule (`MODE = EGRESS`, `TYPE = HOST_PORT`) plus an EAI listing the allowed secrets. Handler code reads credentials through `_snowflake` (for example `_snowflake.get_oauth_access_token('cred')` for an OAUTH2 secret, or `get_generic_secret_string`). [14] The account's existing Graph network rule already covers the two hosts needed.
- **OAuth client credentials in Snowflake:** `CREATE SECURITY INTEGRATION ... TYPE = API_AUTHENTICATION AUTH_TYPE = OAUTH2 OAUTH_GRANT = 'CLIENT_CREDENTIALS' OAUTH_TOKEN_ENDPOINT = 'https://login.microsoftonline.com/<tenant>/oauth2/v2.0/token' OAUTH_ALLOWED_SCOPES = ('https://graph.microsoft.com/.default')`. Snowflake's documentation shows exactly this example for Microsoft Graph/SharePoint. The only client auth methods are `CLIENT_SECRET_BASIC` and `CLIENT_SECRET_POST`, so **a certificate is not supported by the security integration**. [15]
- **Certificate auth instead:** it could be done by storing the PEM private key in a `GENERIC_STRING` secret and building the Entra client-assertion JWT in Python (for example with `msal` or `cryptography`). I did not test this, and I did not confirm that `msal` is in the Snowflake Anaconda channel.
- **Writing Attachments to a stage from a procedure:** Snowflake documentation says stored procedures can write files, but "for file writes using stored procedures, we recommend using the file staging PUT command". In Snowpark Python that means `session.file.put_stream(...)`. (UDFs can use `SnowflakeFile.open_new_result` plus `COPY FILES` instead.) [16]
- **Change notifications are not practical here.** Graph must reach a public, unauthenticated HTTPS webhook, and a task cannot receive inbound calls. So this option uses **delta-query polling**. (I believe SPCS public endpoints require Snowflake authentication, so they could not act as the webhook either. I did not verify this further.)

**Scheduling and latency.** Tasks accept `SCHEDULE = '<n> SECONDS|MINUTES|HOURS'` or CRON. The docs show schedules as short as 10 and 30 seconds. Snowflake runs only one instance at a time and skips a run if the previous one is still going. [17] A practical choice is 1–5 minutes, so latency is about the polling interval plus the run time.

**De-duplication.** `$select=internetMessageId,...` on the delta call. Use `MERGE` on `internetMessageId` (plus Mailbox) in the landing table. Store the `@odata.deltaLink` per Mailbox in a state table. [13]

**What IT must provide.** Entra app registration with `Mail.Read` (application), admin consent, and an **RBAC for Applications** role assignment scoped to the Data Product Mailboxes (and removal of any tenant-wide Entra grant). On the Snowflake side, an ACCOUNTADMIN (or a role with CREATE INTEGRATION) creates the EAI and security integration. Nothing else is hosted. This is the same pattern as `INTUNE_API_ACCESS` and `SHAREPOINT_HC_EAI`.

**Rough cost.** Serverless task compute, billed per actual use (or an XS warehouse with a 60-second minimum per resume) [17], plus internal stage storage. A run that polls a few Mailboxes every few minutes should be well under an XS-hour per day. **This is an estimate, not a measurement.** No Azure cost.

**Fit with the "existing stack only" rule.** **Good.** It uses only Snowflake and Microsoft 365, and the code lives in GitHub. IT still has to grant the Entra/Exchange permissions, which is true of every option.

## Option C: Snowflake Openflow

**Is there a connector?** **No.** The Openflow connector list includes SharePoint, Microsoft Dataverse, Box, Google Drive, Slack, Kafka, databases and others, but nothing for Exchange, Outlook or email. [6]

**Are there processors?** Yes. These are generic Apache NiFi processors that could be put together into a custom flow:
- `ConsumeIMAP`: reads messages over IMAP and writes the raw email bytes as FlowFile content. It has an `Authorization Mode` and an `OAuth2 Access Token Provider` property, plus `Folder`, `Mark Messages as Read` and `Delete Messages`. [7]
- `ExtractEmailHeaders`: writes `email.headers.message-id`, `from`, `to`, `subject`, `sent_date`, `received_date` and `email.attachment_count` as attributes. [8]
- `ExtractEmailAttachments`: splits attachments into separate FlowFiles with `filename` and `mime.type`. [9]
- `PutSnowflakeInternalStageFile` (writes to an internal stage), `InvokeHTTP` (could call Graph instead of IMAP) and `SnowflakeDetectDuplicate` (checks a hash against a Snowflake table). [18]

**Region.** "Openflow Snowflake Deployments are available to all accounts in AWS, Azure, and GCP Commercial regions". BYOC is available in AWS Commercial regions. The listed exceptions concern GCP Dammam only. So **AWS eu-west-2 is supported**. Snowflake connectors are supported "in every region where Snowflake Openflow is available". [19][20]

**How Attachments reach a stage.** `ConsumeIMAP` → `ExtractEmailHeaders` → `ExtractEmailAttachments` → `PutSnowflakeInternalStageFile`. The raw `.eml` can be staged as well.

**De-duplication.** `email.headers.message-id` from `ExtractEmailHeaders`, checked with `SnowflakeDetectDuplicate` or a `MERGE`. [8][18]

**Auth.** For IMAP: `IMAP.AccessAsApp` plus `New-ServicePrincipal` plus `Add-MailboxPermission` per Mailbox (this is an IMAP grant, **not** RBAC for Applications; the RBAC-for-Applications page lists only MS Graph and EWS as supported protocols). [2][3] The runtime reaches `outlook.office365.com:993` through an EAI and network rule. Openflow supports Snowflake secrets. [20] Openflow connects to Snowflake itself with `SNOWFLAKE_MANAGED` tokens.

**What IT must provide.** Entra/Exchange grants as above. On the Snowflake side, an ACCOUNTADMIN creates the Openflow deployment, runtime, execute-as role and EAI. [21]

**Rough cost.** The deployment's management compute pool "continues to run and incurs costs, even if there are no runtimes running": **1 `CPU_X64_S` instance all the time**. A small runtime adds 1 more `CPU_X64_S` while running, and runtime pools scale to 0 when stopped. So a 24×7 flow uses about 2 `CPU_X64_S` instance-hours per hour (about 1,440 per month), billed at the Table 1(d) rate in the Snowflake Service Consumption Table, plus telemetry ingest. [22] **I did not look up the credit rate per instance-hour.** This is clearly more than Option B for a low-volume Mailbox.

**Fit with the "existing stack only" rule.** It is Snowflake-only, so it is within the rule, **but** it is a new platform service with a fixed running cost, and the flow is a hand-built NiFi graph with no supported connector. It uses IMAP rather than Graph, so Mailbox access is granted with `FullAccess` per Mailbox and is not scoped by RBAC for Applications.

## Comparison

| | A. Azure Function + Graph | B. Snowflake EAI + Python procedure + task | C. Openflow (custom flow) |
|---|---|---|---|
| Mail API | Graph change notifications (or delta) | Graph delta query (polling) | IMAP (`ConsumeIMAP`) or Graph via `InvokeHTTP` |
| Attachments → stage | Azure Blob external stage, or `PUT` to internal stage | `session.file.put_stream` to internal stage | `ExtractEmailAttachments` → `PutSnowflakeInternalStageFile` |
| Original email kept | `$value` MIME → stage | `$value` MIME → stage | raw IMAP bytes → stage |
| De-dup key | `internetMessageId` | `internetMessageId` | `email.headers.message-id` |
| Latency | under 1 min average, 3 min max (notifications) | polling interval (for example 1–5 min) | polling interval |
| Auth to M365 | Entra app: secret or certificate | Entra app: client secret in Snowflake OAUTH2 secret (certificate only with custom JWT code, untested) | `IMAP.AccessAsApp`: OAuth2 token provider |
| Mailbox scoping | RBAC for Applications | RBAC for Applications | `Add-MailboxPermission FullAccess` per Mailbox |
| IT provides / hosts | Entra grants **+ Azure subscription, Function App, public endpoint, storage, Key Vault** | Entra grants + Snowflake EAI (same pattern already in account) | Entra grants + Openflow deployment |
| Running cost | Azure consumption + storage (not checked) | Serverless task runs (small) | Always-on control pool + runtime (about 2 `CPU_X64_S` instances 24×7) |
| Existing-stack fit | Needs IT approval for Azure | Fits | Fits, but adds a new service with a fixed cost |
| Existing precedent in account | Azure Blob stages exist | `INTUNE_API_ACCESS`, `SHAREPOINT_HC_EAI` | None |

## Recommendation (for the decision in #10, not a decision)

**Recommendation: Option B**, a Snowflake external access integration with a Python stored procedure on a serverless task, polling Graph **delta query** for each Mailbox. It should store the raw MIME and each Attachment on an internal stage in `SUBMISSION_PLATFORM` and key de-duplication on `internetMessageId` (with the Mailbox). Ask IT for the Entra app with `Mail.Read` limited by **RBAC for Applications** to the Data Product Mailboxes, following the same request route as `INTUNE_API_ACCESS` and `SHAREPOINT_HC_EAI`.

Reasons: it stays inside the approved stack, it follows an integration pattern IT has already approved in this account, it has no always-on cost, and minute-level latency should be acceptable for business Submissions. Revisit Option A only if under-a-minute latency becomes a requirement and IT is willing to host Azure compute.

## Not verified

- Azure Functions and Azure storage prices.
- The Snowflake credit rate per `CPU_X64_S` instance-hour (Table 1(d) of the Service Consumption Table) and serverless task credit rates. The cost figures above are relative, not in currency.
- Outlook-specific Graph throttling limits. The throttling page was truncated before the Outlook section, so I could not confirm the per-mailbox numbers.
- Whether `msal` (for certificate auth) is available in Snowflake's Anaconda channel, and whether certificate JWT signing in a procedure works end to end.
- Whether the Openflow `OAuth2 Access Token Provider` controller can do Entra client credentials against `outlook.office365.com` from an SPCS runtime, and whether a network rule for port 993 works for Openflow egress.
- Whether SPCS public endpoints could receive an unauthenticated Graph webhook (I believe they cannot).
- Whether Entra Conditional Access for workload identities in this tenant would block calls from Snowflake egress IPs.
- The current state of the Entra app behind `INTUNE_API_ACCESS` (what permissions it has, whether it could be reused). This should not be assumed; a dedicated app per purpose is cleaner.

## Sources

1. Microsoft Graph, *Get attachment* (permissions: `Mail.Read` application). https://learn.microsoft.com/en-us/graph/api/attachment-get?view=graph-rest-1.0
2. Microsoft Learn, *Role Based Access Control for Applications in Exchange Online*. https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac
3. Microsoft Learn, *Authenticate an IMAP, POP or SMTP connection using OAuth*. https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth
4. Microsoft Graph, *Receive change notifications through webhooks*. https://learn.microsoft.com/en-us/graph/change-notifications-delivery-webhooks
5. Microsoft Graph, *subscription resource type* (lifetime and latency tables). https://learn.microsoft.com/en-us/graph/api/resources/subscription?view=graph-rest-1.0
6. Snowflake, *Openflow connectors*. https://docs.snowflake.com/en/user-guide/data-integration/openflow/connectors/about-openflow-connectors
7. Snowflake, *ConsumeIMAP*. https://docs.snowflake.com/en/user-guide/data-integration/openflow/processors/consumeimap
8. Snowflake, *ExtractEmailHeaders*. https://docs.snowflake.com/en/user-guide/data-integration/openflow/processors/extractemailheaders
9. Snowflake, *ExtractEmailAttachments*. https://docs.snowflake.com/en/user-guide/data-integration/openflow/processors/extractemailattachments
10. Microsoft Graph, *message resource type*. https://learn.microsoft.com/en-us/graph/api/resources/message?view=graph-rest-1.0
11. Microsoft Graph, *Get MIME content of a message*. https://learn.microsoft.com/en-us/graph/outlook-get-mime-message
12. Microsoft Graph, *Get attachment* (raw contents, attachment types). https://learn.microsoft.com/en-us/graph/api/attachment-get?view=graph-rest-1.0
13. Microsoft Graph, *Get incremental changes to messages in a folder*. https://learn.microsoft.com/en-us/graph/delta-query-messages
14. Snowflake, *External network access examples*. https://docs.snowflake.com/en/developer-guide/external-network-access/external-network-access-examples
15. Snowflake, *CREATE SECURITY INTEGRATION (External API Authentication)*. https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-api-auth
16. Snowflake, *Creating User-Defined Functions (UDFs) for DataFrames in Python*, section "Writing files from Snowpark Python UDFs and UDTFs" (limitations note on stored procedures). https://docs.snowflake.com/en/developer-guide/snowpark/python/creating-udfs
17. Snowflake, *Introduction to tasks*. https://docs.snowflake.com/en/user-guide/tasks-intro
18. Snowflake, *All processors (alphabetical)*. https://docs.snowflake.com/en/user-guide/data-integration/openflow/processors
19. Snowflake, *About Openflow - Snowflake Deployments* (available regions). https://docs.snowflake.com/en/user-guide/data-integration/openflow/about-spcs
20. Snowflake, *About Openflow* (deployment types, secrets). https://docs.snowflake.com/en/user-guide/data-integration/openflow/about
21. Snowflake, *Set up Openflow - Snowflake Deployment: Create the execute-as role and external access integrations*. https://docs.snowflake.com/en/user-guide/data-integration/openflow/setup-openflow-spcs-create-rr
22. Snowflake, *Openflow Snowflake Deployment cost and scaling considerations*. https://docs.snowflake.com/en/user-guide/data-integration/openflow/cost-spcs
