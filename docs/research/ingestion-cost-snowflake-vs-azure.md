# Mailbox ingestion cost: Snowflake serverless task (Option B) vs Azure Function (Option A) vs Openflow

Research for #36 (part of #31). Written 2026-10-29. Planning only; nothing was built.

## Answer

For the base scenario, **Option A (Azure Function) costs about USD 3.5-4 a month (about GBP 2.7-3)**, and the cost barely moves with the polling interval. **Option B costs 9 credits a month at a 15-minute poll, 21 at 5 minutes and 90 at 1 minute** (about USD 37, 83 and 359 at an assumed USD 4.00 per credit). **Openflow costs about 161 credits a month (about USD 640)**, almost all of it fixed.

- Option B's cost is dominated by **idle polling runs**, not by email volume. Only the run time per idle poll is unmeasured, and it is the biggest swing factor (see "Sensitivity").
- Everything marked **[E]** below is an estimate. Everything marked **[M]** was measured in this account. Everything marked **[P]** comes from a published price page.

## What could not be reused

The task said to reuse the base scenario and Openflow figures from `origin/research/ingestion-cost-comparison` (`docs/research/ingestion-cost-comparison.md`). **That branch does not exist on the remote or locally** (`git ls-remote origin` lists only `main`, `chore/agent-skills-setup`, `docs/domain-model-and-readme`, `research/mailbox-ingestion-options`, `research/sis-review-queue`, `research/snowflake-ai-extraction`). I could not check it, so:

- The **base scenario** is taken from the wording in #36: 3 Mailboxes, 50 emails a day each, about 4,500 emails a month, about 3.4 GB a month.
- The **Openflow figure** is rebuilt from published rates and cross-checked against this account's own metering (below), rather than copied from the missing branch.

If that branch turns up, compare its Openflow number with section "Openflow" before merging.

## Base scenario

| Item | Value | Source |
| --- | --- | --- |
| Mailboxes | 3 | #36 |
| Emails per Mailbox per day | 50 | #36 |
| Emails per month | about 4,500 (3 x 50 x 30) | #36 |
| Data landed per month | about 3.4 GB (original `.eml` plus Attachments) | #36 |
| Average per email | about 0.75 MB | derived |
| Files per month | about 11,250 (2.5 per email: `.eml` plus Attachments) | [E] assumption |
| Month | 30 days = 720 h | assumption |
| Snowflake account | AWS eu-west-2 (London) | [M] `CURRENT_REGION()` = `AWS_EU_WEST_2` |

Out of scope for every design (the same in all of them): Microsoft Graph calls (no per-call charge), Extraction, Validation, Review, Routing, and Exchange Online licensing.

## Prices used

### Snowflake credits to currency

| Item | Value | Source |
| --- | --- | --- |
| On-demand price per credit, AWS Europe (London) | Standard USD 2.70, Enterprise USD 4.00, Business Critical USD 5.40 | [P] Service Consumption Table, effective 15 Oct 2026, Table 2(a) |
| **Price used in this document** | **USD 4.00 (Enterprise on-demand)** | [E] assumption |
| Contract price per credit | **Not found.** `SNOWFLAKE.ORGANIZATION_USAGE.RATE_SHEET_DAILY` and `ACCOUNTS` returned no rows for this role. The edition is also unverified. | [M] |
| Stage storage, AWS Europe (London), on demand | USD 24 per TB per month | [P] Table 3(a) |
| Serverless Tasks | 0.9 x Compute-Hour (plus Cloud Services multiplier 1) | [P] Table 5 |
| Serverless Tasks Flex | 0.5 x Compute-Hour (plus Cloud Services multiplier 1) | [P] Table 5 |
| Openflow Snowflake Deployment, `CPU_X64_S` | 0.11 credits per instance-hour | [P] Table 1(f) |
| Snowpipe | 0.0037 credits per GB | [P] Table 5 |
| Directory/external table automated refresh | 1.25 x compute + 0.06 credits per 1,000 files | [P] Table 5 |

Edition drives the answer by up to 2x. Use the contract price once known. Credits are the edition-independent figure.

### Azure (UK South)

All Azure figures are from the Azure Retail Prices API (`prices.azure.com`, `armRegionName eq 'uksouth'`, `priceType eq 'Consumption'`) and the official pricing pages, fetched 2026-10-29.

