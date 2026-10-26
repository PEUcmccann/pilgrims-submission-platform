# Pilgrim's Europe Enterprise Submission Platform

Turns business information that arrives by email into governed Snowflake Data Products.

> Email is a Transport Mechanism. The Submission is the core business concept.

Business teams send invoices, quality certificates, feed plans and similar documents by email. The platform keeps every original, extracts structured data, validates it, routes low-confidence results to a person for review, and writes approved data into the Business Schema of the owning Business Unit. Every step is audited and can be replayed.

**Status:** design phase. The domain model and architecture decisions are agreed. There is no implementation code yet. The build spec is [issue #2](https://github.com/Pilgrims-Europe/pilgrims-submission-platform/issues/2). Its open decisions are being worked through the pilot map, [Pilot: Agri Feed Planning and 3PL running end-to-end in DEV](https://github.com/Pilgrims-Europe/pilgrims-submission-platform/issues/4).

---

## Contents

- [How it works](#how-it-works)
- [Domain model](#domain-model)
- [Architecture](#architecture)
- [Snowflake structure](#snowflake-structure)
- [Key rules](#key-rules)
- [Processing principles](#processing-principles)
- [Onboarding a Data Product](#onboarding-a-data-product)
- [Testing approach](#testing-approach)
- [Out of scope](#out-of-scope)
- [Repository layout](#repository-layout)
- [Working in this repo](#working-in-this-repo)

---

## How it works

1. A sender emails a document to the **Mailbox** of a Data Product.
2. Mailbox ingestion checks the sender is an **Approved Sender**, discards duplicate deliveries, and stores the message, body and Attachments in the Platform Schema as a new **Version** of a **Submission**.
3. Deterministic parsing runs first. **Snowflake AI extraction** runs when parsing alone cannot read the document. Each extraction has a **Confidence Score**.
4. The **Business Key** of the extracted data is matched against existing Submissions. A match makes this Version part of that Submission. No match makes a new Submission.
5. **Validation Rules** run. If every rule passes and the Confidence Score meets the **Confidence Threshold**, the Version is approved automatically.
6. Everything else goes to the Data Product's **Review Queue**. The **Data Product Owner** reviews it in a Streamlit portal and can correct, approve or reject it.
7. Approved data is routed by metadata into the correct **Dataset** in the owning **Business Schema**.
8. Silver and Gold layers build on the Business Schemas for Qlik Sense, Power BI and AI applications.

---

## Domain model

```text
Business Unit
  └── Function                     (a team inside a Business Unit)
        └── Data Product           (one Mailbox, one Business Key, default Confidence Threshold)
              ├── Document Type    (optional Confidence Threshold override)
              │     └── Dataset    (exactly one per Document Type, in a Business Schema)
              └── Submission       (one business event)
                    └── Version(s) (each received state of the Submission)
                          ├── Email (Transport Mechanism)
                          ├── Attachment(s)
                          ├── Extraction(s)
                          ├── Validation Result(s)
                          ├── Correction(s)
                          └── Review Decision(s)
```

All terms are defined in [`GLOSSARY.md`](GLOSSARY.md). Use those words, and avoid the listed alternatives, in code, docs and tickets.

---

## Architecture

```text
Exchange Online
      │
      ▼
Dedicated Mailbox (one per Data Product)
      │
      ▼
Graph API / Azure Function ──► Approved Sender check, de-duplication, Foreign Attachment flag
      │
      ▼
<ENV>_RAW.SUBMISSION_PLATFORM            (Platform Schema: system of record)
      │
      ▼
Deterministic parsing ──► Snowflake AI extraction
      │
      ▼
Validation (Validation Rules + Confidence Threshold)
      │
      ├── Auto approved
      └── Review Queue ──► Streamlit Review Portal ──► Approval / Correction / Rejection
      │
      ▼
Routing (Data Product + Document Type ──► Dataset)
      │
      ▼
Business Schemas: FEED_PLANNING · COMMERCIAL · QUALITY · FINANCE · AGRICULTURE
      │
      ▼
Silver ──► Gold ──► Qlik Sense / Power BI / AI applications
```

| Stage | Owner technology | Responsibility |
|---|---|---|
| Mailbox ingestion | Microsoft (Exchange Online, Graph API, Azure Function) | Read Mailboxes, check senders, land originals |
| Extraction | Snowflake (deterministic parsing, then Cortex AI extraction) | Turn Attachments into values with Confidence Scores |
| Validation | Snowflake | Apply Validation Rules and the Confidence Threshold |
| Review | Streamlit in Snowflake | Let the Data Product Owner correct, approve or reject |
| Routing | Snowflake, metadata-driven | Write approved data to the right Dataset |

---

## Snowflake structure

Raw databases are environment-prefixed, for example `DEV_RAW`.

**Platform Schema**, the system of record for all platform artefacts:

```text
<ENV>_RAW.SUBMISSION_PLATFORM
├── SUBMISSION
├── ATTACHMENT
├── EMAIL_BODY
├── EXTRACTION
├── AUDIT
└── TRAINING_FEEDBACK
```

**Business Schemas**, owned by Business Units and holding approved data only:

```text
<ENV>_RAW.FEED_PLANNING
<ENV>_RAW.COMMERCIAL
<ENV>_RAW.QUALITY
<ENV>_RAW.FINANCE
<ENV>_RAW.AGRICULTURE
```

Security on business data is owned by the Business Schema, not by the platform.

---

## Key rules

Each rule is recorded as an Architecture Decision Record in [`docs/adr/`](docs/adr/).

| ADR | Rule |
|---|---|
| [0001](docs/adr/0001-platform-schema-is-system-of-record.md) | `SUBMISSION_PLATFORM` is the system of record for Submissions, Extractions, Corrections and audit. Business Schemas hold approved data only. |
| [0002](docs/adr/0002-mailbox-decides-data-product.md) | Each Data Product has its own Mailbox, and the Mailbox decides the Data Product. A file that belongs elsewhere is a Foreign Attachment: it is flagged, and the rest of the Submission continues. |
| [0003](docs/adr/0003-business-key-decides-versioning.md) | Each Data Product defines a Business Key. A resend with the same Business Key is a new Version of the same Submission, not a new Submission. |
| [0004](docs/adr/0004-late-version-needs-human-approval.md) | A Late Version (one that arrives after approval) always needs human approval. On approval it replaces the current data, and the old data is kept as a Historical Record. |

Also agreed:

- **Snowflake-native extraction.** Snowflake AI extraction is the primary engine. Microsoft technology handles only mailbox integration, and extraction runs as close as possible to governed data.
- **Metadata-driven onboarding.** New Data Products are added by configuration, not code.
- **Retention.** Originals and audit entries are kept for as long as the Data Product exists.

---

## Processing principles

1. Preserve all originals.
2. Run deterministic parsing before AI.
3. Use AI only when it is required.
4. Send low-confidence outcomes to human review.
5. Store every Correction.
6. Keep improving extraction quality.
7. Keep complete auditability.
8. Support replay and Reprocessing.
9. Let the destination Business Schema own security.
10. Collect platform metrics per Data Product.

---

## Onboarding a Data Product

A new Data Product is configuration only. It needs:

| Metadata | Example |
|---|---|
| Business Unit | Finance |
| Function | Accounts Payable |
| Data Product | Supplier Invoices |
| Owner (Data Product Owner) | Accounts Payable team |
| Mailbox | a dedicated address for this Data Product |
| Approved Senders | list of allowed sender addresses or domains |
| Document Types | Invoice, Credit Note |
| Dataset per Document Type | `FINANCE.SUPPLIER_INVOICE`, `FINANCE.SUPPLIER_CREDIT_NOTE` |
| Business Key | supplier ID + invoice number |
| Confidence Threshold | Data Product default, with optional per-Document-Type override |

The examples are illustrative only. They are not configured Data Products.

---

## Testing approach

Each of the five stages (Mailbox ingestion, Extraction, Validation, Review, Routing) has its own test seam. Tests check behaviour at the stage boundary: given these input records, these output records and states exist. The Graph API and AI extraction are replaced by fakes at the boundary, and real calls run only in a small set of smoke tests. See [issue #2](https://github.com/Pilgrims-Europe/pilgrims-submission-platform/issues/2) for details.

---

## Out of scope

Out of scope for the first build:

- Transport Mechanisms other than Email (API, SFTP, SharePoint). The model allows them.
- Retraining extraction from Corrections. Corrections are stored, not yet used.
- Automatic splitting of mixed emails into several Submissions.
- Silver and Gold modelling, and the reports built on them.
- A user interface for editing Data Product configuration.

---

## Repository layout

```text
.
├── README.md            this file
├── AGENTS.md            instructions for coding agents
├── GLOSSARY.md          the domain language (ubiquitous language)
└── docs/
    ├── adr/             Architecture Decision Records
    └── agents/          issue tracker, triage labels and domain-doc conventions
```

---

## Working in this repo

- **Issues and specs** live in [GitHub Issues](https://github.com/Pilgrims-Europe/pilgrims-submission-platform/issues). See [`docs/agents/issue-tracker.md`](docs/agents/issue-tracker.md).
- **Triage labels:** `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See [`docs/agents/triage-labels.md`](docs/agents/triage-labels.md).
- **Language:** when a new domain term is agreed, add it to `GLOSSARY.md`. When a hard-to-reverse decision is made, add an ADR.
- **Current effort:** the wayfinder map [Pilot: Agri Feed Planning and 3PL running end-to-end in DEV](https://github.com/Pilgrims-Europe/pilgrims-submission-platform/issues/4). Run `/wayfinder` with the map to take the next frontier ticket. Research findings live on `research/*` branches and are linked from their tickets.
