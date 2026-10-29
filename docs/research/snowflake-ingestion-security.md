# Security comparison: Snowflake ingestion (Option B) versus Azure Function (Option A)

Research for issue #35 (part of #31). The decision is made later by a human; this note only compares security and lists what IT must clear.

Question: compare the security of Option B (external access integration, `SECRET` / `API_AUTHENTICATION` security integration, Python procedure on a task) with Option A (Azure Function calling Graph) for reading the **Mailboxes** that feed the **Platform Schema**.

Date of research: 2026-10-29. Sources are Microsoft Learn and Snowflake documentation, plus read-only SQL against account `JBSUSA-PILGRIMS_EUROPE_PROD` (AWS `eu-west-2`) as `ENGINEER_ROLE`. Nothing was changed. Facts from `docs/research/mailbox-ingestion-options.md` (#7) are reused, not redone.

Prior art note: the ticket names the branch `origin/research/openflow-exchange-auth`. It does **not exist on origin** (`git ls-remote` lists only `mailbox-ingestion-options`, `sis-review-queue`, `snowflake-ai-extraction` under `research/`), so it was not read. Nothing here depends on it.

## Summary

- **Mailbox scoping is identical in both options.** RBAC for Applications in Exchange Online scopes `Mail.Read` to a management scope or administrative unit. It is the same grant whether the caller is a Function or a Snowflake procedure. The one trap is the same too: any tenant-wide `Mail.Read` consent in Entra ID cancels the scope. [1]
- **The main difference is where the credential lives and who can touch it.** Option A can use a managed identity for Graph, so no Graph secret exists. Option B stores a client secret inside Snowflake, and in this account that means a role held by 28 users and 2 roles. [5][14][S5]
- **The existing Snowflake pattern is weaker than the documented one.** `INTUNE_API_ACCESS` and `SHAREPOINT_HC_EAI` hold raw `GENERIC_STRING` secrets owned by `ENGINEER_ROLE`, not an `API_AUTHENTICATION` integration. Copying them for Mailbox read would put a Mailbox-reading credential in a broadly held role. [S3][S5]
- **Certificate auth is not available through the Snowflake security integration** (only `CLIENT_SECRET_BASIC` / `CLIENT_SECRET_POST`). Microsoft says to use certificates, not client secrets. [4][5]
- **There is a secret-free route for Option B:** Snowflake can act as an OIDC provider (`SECRET TYPE = WORKLOAD_IDENTITY_FEDERATION`) and Entra can trust it through a federated identity credential. This removes the stored client secret. It is documented on each side, but **the end-to-end flow from a procedure is not verified** and needs a DEV proof. [6][7][8]
- **Conditional Access for workload identities could block Snowflake, but only if IT has such a policy and it is location-based.** Snowflake publishes stable egress IP ranges for AWS, so an allowlist is possible. Those ranges expire and must be refreshed. Managed identities (Option A) are not covered by this policy. [9][10][11]
- **Egress control in Snowflake is host-level only** (`graph.microsoft.com:443`), not path-level. Mailbox scoping therefore has to be enforced in Exchange, not in the network rule. [12]

## 1. Mailbox scoping (RBAC for Applications)

| | A. Azure Function | B. Snowflake procedure |
|---|---|---|
| Mechanism | `New-ServicePrincipal`, `New-ManagementRoleAssignment -Role "Application Mail.Read"` with `-CustomResourceScope` or `-RecipientAdministrativeUnitScope` | Same |
| Needs | Organization Management role group; Exchange Administrator in Entra | Same |
| Trap | Tenant-wide `Mail.Read` in Entra ID is additive and defeats the scope; remove it | Same |
| Propagation | 30 minutes to 2 hours; `Test-ServicePrincipalAuthorization` bypasses the cache | Same |
| Protocols | Graph and EWS only | Same (Graph) |

Source: [1]. Nothing about scoping favours either design. What differs is the **blast radius if the credential leaks**: with a correct scope, a leaked credential reads only the Data Product Mailboxes in both designs. With a wrong scope (tenant-wide consent left in place), a leaked Snowflake secret reads every mailbox in the tenant. Option A with a managed identity has the same exposure but the identity cannot be copied out of Azure. [1][5]

Unverified: that RBAC for Applications accepts a managed identity's service principal in `New-ServicePrincipal`. The page says the pointer is to an Entra service principal and does not mention managed identities either way. [1]

## 2. Secret handling

### Option B: what Snowflake offers

- Documented best practice: for OAuth targets, store a `SECRET TYPE = OAUTH2` that points at an `API_AUTHENTICATION` security integration. The integration holds `OAUTH_CLIENT_ID` and `OAUTH_CLIENT_SECRET`; the handler calls `_snowflake.get_oauth_access_token`. Snowflake's own example for Microsoft Graph uses `OAUTH_GRANT = 'CLIENT_CREDENTIALS'`, token endpoint `login.microsoftonline.com/<tenant>/oauth2/v2.0/token` and scope `https://graph.microsoft.com/.default`. [4][12][13]
- A procedure or UDF can only use a secret if its creator holds `READ` on the secret and `USAGE` on the integration, and the secret is listed in `ALLOWED_AUTHENTICATION_SECRETS`. Snowflake describes this as the way to audit and manage credential use. [12]
- Creating the integration needs `CREATE INTEGRATION` (ACCOUNTADMIN by default) or `CREATE SECURITY INTEGRATION` for the API auth type. [4]
- Calls are logged in `SNOWFLAKE.ACCOUNT_USAGE.EXTERNAL_ACCESS_HISTORY`. [12]

### Option A: what Azure offers

- Microsoft's checklist: use certificate credentials, not client secrets; never store credentials in code or config; use managed identities or Key Vault to store and rotate them. [5]
- Key Vault guidance: Azure RBAC (not legacy access policies), managed identities, soft delete and purge protection, firewall or private endpoint, audit logging, one vault per application and environment. [14]
- Option A in #7 also needs a Snowflake credential (key pair for `PUT`) in Key Vault, unless it writes to Azure Blob and Snowflake reads that through a storage integration. That is a second secret to protect. (#7)

### What this account actually does today (read-only SQL)

| Fact | Evidence |
|---|---|
| Both Graph EAIs use raw `GENERIC_STRING` secrets, not `OAUTH2` secrets or an `API_AUTHENTICATION` integration | `SHOW SECRETS IN SCHEMA INTUNE_DATA.RAW` and `DEV_RAW.SHAREPOINT_HC`; `SHOW SECURITY INTEGRATIONS` lists only SAML, SCIM and a local OAuth app, no `API_AUTHENTICATION` |
| All Graph and CrowdStrike secrets are owned by `ENGINEER_ROLE` | `SHOW GRANTS ON SECRET INTUNE_DATA.RAW.INTUNE_CLIENT_SECRET` returns OWNERSHIP to `ENGINEER_ROLE`, granted by `ENGINEER_ROLE` itself |
| The EAIs are owned by `ACCOUNTADMIN`, with `USAGE` granted to `ENGINEER_ROLE` | `SHOW GRANTS ON INTEGRATION INTUNE_API_ACCESS` and `SHAREPOINT_HC_EAI` |
| `ENGINEER_ROLE` is held by 28 users (humans and Talend, Anaplan and Snowflake service users), plus `SYSADMIN` and `ROBO_ENGINEER_ROLE` | `SHOW GRANTS OF ROLE ENGINEER_ROLE` returns 30 rows |
| The Intune client secret is documented as expiring 2028-10-08 | secret comment |
| The SharePoint secret belongs to an app registration called `Snowflake-Openflow-Sharepoint` (an Openflow-era app, not purpose-built) | secret comment |
| Graph calls from Snowflake already succeed from `aws/eu-west-2` to Azure (`westeurope` for login, `uksouth` for Graph) | `ACCOUNT_USAGE.EXTERNAL_ACCESS_HISTORY` |

Reading: owning a secret plus `USAGE` on the EAI is enough to create a procedure that reads the secret and calls Graph. Anyone in `ENGINEER_ROLE` can therefore use the Intune credential. That is acceptable for Intune read-only data. It is **not** acceptable as the model for a Mailbox-reading credential, where the data is customer Submissions. (Inference from [12] and the grants above; I did not test creating such a procedure because the task was read-only.)

### Secret-free option for B: workload identity federation

- Snowflake can be the OIDC issuer: `CREATE SECRET ... TYPE = WORKLOAD_IDENTITY_FEDERATION`. `DESC SECRET` yields the issuer URL and subject. A procedure obtains short-lived ID tokens with `SYSTEM$ISSUE_WORKLOAD_IDENTITY_FEDERATION_TOKEN(secret, '{"aud": ...}')`. Tokens can only be issued from the primary account. [6][8]
- Entra supports this: add a federated identity credential on the app registration (issuer, subject, audience `api://AzureADTokenExchange`, case-sensitive match), then call the client credentials flow with the external JWT as the client assertion. Maximum 20 federated credentials per app. [7][15]
- Result: no client secret or certificate exists anywhere; nothing to expire or leak.
- **Not verified:** that the token function is callable inside a Python procedure that also has an EAI; that Entra accepts the Snowflake issuer's discovery document; the audience the token must carry; whether the Python handler can make the Entra token call using `login.microsoftonline.com` alone. This is the single most valuable DEV proof for the security case. Snowflake documents the feature for "external services" generally and does not name Entra or Graph. [6]

## 3. Client secret versus certificate

| | Client secret | Certificate | Federated credential |
|---|---|---|---|
| Microsoft guidance | Avoid; if unavoidable, do not set manually or handle by humans [5] | Preferred [5] | Preferred where supported; no stored credential [7] |
| Option A (Function) | Possible | Yes, Key Vault certificate with autorotation [14] | Not needed; managed identity instead |
| Option B, security integration | Yes (`CLIENT_SECRET_BASIC` or `CLIENT_SECRET_POST`) [4] | **Not supported** by the integration [4] | Possible via WIF secret (unproven end to end) [6][7] |
| Option B, custom code | n/a | Possible: PEM private key in a `GENERIC_STRING` secret, build the `PS256` client assertion in Python [16]. The private key is then readable by the handler, so it is the same exposure as a secret. Unverified: `msal` / `cryptography` availability in the Snowflake Anaconda channel | n/a |

Conclusion: on credential type alone Option A is stronger today (managed identity, or a Key Vault certificate). Option B can only match it through federation, which needs proving.

## 4. Egress control

### Option B (Snowflake side)

- An `EGRESS` / `HOST_PORT` network rule lists hosts and ports. An EAI lists allowed network rules and secrets. A handler can only reach hosts in a rule attached to an integration it was created with. Port defaults to 443. [12]
- Control is by **host**, not URL path. A rule for `graph.microsoft.com:443` lets the procedure call any Graph endpoint the token permits. [12]
- Existing Graph rules: `INTUNE_DATA.RAW.MICROSOFT_GRAPH_NETWORK_RULE` (owner `ACCOUNTADMIN`) and `DEV_RAW.SHAREPOINT_HC.MICROSOFT_GRAPH_RULE` (owner `ACCOUNTADMIN`), each with two entries. `CROWDSTRIKE_NETWORK_RULE` is owned by `ENGINEER_ROLE`, so that role can create network rules (needs `CREATE NETWORK RULE` on the schema). [12]
- Who can use an EAI: whoever holds `USAGE` on it and `READ` on the secrets. Today that is `ENGINEER_ROLE` for both Graph EAIs. Seven EAIs exist (`APIS_ACCESS_INTEGRATION`, `APP_BUILD_EAI`, `QVD_COMPARE_PYPI_EAI`, `PYPI_ACCESS`, `SHAREPOINT_HC_EAI`, `INTUNE_API_ACCESS`, `CROWDSTRIKE_API_ACCESS`). `APIS_ACCESS_INTEGRATION` and `PYPI_ACCESS` have no comment; I did not inspect them.
- Private connectivity (Azure Private Link) is possible but needs Business Critical edition and an ACCOUNTADMIN, and costs extra. Graph is a public SaaS endpoint, so this is not a practical control here. [12] Edition of this account was not checked.

### Option A (Azure side)

- Egress control is whatever IT sets on the Function App (VNet integration, NSG, firewall). Ingress is the harder part: change notifications need a **public HTTPS endpoint** that Graph can reach, which is an extra attack surface that Option B does not have. (#7, [S1])
- Key Vault supports firewall, private endpoint, or network security perimeter. [14]

## 5. Which Snowflake role runs the procedure and owns the secrets

Current practice runs everything as `ENGINEER_ROLE`. That cannot be the model for Mailbox ingestion. A suggested split, to be agreed with IT:

| Object | Owner | Why |
|---|---|---|
| Security integration / EAI | `ACCOUNTADMIN` (as today) | Only `ACCOUNTADMIN` has `CREATE INTEGRATION` by default [4] |
| Secret (or WIF secret) | a dedicated role, for example `MAILBOX_INGEST_ADMIN`, **not** `ENGINEER_ROLE` | Ownership implies the ability to grant `READ`; keep it away from 28 users |
| Procedure and task | a dedicated service role, for example `MAILBOX_INGEST_RUNNER`, with `READ` on the secret, `USAGE` on the EAI and write on the Platform Schema landing objects only | Least privilege; the procedure must not run as a human's default role |
| Humans | granted neither `READ` on the secret nor `USAGE` on the EAI | Developers test in DEV with a different app |

Open issue: a task needs `EXECUTE TASK` and the owner role must own the procedure. Whether a procedure owned by the runner role can be called without the humans being able to read the secret is standard Snowflake behaviour for owner's-rights procedures but I did not verify it for secrets. Procedure code is visible to anyone who can `DESC` it, so the Python must contain no credentials (use only `_snowflake` calls). [12]

Role names above are placeholders; none exist today (not checked beyond `ENGINEER_ROLE`).

## 6. Could Entra Conditional Access for workload identities block Snowflake?

- It can block a service principal's token request when it comes **from outside allowed locations** or when Identity Protection raises risk. Block is the only grant action. [10]
- It applies only to **single-tenant service principals owned by the organization** and needs Workload ID Premium licences to create or change a policy. Existing policies keep working without a licence but cannot be edited. Managed identities are not covered. Policies on a group that contains the service principal are not enforced; the service principal must be assigned directly. [10]
- So: **Option A with a managed identity is not subject to it.** **Option B's app registration is.** Failure message would be "Access has been blocked due to Conditional Access policies" in the service principal sign-in logs. [10]
- Observed egress: Snowflake calls to `login.microsoftonline.com` and `graph.microsoft.com` are made from Snowflake's side (the token request comes from Snowflake, not from the Graph IP). The Entra sign-in source is therefore a Snowflake egress address. [S5]
- Snowflake publishes stable egress IP ranges for external access from UDFs and procedures on AWS Commercial. The current output for this account is `153.45.19.0/24` and `153.45.95.0/24`, both expiring **2027-01-27**. Ranges are shared among Snowflake accounts in the region, not unique to Pilgrims, so an IP allowlist narrows trust to "Snowflake in this region", not to "this account". New ranges appear at least 60 days before they take effect and IT must refresh the Entra named location. [9][11]
- Whether this tenant has such a policy is **unknown**. It cannot be read from Snowflake. The existing Intune and SharePoint calls succeed, which shows only that no *blocking* policy applied to *those* apps.
- Report-only mode lets IT test a policy before enforcing it. [10]

## 7. Comparison table

| Criterion | A. Azure Function | B. Snowflake EAI + procedure |
|---|---|---|
| Mailbox scoping | RBAC for Applications | RBAC for Applications (same) |
| Credential to Graph | Managed identity (none stored) or Key Vault certificate | Client secret in Snowflake; certificate only through custom code; federation unproven |
| Who can read the credential | Key Vault RBAC, PIM, audit log | Holders of `READ` on the secret; today `ENGINEER_ROLE` (30 grantees) |
| Credential rotation | Autorotation in Key Vault | Manual; `ALTER` the integration or secret; Intune secret expires 2028-10-08 |
| Egress control | Function App networking, IT-owned | Host-level network rule, `ACCOUNTADMIN`-owned EAI |
| Inbound attack surface | Public HTTPS webhook (for change notifications) | None; polling only |
| Audit trail | Key Vault logs, Entra sign-in logs, Function logs | `EXTERNAL_ACCESS_HISTORY`, `QUERY_HISTORY`, Entra sign-in logs |
| Entra Conditional Access for workload identities | Not applicable to managed identities | Applicable; needs named location with Snowflake egress CIDRs |
| Second credential | Snowflake key pair in Key Vault (for `PUT`), unless Blob + storage integration | None |
| New infrastructure for IT to secure | Subscription, Function App, storage, Key Vault, public endpoint | None beyond the Snowflake objects |
| Existing in-account precedent | Azure Blob stages | Two Graph EAIs (with the weaknesses above) |

Overall: Option A has the better **credential** story; Option B has the smaller **attack surface and infrastructure** story. Option B closes the credential gap only if IT accepts either a long-lived client secret held in a locked-down Snowflake role, or the federation route once proven.

## 8. Blockers IT must clear

Each item is a decision or action IT owns. Items marked (B) apply to the Snowflake design only.

1. **Entra app registration and admin consent** for a **dedicated** app (do not reuse the Intune or Openflow-SharePoint apps), with `Mail.Read` application permission. (Both)
2. **RBAC for Applications assignment** scoped to the Data Product Mailboxes by management scope or administrative unit, and **removal of any tenant-wide `Mail.Read` consent**. Confirm with `Test-ServicePrincipalAuthorization` against one in-scope and one out-of-scope mailbox. Allow 30 minutes to 2 hours for propagation. (Both)
3. **Credential decision (B):** accept a client secret (with a rotation date and owner), or approve the federated identity credential route, or approve custom certificate code. Certificates cannot go through the security integration.
4. **Conditional Access (B):** tell us whether any Conditional Access policy for workload identities exists or is planned. If so, add the Snowflake egress ranges (currently `153.45.19.0/24` and `153.45.95.0/24`) to a named location and set up refresh before **2027-01-27**, or exempt the app. Test in report-only mode first. Confirm Workload ID Premium licensing if a new policy is needed.
5. **Snowflake role model (B):** agree dedicated owner and runner roles (section 5) and agree that `ENGINEER_ROLE` gets neither `READ` on the secret nor `USAGE` on the new EAI. This is a Snowflake admin and IT security decision. The current practice of putting secrets under `ENGINEER_ROLE` is the thing to avoid.
6. **EAI and network rule creation (B):** an `ACCOUNTADMIN` creates the EAI and security integration. The network rule only needs `graph.microsoft.com:443` and `login.microsoftonline.com:443`. Confirm IT accepts host-level rather than path-level egress control.
7. **Azure approval (A):** an Azure subscription, Function App, public webhook endpoint, storage, Key Vault, monitoring, and network review. Azure is not on the approved-tools list (#7).
8. **Retention and review of audit evidence:** who reviews `EXTERNAL_ACCESS_HISTORY` (B) or Key Vault and sign-in logs (A), and for how long. `ACCOUNT_USAGE` views keep history for a limited period; retention was not confirmed.
9. **Data handling:** the Mailboxes carry Submission content that may include personal data. Whether Snowflake internal stages in `eu-west-2` satisfy any data-residency rule is a question for IT and legal; this research did not examine it.

## Not verified

- Conditional Access policies in this tenant; whether Workload ID Premium is licensed.
- Whether RBAC for Applications works with a managed identity's service principal.
- The workload identity federation flow from a Python procedure to Entra (token function in a handler, audience, discovery document, Entra acceptance of the Snowflake issuer). Needs a DEV proof.
- Whether `msal` or `cryptography` can build the certificate assertion in a Snowflake Python procedure.
- That a non-owner role cannot read secret values through the owner's-rights procedure path; Snowflake documents `READ` as the gate, I did not test it.
- The contents and grants of `APIS_ACCESS_INTEGRATION` and `PYPI_ACCESS`; the Snowflake account edition; `ACCOUNT_USAGE` retention for this account.
- The Entra app permissions behind `INTUNE_API_ACCESS` and the SharePoint app (also not verified in #7).
- Azure Function pricing and Key Vault pricing (out of scope here).

## Sources

Microsoft:

1. Microsoft Learn, *Role Based Access Control for Applications in Exchange Online*. https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac
5. Microsoft Learn, *Microsoft identity platform best practices and recommendations* (use certificates, not secrets; managed identities or Key Vault). https://learn.microsoft.com/en-us/entra/identity-platform/identity-platform-integration-checklist
7. Microsoft Learn, *Workload identity federation concepts*. https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation
10. Microsoft Learn, *Conditional Access for workload identities* (managed identities not covered). https://learn.microsoft.com/en-us/entra/identity/conditional-access/workload-identity
14. Microsoft Learn, *Secure your Azure Key Vault*. https://learn.microsoft.com/en-us/azure/key-vault/general/security-features
15. Microsoft Graph, *Overview of federated identity credentials*. https://learn.microsoft.com/en-us/graph/api/resources/federatedidentitycredentials-overview
16. Microsoft Learn, *Application authentication certificate credentials* (client assertion format, `PS256`). https://learn.microsoft.com/en-us/entra/identity-platform/certificate-credentials

Snowflake:

4. *CREATE SECURITY INTEGRATION (External API Authentication)*. https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-api-auth
6. *Workload identity federation for Snowflake workloads that access external services*. https://docs.snowflake.com/en/user-guide/workload-identity-federation-outbound
8. *SYSTEM$ISSUE_WORKLOAD_IDENTITY_FEDERATION_TOKEN*. https://docs.snowflake.com/en/sql-reference/functions/system_issue_workload_identity_federation_token
9. *Securing ingress of Snowflake requests with egress IP addresses*. https://docs.snowflake.com/en/user-guide/egress-ip/network-egress
11. *SYSTEM$GET_SNOWFLAKE_EGRESS_IP_RANGES*. https://docs.snowflake.com/en/sql-reference/functions/system_get_snowflake_egress_ip_ranges
12. *Creating and using an external access integration*. https://docs.snowflake.com/en/developer-guide/external-network-access/creating-using-external-network-access
13. *CREATE SECRET*. https://docs.snowflake.com/en/sql-reference/sql/create-secret

In-repo and account evidence (cited as [S1], [S3], [S5] above):

- [S1] `docs/research/mailbox-ingestion-options.md` (#7).
- [S3] Read-only SQL on 2026-10-29: `SHOW EXTERNAL ACCESS INTEGRATIONS`, `DESC EXTERNAL ACCESS INTEGRATION INTUNE_API_ACCESS` / `SHAREPOINT_HC_EAI`, `SHOW GRANTS ON INTEGRATION ...`, `SHOW SECRETS IN SCHEMA INTUNE_DATA.RAW` / `DEV_RAW.SHAREPOINT_HC`, `SHOW NETWORK RULES IN SCHEMA ...`, `SHOW SECURITY INTEGRATIONS`.
- [S5] Read-only SQL on 2026-10-29: `SHOW GRANTS OF ROLE ENGINEER_ROLE`, `SHOW GRANTS ON SECRET INTUNE_DATA.RAW.INTUNE_CLIENT_SECRET`, `SELECT ... FROM SNOWFLAKE.ACCOUNT_USAGE.EXTERNAL_ACCESS_HISTORY LIMIT 5`, `SYSTEM$GET_SNOWFLAKE_EGRESS_IP_RANGES()`.