| Item | USD | GBP | Source |
| --- | --- | --- | --- |
| Functions Consumption, execution time | 0.000016 per GB-s after 400,000 GB-s free a month | n/a | [P] azure.microsoft.com/en-gb/pricing/details/functions |
| Functions Consumption, executions | 0.20 per million after 1 million free a month | n/a | [P] same |
| Blob Hot LRS storage | 0.0192 per GB per month | 0.0145 | [P] Retail Prices API |
| Blob Hot LRS write operations | 0.059 per 10K | 0.0445 | [P] same |
| Blob read / other operations | 0.0047 per 10K | 0.0035 | [P] same |
| Key Vault Standard, secret operations | 0.03 per 10K | 0.0226 | [P] azure.microsoft.com/en-gb/pricing/details/key-vault; no monthly vault fee listed for Standard |
| Log Analytics, Analytics Logs ingestion | 2.88 per GB after the first 5 GB a month | 2.1737 | [P] Retail Prices API (tier at 5 GB). The Azure Monitor page lists USD 2.30 per GB (a different region/base price). The API UK South value is used here. |
| Data transfer out of Azure | free for the first 100 GB a month | free | [P] Retail Prices API, Bandwidth, tier at 100 GB |
| GBP/USD | 0.7547 | | [E] implied by Key Vault and Log Analytics rows (GBP 0.0226 / USD 0.03) |

The same page also lists **Flex Consumption** (free grant 100,000 GB-s and 250,000 executions a month, USD 0.000026 per GB-s on demand). At this volume either plan stays inside its free grant, so the plan choice does not change the answer.

## Option B: Snowflake serverless task (EAI + Python procedure + task)

### Design costed

A serverless task runs a Python procedure on a schedule. Each run gets a token, calls Graph delta query for each of the 3 Mailboxes, and for each new email downloads the MIME and Attachments, writes them to an internal stage, and inserts metadata. (Design: `docs/research/mailbox-ingestion-options.md`, Option B.)

### Run-time assumptions (all [E] - not measured)

| Parameter | Base | Low | High |
| --- | --- | --- | --- |
| Fixed time per poll (procedure start, token, 3 delta calls) | 8 s | 4 s | 15 s |
| Extra time per new email (MIME + Attachment download, stage write, insert) | 3 s | 3 s | 3 s |

Credits = (polls x fixed + 4,500 x 3 s) / 3600 x 0.9. Billing is per second with no one-minute floor, which matches this account's history (the smallest task runs below bill about 0.00005 credits).

### Result (base scenario, credits per month)

| Poll interval | Runs a month | Compute seconds (base) | **Credits, base [E]** | Credits, low [E] | Credits, high [E] |
| --- | --- | --- | --- | --- | --- |
| 1 minute | 43,200 | 359,100 | **89.8** | 46.6 | 165.4 |
| 5 minutes | 8,640 | 82,620 | **20.7** | 12.0 | 35.8 |
| 15 minutes | 2,880 | 36,540 | **9.1** | 6.3 | 14.2 |

Email-handling work alone is 4,500 x 3 s = 13,500 s = 3.4 credits a month. The rest is idle polling.

### Internal stage storage

3.4 GB a month, never deleted: USD 24 per TB per month [P] gives **USD 0.08 a month in month 1, rising to about USD 1.00 a month after 12 months** (40.8 GB). [E] volume, [P] price. Compression is ignored.

### Currency (credits x USD 4.00, plus stage storage, month 1)

| Poll interval | Credits | USD | GBP (x 0.7547) | USD at Standard (2.70) |
| --- | --- | --- | --- | --- |
| 1 minute | 89.8 | **359** | **271** | 242 |
| 5 minutes | 20.7 | **83** | **62** | 56 |
| 15 minutes | 9.1 | **37** | **28** | 25 |

## Option A: Azure Function + Graph API (Consumption plan)

### Design costed

A timer-triggered Python Function polls Graph delta query for the 3 Mailboxes, writes the `.eml`, Attachments and a metadata JSON to Azure Blob storage, and Snowflake loads from there. This matches the Option A sketch in `mailbox-ingestion-options.md`; the exact landing mechanism (Snowpipe vs COPY) is not decided there, so Snowpipe plus a directory-table refresh is assumed here ([E]).

### Assumptions (all [E])

- 0.5 GB billed memory, 3 s per idle poll, plus 3 s per new email (so 4,500 x 3 s of busy time).
- Function reads 3 secrets per run from Key Vault.
- Default Application Insights sampling, about 5-10 KB of telemetry per run.
- Storage account doubles as the Functions host storage; host-storage operations are guessed at about USD 0.50 a month.

### Result (USD a month, month 1)

