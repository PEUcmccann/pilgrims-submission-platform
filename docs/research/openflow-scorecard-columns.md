# Openflow C1 and C2 as scorecard columns

Research for issue #38 (part of #31). It restates the earlier Openflow work against the six criteria of map #31. The decision is made later in #40.

- **C1** is the IMAP flow: `ConsumeIMAP` + `ExtractEmailHeaders` + `ExtractEmailAttachments` + `PutSnowflakeInternalStageFile`.
- **C2** is the Graph flow: `InvokeHTTP` against Graph delta query.

Date: 2026-10-29. No new Openflow research was done, as the ticket asked.

## Read this first: the source files are gone

The question named four earlier branches. **None of them exists** on `origin` (checked with `git ls-remote --heads origin` and the GitHub branches API). The links in the closed tickets #23, #24, #25 and #26 now return 404. The only surviving records are:

- `docs/research/mailbox-ingestion-options.md` on `main` (cited as **[M]**), which still has its own source list.
- The summary comments on #23, #24, #25 and #26 (cited as **[#23]**, **[#24]**, **[#25]**, **[#26]**). They state findings and unverified items, but I could not read the full tables or the primary sources behind them.

So every cell below is **second-hand** unless it cites [M]. Where a number or verdict comes only from a comment summary, it is marked "(summary)". If the branches can be recovered (for example from a local clone of whoever wrote them), re-check the cells marked that way against the full files.

