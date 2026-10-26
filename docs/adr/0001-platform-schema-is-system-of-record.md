# Platform Schema is the system of record; Business Schemas hold approved data only

Email is a transport, not a source system. `SUBMISSION_PLATFORM` records every Submission, Extraction, Correction and audit entry, so replay and reprocessing work from one place. Business Schemas receive only approved, routed data and own their own security. We rejected splitting platform records into per-owner schemas, because that scatters audit and replay across many schemas.