| Line | 1 min | 5 min | 15 min | Basis |
| --- | --- | --- | --- | --- |
| Function compute | 0.00 | 0.00 | 0.00 | 71,550 / 19,710 / 11,070 GB-s, all under the 400,000 GB-s free grant. Without the grant: 1.14 / 0.32 / 0.18. |
| Function executions | 0.00 | 0.00 | 0.00 | 43,200 / 8,640 / 2,880, under the 1 million free |
| Key Vault operations | 0.39 | 0.08 | 0.03 | 129,600 / 25,920 / 8,640 secret ops x USD 0.03 per 10K |
| Blob data stored | 0.07 | 0.07 | 0.07 | 3.4 GB x 0.0192 (about 0.78 by month 12) |
| Blob operations | 0.13 | 0.13 | 0.13 | about 22,500 writes x 0.059 per 10K, plus small reads |
| Host-storage operations | 0.50 | 0.50 | 0.50 | [E] guess |
| Monitoring | 0.00 | 0.00 | 0.00 | under 0.5 GB of telemetry a month, inside the first 5 GB free |
| Data transfer out of Azure | 0.00 | 0.00 | 0.00 | 3.4 GB, inside 100 GB free |
| **Azure subtotal** | **1.09** | **0.78** | **0.73** | |
| Snowflake side (see below) | 2.76 | 2.76 | 2.76 | |
| **Total, USD** | **3.85** | **3.54** | **3.49** | |
| **Total, GBP** | **2.9** | **2.7** | **2.6** | |

Snowflake side of Option A: Snowpipe 3.4 GB x 0.0037 = 0.013 credits, plus directory-table refresh 11,250 files x 0.06 / 1000 = 0.68 credits (compute multiplier 1.25 on a small refresh ignored). About **0.69 credits = USD 2.76** at USD 4.00. This is the one place where Option A spends Snowflake credits, and it is a guess. If the Function writes straight to an internal stage instead, this drops to about 0 but a warehouse is then needed to insert metadata, which would cost more (an XS resume per batch bills a 60-second minimum).

Not costed (not usage-priced, or not per-month): the Entra app registration, the engineering effort to run and patch the Function, IT time, and a private network setup (VNet, private endpoint) if security requires one. Standard-tier Key Vault has no monthly vault fee in the price page; a Premium or HSM-backed key would add cost.

## Openflow (reconstructed, not copied)

The earlier branch figure was not available (see "What could not be reused"), so this is rebuilt.

**From published rates [P/E]:** one always-on management pool (1 x `CPU_X64_S`) plus one small runtime running 24x7 (1 x `CPU_X64_S`) is 2 instance-hours per hour. 2 x 0.11 x 730 h = **160.6 credits a month**, about **USD 642 / GBP 485** at USD 4.00. Source: Openflow cost page (`docs.snowflake.com/en/user-guide/data-integration/openflow/cost-spcs`) and Table 1(f). Telemetry Data Ingest (0.0212 credits per GB) is extra and not estimated.

**From this account's metering [M]:** `SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY`, `SERVICE_TYPE = 'OPENFLOW_COMPUTE_SNOWFLAKE'`:

| Month | `INTERNAL_OPENFLOW_0_CONTROL_POOL` | `INTERNAL_OPENFLOW_0_SMALL` | Total |
| --- | --- | --- | --- |
| Dec 2025 (31 d) | 81.75 | 81.75 | 163.5 |
| Jan 2026 (31 d) | 81.74 | 81.74 | 163.5 |
| Jun 2026 (30 d) | 79.10 | 79.10 | 158.2 |

Measured rate is 0.1099 credits per instance-hour, matching the published 0.11. A full month costs 158-164 credits, so the rebuilt figure agrees with real billing. The last Openflow row is 2026-10-08 (control pool only), so there is no live deployment now. This was not a mailbox flow as far as the metering can tell (it carries no flow name), so it is a cost check, not a measurement of a mailbox design. Openflow has no usage-based element at this volume: 4,500 emails a month is a rounding error against 160 credits of standing cost.

## Comparison

| Design | Credits / month | USD / month | GBP / month | Nature |
| --- | --- | --- | --- | --- |
| **A. Azure Function** (any interval) | about 0.7 (Snowflake side) | **3.5 - 3.9** | **2.6 - 2.9** | [E] mostly free-grant, small fixed items |
| **B. Serverless task, 15 min** | 9.1 (range 6.3-14.2) | 37 (25-57) | 28 | [E] |
| **B. Serverless task, 5 min** | 20.7 (12.0-35.8) | 83 (48-143) | 62 | [E] |
| **B. Serverless task, 1 min** | 89.8 (46.6-165.4) | 359 (186-662) | 271 | [E] |
| **C. Openflow** (2 x `CPU_X64_S` 24x7) | 160.6 | 642 | 485 | [P], cross-checked [M] |