Criteria weights and disqualifiers are not agreed yet (#32 is open), so the cells carry no scores.

## Scorecard columns

### 1. Cost

| | C1 (IMAP) | C2 (Graph via `InvokeHTTP`) |
|---|---|---|
| Rate | `CPU_X64_S` is 0.11 credits per instance-hour. In the Service Consumption Table effective 2026-10-15 it is Table 1(f) via 1(h), not 1(d) as [M] says. The account metered 0.1098 credits/hour on the Openflow control pool. [#25] (summary) | Same. [#25] |
| Base scenario (3 Mailboxes, 50 emails/day each, 4,500/month, about 3.4 GB; assumed, not measured) | About 160.6 credits/month for 24x7: 80.3 for the always-on control pool plus 80.3 for one Small runtime. About 109 if the runtime runs business hours only. [#25] (summary) | Same as C1. The same deployment and runtime size are assumed. [#25] (summary) |
| Volume sensitivity | "Volume barely matters." The cost is the fixed control pool plus runtime hours. [#25] (summary) | Same. |
| Fixed cost is the problem | The deployment's control pool "continues to run and incurs costs, even if there are no runtimes running". [M, source 22] | Same. |
| Sharing | One deployment can host several runtimes, so the 80.3-credit control pool can be shared with future uses. [#25] (summary) Shared `INTERNAL_OPENFLOW_0_*` pools metered from 2026-08-30 to 2026-10-08, so someone has used Openflow in this account. Whether it is still wanted is unknown. [#25] (summary) | Same. |
| For comparison | Option B about 68 credits/month at a 5-minute poll and 30 s per poll (about 24 at 15 minutes). Azure Function about GBP 0 to 3 on consumption, about GBP 128 on Premium. [#25] (summary) | Same. |

Unverified: the volumes, the contract credit price (not readable from `ENGINEER_ROLE`, so there is no currency figure), how many runtimes can share one node, and any Openflow cost beyond the compute pools. Telemetry ingest cost is named in [M] but was not priced in the summary. C1 and C2 cost the same, so cost does not separate them.

### 2. IT burden

| | C1 (IMAP) | C2 (Graph) |
|---|---|---|
| Entra and Exchange | App registration with `IMAP.AccessAsApp`, admin consent, `New-ServicePrincipal`, then `Add-MailboxPermission -AccessRights FullAccess` for each Mailbox. Token scope `https://outlook.office365.com/.default`. [M, source 3] [#24] | App registration with `Mail.Read` (application), admin consent, an RBAC for Applications assignment scoped to the Mailboxes, and removal of any tenant-wide `Mail.Read`. [M, source 2] [#24] |
| Snowflake side | An ACCOUNTADMIN creates the Openflow deployment, runtime, execute-as role and EAI. [M, source 21] No deployment exists today (`SHOW OPENFLOW DEPLOYMENTS` returned no rows, per [M] and #22). | Same. |
| Egress | A `HOST_PORT` EGRESS network rule and EAI for `outlook.office365.com:993` and `login.microsoftonline.com:443`. [#24] | The same type of rule for `graph.microsoft.com:443` and `login.microsoftonline.com:443`. The account already has a rule for these two hosts (`INTUNE_DATA.RAW.MICROSOFT_GRAPH_NETWORK_RULE`). [M] |
| New platform to own | Yes: an Openflow deployment, NiFi flow design, and upgrades. See criterion 5. | Same. |
| Blockers | "None documented." [#24] (summary) | None documented. [#24] (summary) |

Unverified: whether IMAP is enabled on the Mailboxes (C1), and whether Conditional Access for workload identities blocks calls from Snowflake egress IPs. The second applies to both columns. [#24]

### 3. Security scoping

| | C1 (IMAP) | C2 (Graph) |
|---|---|---|
| Mailbox scoping | A per-Mailbox allow list through `FullAccess`. **No RBAC for Applications**: Microsoft's page lists only Graph and EWS as supported protocols. [M, source 2] [#24] | RBAC for Applications (`Application Mail.Read` with a management scope or administrative unit). Entra and RBAC grants are a union, so the tenant-wide grant must be removed. Changes can take up to 2 hours. [M, source 2] [#24] |
| Access level | `FullAccess` is wider than read. Whether it can be limited to read-only over IMAP is unverified. [#24] | `Mail.Read` is read-only by name. |
| Credential type | `StandardOauth2AccessTokenProvider` supports client credentials. A certificate or federated credential for the Exchange scope is unverified. [#24] | `MicrosoftClientCertificateOAuth2TokenProvider` documents certificate client credentials for Graph. [#24] (summary) This is better than Option B, whose security integration supports only `CLIENT_SECRET_BASIC` and `CLIENT_SECRET_POST`. [M, source 15] |
| Secret storage | Snowflake secrets exposed to flows through the Snowflake Parameter Provider. External AWS/Azure/GCP secret stores are also supported. [#24] (summary) | Same. Snowflake Workload Identity Federation (no stored secret) is documented for Azure and AWS targets. [#24] (summary) |

Unverified: which native Snowflake secret types the parameter provider exposes, and how a hand-built flow picks up a rotated secret. [#24]

C2 matches Option B on Mailbox scoping. C1 is weaker, and this is the main reason C1 ranks below C2.

### 4. Latency and reliability

| | C1 (IMAP) | C2 (Graph) |
|---|---|---|
| Latency | Polling, so about the poll interval plus run time. [M comparison table] The `ConsumeIMAP` poll interval and any IMAP IDLE behaviour were not covered in the earlier work. | Polling of delta query, same shape as Option B. [M] |
| Receiving mail | `ConsumeIMAP` logs in to `outlook.office365.com:993` with the OAuth2 provider. **Unverified end to end**: the `Authorization Mode` values are not documented. [#24] | Structured JSON, `internetMessageId` and attachment type come with the response. [#23] (summary) |
| Missed-mail tracking | How `ConsumeIMAP` tracks already-read mail is **not documented**, so a missed email's behaviour is unknown. It needs a DEV spike. [#26] (summary) Options are `Mark Messages as Read` and `Delete Messages`. [M, source 7] | The `@odata.nextLink` paging and the per-Mailbox `deltaLink` must be **hand-built in the flow**. [#23] (summary) |
| Throttling | Not covered. | A Graph 429 is not retried by default `InvokeHTTP` routing: 400-499 goes to No Retry. [#23] (summary) Outlook throttling limits are unverified. [M] |
| De-duplication | `ExtractEmailHeaders` gives `email.headers.message-id`. Checked with `SnowflakeDetectDuplicate` or a `MERGE`. [M, sources 8, 18] The durability of the duplicate cache is unverified. [#23] | Same key from `internetMessageId`. |
| After a restart | In-flight data survives restarts. NiFi documents FlowFile replay from provenance. [#26] (summary) | Same. |

### 5. Maintainability and testability

| | C1 (IMAP) | C2 (Graph) |
|---|---|---|
| Duty coverage | Short, linear flow. Gives headers only: no stock processor extracts the body, so that is a Snowflake step. `ExtractEmailAttachments` takes only parts with disposition `attachment`, so inline parts are skipped. [#23] (summary) | Body, `internetMessageId` and attachment type are in the JSON. Paging and `deltaLink` state need hand-built logic. [#23] (summary) |
| Custom code | Python (Medium or Large runtimes only), Groovy (Experimental) or a Java NAR. [#23] (summary) | Same. |
| Version control | A custom flow lives on a NiFi canvas on runtime storage, with no Time Travel or Fail-safe. The documented safeguard is a manual Version > Export. Git promotion (`PUSH TO` / `CREATE ... FROM`) and Terraform are documented only for gen 2 catalog connectors (Public Preview). Whether a hand-built flow can use them is unverified. [#26] (summary) | Same. |
| Upgrades | Snowflake auto-upgrades Snowflake-type deployments. The owner starts a runtime upgrade, which restarts the runtime. The effect on a custom flow's processors is unverified. [#26] (summary) | Same. |
| Monitoring | Logs and metrics go to a Snowflake event table. No alerting recipe is documented. Gen 2 objects cannot join a budget or resource monitor. [#26] (summary) | Same. |
| Test seam (issue #2) | No documented way to run a whole flow against fakes. NiFi `TestRunner` is for Java processors. Workarounds are inference only: a fake IMAP server in a DEV runtime, or a thin flow with the logic tested after the stage. [#26] (summary) | Same, with a fake HTTP server instead. A fake Graph endpoint is the easier fake to build, but that is my inference, not a finding. |
| Gen 2 gaps | No clone, replication, failover, tags, budgets, shares, DCM or lineage. `DROP` is irreversible. [#26] (summary) | Same. |
| Skills | NiFi flow design plus Snowflake SPCS, EAI and event-table knowledge. [#26] (summary) | Same. |

For comparison, a Function or a procedure is Python in GitHub and can be unit tested directly with `pytest` and fakes. [#26] (summary)

### 6. Future reach (SFTP, SharePoint, API transports)

| | C1 and C2 |
|---|---|
| SharePoint | An Openflow SharePoint connector exists. [M, source 6] |
| SFTP, API transports | Not researched in the earlier work. #22 listed them as "not yet specified". NiFi has generic processors, but this research has no verified claim for them. |
| Shared fixed cost | Because one deployment hosts several runtimes, extra transports would reuse the control pool. [#25] (summary) |
| Caveat | Each new transport is another hand-built or catalog flow with the same version-control and test-seam gaps as criterion 5. |

This is where Openflow is strongest, but it is a claim about potential. It is not evidence about the pilot.

### Duty table (carried over from #23)

| Duty | C1 | C2 |
|---|---|---|
| Approved Sender check | Native for a fixed list; table-driven list unverified | Same |
| De-duplicate on `internetMessageId` | Native (stock processors) | Native |
| Keep original `.eml` | Native (raw IMAP bytes) | Native, via `$value` MIME |
| Land `fileAttachment` | Native | Native |
| `itemAttachment` | Unverified | Native (`$value` returns MIME, vCard or iCal) |
| `referenceAttachment` | Not possible | Not possible (`$value` returns HTTP 405) |
| Foreign Attachment flag | Snowflake step after landing | Snowflake step after landing |
| Email body | Snowflake step | Native in JSON |

Source for the whole table: [#23] (summary), consistent with [M] on `referenceAttachment` (source 12).

## Is there a gap that would decide the outcome?

**No new Openflow research is needed to rank C1 and C2 against Option B and the Azure Function.** On the evidence we have:

- Both columns carry a fixed cost of about 160 credits/month in the base scenario, against about 68 for Option B and near zero for the Azure Function on consumption (all from [#25] summary, with assumptions).
- Both add a new platform to own, with the version-control and test-seam gaps in criterion 5.
- C2 is better than C1 on security scoping and avoids the unproven IMAP OAuth login. C1 is better only on flow length.

Two things could change that, and both are questions for a human, not research:

1. **Is Openflow already running and wanted for another purpose?** The metered `INTERNAL_OPENFLOW_0_*` pools suggest it may be. If the control pool is already paid for, the cost difference above shrinks and future reach counts for more. Ask the account owner.
2. **The missing source files.** The cells marked "(summary)" are second-hand. If someone can recover the four branches, a short check would confirm them. It does not change the ranking logic above.

The unverified items that a spike would settle (the IMAP OAuth login, `ConsumeIMAP` missed-mail behaviour, hand-built flow in git promotion) matter only if Openflow becomes competitive after answer 1. I recommend not spending a spike before then.

## Sources

- **[M]** `docs/research/mailbox-ingestion-options.md` on `main`, with its source list (Microsoft Learn, Snowflake Openflow documentation).
- **[#23]** Summary comment on closed issue #23 (capabilities). Full file was on `research/openflow-capabilities` (missing).
- **[#24]** Summary comment on closed issue #24 (Exchange auth). Full file was on `research/openflow-exchange-auth` (missing).
- **[#25]** Summary comment on closed issue #25 (cost). Full file was on `research/ingestion-cost-comparison` (missing).
- **[#26]** Summary comment on closed issue #26 (operations). Full file was on `research/openflow-operations` (missing).
