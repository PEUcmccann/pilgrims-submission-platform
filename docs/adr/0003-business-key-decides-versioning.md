# Business Key per Data Product decides Submission versioning

Each Data Product defines its own Business Key. A resend with the same Business Key is a new Version of the same Submission. Matching runs after extraction, so a resend is first stored as received and then linked. We rejected email thread and file name matching because both break when a sender starts a new email or renames a file.