USD columns use USD 4.00 per credit and ranges use the low/high run-time assumptions. GBP is USD x 0.7547.

## Sanity check against this account (read-only)

- 90-day metering: `SERVERLESS_TASK` used only 0.67 credits across 8 tasks (124 runs). Of those, 124 runs billed between about 0.00005 and 0.0125 credits each. So a short task run bills a tiny amount, and the per-run floor is not the problem; the risk is a Python procedure that runs for seconds, 43,200 times a month.
- The whole account used about 12,950 credits in the last 90 days (about 4,300 a month). Option B at 5 minutes (about 21 credits) would add about 0.5%. Openflow at 161 credits would add about 3.7%.
- Existing serverless task cost relevant to the platform: none. No Mailbox task exists yet.
- Contract price per credit is not visible (see above).

## Sensitivity and what would change the answer

- **Option B's cost is almost entirely the fixed time per idle poll.** Going from 8 s to 15 s lifts the 15-minute figure from 9 to 14 credits; from 8 s to 4 s drops it to 6. The only way to remove the guess is to measure the actual procedure.
- **Flex serverless tasks** (Table 5 multiplier 0.5 vs 0.9) would cut Option B compute by about 44% if they suit this workload. Their scheduling behaviour was not checked here.
- **Longer intervals are cheap to run but slower to react.** Moving 5 to 15 minutes saves about 11 credits a month (about USD 45).
- **The Function's cost is flat** because the free grants (400,000 GB-s and 1 million executions per subscription, shared across all apps) cover this volume. If other Function apps in the same subscription use the grant, the Function would cost up to about USD 1.2 a month more at 1-minute polling. That is still small.
- **Credit price.** At Standard (USD 2.70) all Snowflake-side USD figures fall by 32%; at Business Critical (USD 5.40) they rise by 35%. This does not change the ranking.
- **Volume scaling.** All designs scale gently with email volume except Openflow (fixed). At 10x volume Option B adds about 30 credits (the email-handling part grows from 3.4 to 34), still below Openflow.

## Not decided here

This note gives costs only. Cost is one of six criteria in #31 (IT burden, security scoping, latency, maintainability, future reach are the others). On cost alone: A is cheapest by an order of magnitude, B at 15 minutes costs about USD 30-35 a month more, and Openflow is the most expensive and fixed. The gap between A and B at 15 minutes is small against the account's size; whether it matters depends on the other criteria.

## Gaps and proposed follow-up

1. **Measure Option B's run time.** A DEV proof (a Graph call from a Snowflake procedure) would replace the 8 s / 3 s guesses with `SERVERLESS_TASK_HISTORY` credits per run. This is the single biggest uncertainty. The #31 map already allows a DEV proof.
2. **Confirm the contract price per credit and edition** with whoever owns the Snowflake contract.
3. **Find or restore `research/ingestion-cost-comparison`** and reconcile its Openflow number with the rebuilt one above.
4. **Decide Option A's landing path** (Blob + Snowpipe vs direct PUT); it moves Option A by about 0.7 credits, or more if a warehouse is needed.
5. Azure GBP figures use an implied exchange rate. Re-check with the Azure pricing calculator before quoting to a budget holder.

## Sources

1. Snowflake, *Service Consumption Table*, effective 15 October 2026. https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf (Tables 1(f), 2(a), 3(a), 5)
2. Snowflake, *Openflow Snowflake Deployment cost and scaling considerations*. https://docs.snowflake.com/en/user-guide/data-integration/openflow/cost-spcs
3. Microsoft, *Azure Functions pricing*. https://azure.microsoft.com/en-gb/pricing/details/functions/
4. Microsoft, *Key Vault pricing*. https://azure.microsoft.com/en-gb/pricing/details/key-vault/
5. Microsoft, *Azure Monitor pricing*. https://azure.microsoft.com/en-gb/pricing/details/monitor/
6. Microsoft, *Azure Retail Prices API* (UK South, USD and GBP), queried 2026-10-29. https://prices.azure.com/api/retail/prices
7. Account metering, read-only: `SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY`, `SERVERLESS_TASK_HISTORY` (queried 2026-10-29).
8. `docs/research/mailbox-ingestion-options.md` (Options A, B and C design descriptions).
