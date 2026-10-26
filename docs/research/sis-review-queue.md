# Research: Streamlit in Snowflake (warehouse runtime) for a Review Queue

Question (issue #9): what can a Streamlit-in-Snowflake (SiS) app on the **warehouse runtime** do for a
Review Queue — showing an original Attachment next to extracted values, editing Corrections, knowing who
the viewer is, scoping rows per Data Product Owner, packages, and runtime limits?

Sources are Snowflake docs (fetched 2026-10-26), Streamlit docs/source, and read-only queries against
account `JBSUSA-PILGRIMS_EUROPE_PROD` (AWS_EU_WEST_2). "Verified" below means a primary source states it;
"Inferred" means it follows from sources but was not tested in an app; "Conflict" means sources disagree.

## Summary

- **Identity:** `st.user.user_name` / `st.user.email` give the viewer, in both runtimes. Use them in
  Python and pass them into SQL as bound parameters. This is the only documented-unambiguous way to know
  who the viewer is.
- **Row scoping:** the app runs as its owner role. `CURRENT_ROLE()` and `IS_ROLE_IN_SESSION()` see the
  **owner's** role, so role-based row access policies do not scope by viewer. `CURRENT_USER()`-based row
  access policies are documented as working if the owner role has `READ SESSION` — but another doc page
  says `CURRENT_USER()` returns the owner. Needs a spike before we rely on it.
- **Restricted caller's rights:** not available on the warehouse runtime. Container runtime only.
- **Attachments:** read the bytes with `session.file.get_stream` from an **internal** stage. Show images
  with `st.image`, Excel with `pandas.read_excel` + `st.dataframe`, and PDFs as pages rendered to PNG
  (`pypdfium2`). `st.pdf` needs Components v2, which the warehouse runtime does not support. An iframe of
  a presigned URL is blocked by the CSP.
- **Corrections:** `st.data_editor` + DML (`MERGE`/`INSERT` with bound parameters) as the owner role.
  Owner's-rights stored procedures allow DML.
- **Limits:** 32 MB per frontend message, 200 MB upload, a separate app instance per viewer, sleep timer
  5–240 min (default 15), single-session caching, Anaconda channel only, Streamlit ≤ 1.52.2.

## 1. Who is the viewer, and what does SQL see?

### Owner's rights model (Verified)

- Apps run with the owner role's privileges, the owner's warehouse, and the app's database and schema.
  Viewers need only USAGE on the database, schema and app.
  [owners-rights](https://docs.snowflake.com/en/developer-guide/streamlit/object-management/owners-rights),
  [privileges](https://docs.snowflake.com/en/developer-guide/streamlit/object-management/privileges)
- "All of the privileges of the app owner's role can be used by the app when shared with other roles."
  So anyone who can open the app can do whatever the app's code lets them do as the owner. Authorisation
  per Data Product Owner must therefore live in **app code or in views/policies that key on the viewer's
  user name**, not in Snowflake grants. [owners-rights]
- Warehouse-runtime apps "run as a stored procedure and are subject to the same restrictions as owner's
  rights stored procedures". [owners-rights] This means:
  - only SELECT, DML, DDL, GRANT/REVOKE, variable assignment, DESCRIBE/SHOW (limited) and LIST are
    allowed;
  - `SHOW GRANTS` without an `ON/TO/OF` clause is not allowed, and neither are `SHOW VARIABLES`,
    `SHOW PARAMETERS IN SESSION`, or caller session variables;
  - INFORMATION_SCHEMA views that check the current user can fail with "Requested information on the
    current user is not accessible in stored procedure".
  [stored-procedures-rights](https://docs.snowflake.com/en/developer-guide/stored-procedure/stored-procedures-rights),
  [KB](https://community.snowflake.com/s/article/Stored-Procedure-fails-with-error-Requested-information-on-the-current-user-is-not-accessible-in-stored-procedure)

### `st.user` (Verified)

- In SiS, `st.user.user_name` (the Snowflake user name) and `st.user.email` identify the **viewer**, and
  the docs say this holds in both runtimes. Their documented pattern is
  `session.sql("... WHERE owner = ?", params=[st.user.user_name])` for filtering, and writing
  `created_by = st.user.user_name` on insert.
  [personalization](https://docs.snowflake.com/en/developer-guide/streamlit/app-development/personalization)
- A display name needs `DESCRIBE USER`, which needs MONITOR on the user object or ACCOUNTADMIN. [personalization]

### `CURRENT_USER()` (Conflict)

- The personalization page says: "In Streamlit in Snowflake apps using owner's rights, [CURRENT_USER()]
  is the owner of the Streamlit object, not the viewer. To identify the viewer in your Python code, use
  `st.user` instead." [personalization]
- The row access page says that once `READ SESSION` is granted to the owner role, context functions such
  as `CURRENT_USER()` can be used. Its worked example is a row access policy `the_owner = CURRENT_USER()`
  that returns per-viewer rows (ALICE vs FRANK). This was announced GA on 2024-07-31.
  [row-access](https://docs.snowflake.com/en/developer-guide/streamlit/features/row-access),
  [release note](https://docs.snowflake.com/en/release-notes/2024/other/2024-07-31-sis)
- The security overview repeats: "In warehouse runtimes, if your apps use context functions (such as
  CURRENT_USER()) or access tables with row access policies, grant the global READ SESSION privilege to
  app owner roles."
  [security](https://docs.snowflake.com/en/developer-guide/streamlit/object-management/security)
- **Reading:** the row-access page is specific and has a worked example, so it most likely describes the
  warehouse runtime with `READ SESSION`: `CURRENT_USER()` = viewer. The personalization sentence may
  describe the case without `READ SESSION`, or the container runtime. **Not tested.** Treat it as
  unknown until a spike shows `SELECT CURRENT_USER()` inside a warehouse-runtime app opened by a
  non-owner.

### `CURRENT_ROLE()` / `IS_ROLE_IN_SESSION()` (Verified)

- "You can't use row access policies that use CURRENT_ROLE. … CURRENT_ROLE inside a Streamlit app always
  returns the app owner role." [row-access]
- `IS_ROLE_IN_SESSION` "looks only at the currently active set of roles … The currently active roles can
  differ from the session roles, for example, when executing an owner's rights procedure or a Streamlit."
  [IS_ROLE_IN_SESSION](https://docs.snowflake.com/en/sql-reference/functions/is_role_in_session)
- So role-based policies evaluate against the **owner** role. The viewer would see whatever the owner
  sees. You cannot tell from inside SQL which of the viewer's roles (for example, a Function's role)
  they hold.

### Row access policies under owner's rights (Verified + Inferred)

- A table's row access policy is still evaluated when the app queries it. Whether that scopes per viewer
  depends on what the policy body keys on:
  - `CURRENT_ROLE` / `IS_ROLE_IN_SESSION`: scopes to the owner, so no per-viewer scoping (Verified).
  - `CURRENT_USER()`: per viewer, if the row-access page is right and `READ SESSION` is granted (Conflict
    above).
- For a Review Queue, the practical options are:
  1. A **mapping table** (user name → Data Product) maintained by the platform. App queries filter with
     `st.user.user_name` as a bound parameter. This works regardless of the `CURRENT_USER()` question,
     but enforcement then lives in app code (Inferred).
  2. The same mapping table used inside a row access policy keyed on `CURRENT_USER()`. This is enforced
     by Snowflake, but depends on the spike and on granting `READ SESSION` (Inferred).
- A user name → Data Product mapping cannot be derived from role grants inside the app. `SHOW GRANTS`
  without a target is blocked, and `IS_ROLE_IN_SESSION` sees the owner. `SHOW GRANTS TO USER <name>` is
  an allowed form, but I did not verify what privilege the owner role needs to run it.
  `SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS` is an alternative, but it has latency and needs imported
  privileges (Unverified for this use).

### Restricted caller's rights (Verified)

- "Restricted caller's rights aren't supported in warehouse runtimes." It is container-runtime only,
  Preview, needs Streamlit ≥ 1.53.1 and `st.connection("snowflake-callers-rights")`, and uses the
  viewer's **default** role, not the one selected in Snowsight. Caller grants need `MANAGE CALLER GRANTS`.
  On a warehouse-runtime app, trying to use it raises an error.
  [restricted-callers-rights](https://docs.snowflake.com/en/developer-guide/streamlit/features/restricted-callers-rights)

### This account (Verified, read-only queries 2026-10-26)

- `READ SESSION ON ACCOUNT` is held only by **ACCOUNTADMIN** (`SHOW GRANTS ON ACCOUNT`). No other app
  owner role (ENGINEER_ROLE, JBS_DEVELOPERS, BUILDER_ROLE, …) has it. A Review Queue owner role would
  need ACCOUNTADMIN to grant it.
- `MANAGE CALLER GRANTS` is held by SECURITYADMIN.
- Row access policies exist (3, all owned by ACCOUNTADMIN), so the edition supports them.
- There are 66 Streamlit apps. Runtimes are mixed: for example, `PILGRIMS_AI.SANDBOX.IC_RECON_APP` is
  `SYSTEM$WAREHOUSE_RUNTIME` (Streamlit 1.52.2), and `FINANCE_STORE.BSR.BSR_APP` is
  `SYSTEM$ST_CONTAINER_RUNTIME_PY3_11` on `SYSTEM_COMPUTE_POOL_CPU` with EAI `PYPI_ACCESS`. Some apps
  attach `SNOWFLAKE.SNOWPARK.PYPI_SHARED_REPOSITORY`. So the container runtime is already in use here.

## 2. Showing an original Attachment next to extracted values

| Approach | Warehouse runtime | Basis |
| --- | --- | --- |
| `session.file.get_stream("@db.sch.stage/path", decompress=False).read()` to get the bytes | Works (internal stage) | [KB example in SiS](https://community.snowflake.com/s/article/How-to-load-an-image-from-a-stage-in-a-Streamlit-in-Snowflake-application). External stages are not supported in SiS: [limitations](https://docs.snowflake.com/en/developer-guide/streamlit/limitations) |
| `st.image(bytes)` for PNG/JPEG Attachments | Works | Same KB; the CSP allows data/blob URIs for images: [security] |
| `st.pdf` (`streamlit-pdf`) | **Not expected to work** | `streamlit-pdf` 1.0.7/1.0.8/2.0.0 is in the Snowflake Anaconda channel, but its source mounts via `st.components.v2.component`, and "In warehouse runtimes, Components v2 isn't supported". [streamlit-pdf source](https://github.com/streamlit/streamlit-pdf/blob/main/streamlit_pdf/__init__.py), [limitations]. Not tested. |
| Render PDF pages to PNG server-side (`pypdfium2`, or `pdf2image` + `poppler`), then `st.image` | Expected to work | Packages present in channel (`pypdfium2` 4.19.0 / 5.12.1, `pdf2image`, `poppler`). Not tested in an app. Limitations say ".so files aren't supported"; I read that as user-supplied files, not conda packages, but this is unverified for pypdfium2. |
| PDF text/tables via `pdfplumber` / `pypdf` | Available | In channel |
| Excel: `pandas.read_excel` (openpyxl 3.1.5, xlrd 2.0.2, python-calamine) → `st.dataframe`, one tab per sheet | Expected to work | In channel. Cell formatting and colours are lost. |
| `GET_PRESIGNED_URL` → `st.image(url)` | Probably works for images | The CSP allows images from any HTTPS origin. The URL is a bearer link to S3 until it expires (default 3600 s). `ENFORCE_NETWORK_RULES_FOR_INTERNAL_STAGES` network policies can block it. [GET_PRESIGNED_URL](https://docs.snowflake.com/en/sql-reference/functions/get_presigned_url), [bcr-1558](https://docs.snowflake.com/en/release-notes/bcr-bundles/2024_03/bcr-1558) |
| `<iframe src=presigned_url>` (PDF in browser viewer) | **Blocked** | "The CSP blocks … embedding external content in iframes." The CSP is not configurable. [limitations], [security] |
| `components.v1.html` with a `data:application/pdf` iframe | Unknown | Components v1 is supported. Whether the CSP / sandbox lets the browser PDF viewer render a data-URI PDF was not found in the docs. Spike if needed. |
| `st.download_button(bytes)` to let the reviewer open the original locally | Expected to work | Standard widget; not tested |

Layout: `st.columns([1, 1])` with the Attachment on the left and the extracted values / data editor on
the right is ordinary Streamlit. Each render is bounded by the 32 MB message limit, so render a PDF
page by page rather than as the whole document.

## 3. Editable grids for Corrections

- `st.data_editor` is a core widget. The account's warehouse runtime supports Streamlit up to **1.52.2**
  (supported list: 1.22.0 … 1.52.2), and existing apps pin 1.52.2.
  [dependency-management](https://docs.snowflake.com/en/developer-guide/streamlit/app-development/dependency-management)
- Write-back runs as the owner role. Owner's-rights stored procedures permit **DML**, so `INSERT`/`MERGE`
  into Platform Schema tables works if the owner role has the privileges.
  [stored-procedures-rights]
- Recommended pattern (Inferred):
  1. Diff the edited frame against the original.
  2. Write one Correction row per changed field, with `corrected_by = st.user.user_name`. Use bound
     parameters (`session.sql(..., params=[...])`) or `session.write_pandas` into a staging table and
     then one `MERGE`.
  3. Commit with an explicit button in an `st.form`, so a rerun doesn't write twice.
- Not verified: whether explicit `BEGIN … COMMIT` transactions are allowed in a warehouse-runtime
  session. They are not in the documented list of owner's-rights statements. Prefer single-statement
  `MERGE`s, or call a stored procedure that owns the transaction.
- `CURRENT_USER()` in a column default for audit is unreliable until the conflict above is resolved. Pass
  `st.user.user_name` explicitly.

## 4. Packages and runtime limits (warehouse runtime)

- **Packages:** Snowflake Anaconda channel only, declared via `environment.yml` (channel `snowflake`,
  `=` pins, `*` wildcards). No pip packages, no other channels. Python 3.9/3.10/3.11.
  [dependency-management]
- **Streamlit versions:** a limited list, 1.22.0–1.52.2 at time of writing. No nightly builds. New
  releases lag. [dependency-management]
- **Components:** Components v1 is OK. Components v2 is not supported. The CSP blocks external
  scripts/styles and `eval()`. [limitations]
- **Frontend message limit:** 32 MB per command, not configurable (`MessageSizeError`). [limitations]
- **Uploads:** `st.file_uploader` 200 MB, not configurable. [limitations]
- **Instance model:** a separate, temporary Streamlit server per viewer session, with nothing shared
  between viewers. Caching is single-session only. Startup is slower per viewer.
  [runtime-environments](https://docs.snowflake.com/en/developer-guide/streamlit/app-development/runtime-environments)
- **Session length:** the app suspends after the WebSocket times out (about 15 min by default, account
  level, changed via Support), or after a custom sleep timer `[snowflake.sleep] streamlitSleepTimeoutMinutes`
  of 5–240 min (default 15). Mouse movement resets both. Unsaved edits in `st.session_state` are lost on
  suspend.
  [sleep-timer](https://docs.snowflake.com/en/developer-guide/streamlit/features/sleep-timer)
- **Compute:** `QUERY_WAREHOUSE` runs both the app code and the queries, unless code calls
  `session.use_warehouse(...)`. The warehouse is billed while any viewer has the app open. [runtime-environments]
- **Network:** no outbound access by default. It can be enabled with an EAI plus secrets
  (`_snowflake.get_generic_secret_string`), which is GA for SiS. A Review Queue shouldn't need it.
  [external-access](https://docs.snowflake.com/en/developer-guide/streamlit/features/external-access)
- **Other:** `st.set_page_config` page_title/page_icon/menu_items are ignored. Query params get a
  `streamlit-` prefix in the URL. External stages are not supported. [limitations]
- **Access control extras:** `ALLOWED_INTERFACES = (STREAMLIT)` can restrict a reviewer user to SiS
  apps only. `LOGIN_IDP_REDIRECT` can send app viewers straight to the IdP. [security]

## 5. Warehouse vs container runtime, for this decision

| Need | Warehouse | Container |
| --- | --- | --- |
| Viewer identity in Python (`st.user`) | Yes | Yes |
| Viewer identity in SQL | `CURRENT_USER()` with `READ SESSION` — conflicting docs | Yes, via restricted caller's rights connection (Preview) |
| Viewer's roles in SQL (`CURRENT_ROLE`, `IS_ROLE_IN_SESSION`) | No (owner) | Yes, on the caller's-rights connection (viewer's default role) |
| `st.pdf` | No (CCv2 unsupported) | Expected yes (CCv2 supported) — untested |
| Packages | Anaconda channel | PyPI via artifact repository/EAI (already used in this account) |
| Message / upload limits | 32 MB / 200 MB fixed | Configurable |
| Shared state / cache across reviewers | No | Yes (thread-safety needed) |

## Could not verify

1. What `CURRENT_USER()` returns inside a warehouse-runtime app opened by a non-owner, with and without
   `READ SESSION` on the owner role. The docs conflict. **This blocks any design that puts viewer
   scoping in SQL policies.**
2. Whether `pypdfium2` (native lib) loads in the warehouse runtime, given the ".so files aren't
   supported" limitation.
3. Whether a `data:application/pdf` iframe via `components.v1.html` renders under the SiS CSP.
4. Whether explicit multi-statement transactions work in a warehouse-runtime Snowpark session.
5. What privilege the owner role needs to run `SHOW GRANTS TO USER <name>` from inside the app.
6. `st.pdf` failing on the warehouse runtime is inferred from source plus docs. It was not run.

All six can be settled with one throwaway warehouse-runtime spike app opened by a second user. Creating
it was out of scope for this ticket (no Snowflake objects created).

Note: "Attachment" and "Correction" are used here as in the issue. GLOSSARY.md does not define them yet.
