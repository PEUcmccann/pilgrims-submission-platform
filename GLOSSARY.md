# Enterprise Submission Platform

Turns business Submissions received through a Transport Mechanism into governed Snowflake Data Products.

## Language

**Submission**:
A single business event received through the platform. It can have many Versions. A corrected resend of the same business event is a new Version of the same Submission, not a new Submission.
_Avoid_: Email, message, request

**Version**:
One received state of a Submission. A Submission keeps all its Versions.
_Avoid_: Historical Record (that term means preserved data, not a received state)

**Business Unit**:
An organisational area. It contains one or more Functions.

**Function**:
A team inside a Business Unit. It owns one or more Data Products.

**Data Product**:
A business-owned collection of information delivered for a specific purpose. It has one Mailbox, accepts one or more Document Types, and defines a Business Key and a default Confidence Threshold.

**Document Type**:
A recognised business document format accepted by a Data Product. Each Document Type fills exactly one Dataset and may override the Data Product's Confidence Threshold.

**Dataset**:
A logical collection of records in a Business Schema. It is filled by exactly one Document Type.

**Business Key**:
The fields, defined per Data Product, that identify a business event. Two received emails with the same Business Key value are Versions of one Submission. Matching happens after extraction.
_Avoid_: Thread ID, file name

**Confidence Threshold**:
The minimum Confidence Score for an extraction to be approved automatically. Set per Data Product, with an optional override per Document Type.

**Platform Schema**:
The technical schema `SUBMISSION_PLATFORM`. It is the system of record for every Submission, Extraction, Correction and audit entry.

**Business Schema**:
A schema owned by a Business Unit. It holds only approved business data.

**Review Queue**:
The collection of Versions and Foreign Attachments that need a person. Each Data Product has its own Review Queue, worked by its Data Product Owner.

**Data Product Owner**:
The Function team accountable for the quality and operation of a Data Product. It works the Data Product's Review Queue.

**Late Version**:
A Version that arrives after its Submission was approved. It always goes to the Review Queue. On approval, its data replaces the current data in the Business Schema, and the old data stays as a Historical Record.

**Historical Record**:
Preserved earlier data kept for lineage. Originals and audit entries are kept for as long as the Data Product exists.

**Mailbox**:
The dedicated email address of exactly one Data Product. The Mailbox decides which Data Product a Submission belongs to.
_Avoid_: Inbox, shared mailbox

**Approved Sender**:
A sender authorised to send Submissions to a Data Product's Mailbox. It is a security check, not a routing key.

**Foreign Attachment**:
An Attachment that does not belong to the Data Product of the Submission's Mailbox. The platform flags it for a person to remove or move. The rest of the Submission stays whole.

**Transport Mechanism**:
The channel that delivers a Submission, for example Email, API, SFTP or SharePoint.
_Avoid_: Source system
