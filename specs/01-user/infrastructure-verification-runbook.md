# 01 — User & Authentication: Infrastructure Verification Runbook

**Status:** DRAFT runbook. **No check in it has been performed, and no result is claimed.** It supports security-design §20 and physical-design §10. Running it is an action for the project owner or a delegate with Render and Supabase access; the assistant cannot run it.

All checks below are **read-only**. None creates a table, role, schema or policy, and none changes any setting. Exceptions are marked.

## 1. Safety rules (read first)
1. **Never paste a connection string, password, key or token** into the chat, a ticket or any file in this repository. Send back only the results listed in §4.
2. Put each credential in an **environment variable** in your own terminal (or Render's shell) and never type it inline in a command. The variable names below are placeholders: `DIRECT_URL`, `POOLER_URL`, `OWNER_URL`, `APP_URL`. Do not save them in shell history files you share.
3. Use the **owner** credential only from your own trusted machine, only for the catalog checks (4 to 9). **Never put the owner credential on Render.** The Render runtime will hold only the application role.
4. A password that was pasted into a conversation is **exposed**. Rotate it (RISK-002, RISK-004) before relying on any check result as a production sign-off.
5. Run against a **test or non-production** database if one exists. If only one database exists, run only the read-only queries here, and run nothing outside this document.
6. If any step fails or surprises you, stop and report it. Do not "fix" it from this runbook.

**Tools:** the PostgreSQL command-line client `psql` on your machine, and a browser for the Supabase dashboard. Nothing is added to the project.

## 2. Checks that can run now
Numbers match security-design §20.

### Check 1 — Connectivity from Render to the database (direct and session pooler)
**Where:** Render's shell for the backend service, or a one-off job. If Render offers no shell on your plan, record "no shell available" and use a temporary test service only with the owner's approval; do not change the production service.
**Steps:**
1. Run a trivial query through the **direct** route: `psql "$DIRECT_URL" -c "select 1"`.
2. Run it through the **session pooler**: `psql "$POOLER_URL" -c "select 1"`. (Supabase's session mode uses the same port as direct on a different host; use the host and port shown in the project's connection settings. Do not use the transaction-mode port.)
3. Note, for each: success or failure, the error text with any credential removed, and whether the error mentions IPv6 or "network unreachable".
**Pass:** at least one route works from Render. **Record:** which routes work. The result decides the connection method (direct preferred, otherwise session pooler; `USER-SD-DEC-026`).

### Check 2 — TLS and certificate verification
**Steps:**
1. Download the CA certificate that Supabase documents for the project (from the dashboard's database settings).
2. Connect with full verification: add `sslmode=verify-full` and `sslrootcert=<path to that certificate>` to the connection parameters for the chosen route, and run `select 1`.
3. Repeat with a deliberately wrong certificate file to confirm the connection is **refused**.
**Pass:** step 2 succeeds and step 3 fails. **Record:** both outcomes, and whether the driver the application will use can be configured the same way (verify when the driver is installed).

### Check 3 — Behavior when a connection is lost mid-transaction
**Limit of this check:** it shows how the client sees a dropped connection. It **cannot** reproduce a lost commit acknowledgment. That is verified by an application-level test after implementation.
**Steps (own machine, owner or app credential, each route):**
1. Open `psql`, run `begin;` then `select pg_terminate_backend(pg_backend_pid());`.
2. Record the error the client prints and whether the session is closed.
3. Reconnect and confirm nothing from that session persists. (The test creates nothing.)
**Pass:** the client reports an error and no state remains. **Record:** the exact error class (no credentials).

### Check 4 — Roles: owner and application are separate and the application role is least-privileged
**Prerequisite:** the application role may not exist yet. If it does not, record "application role not created" and do **not** create it from this runbook.
**Run as the owner** (read-only catalog queries):
- List role attributes: query `pg_roles` for the columns `rolname`, `rolsuper`, `rolcreaterole`, `rolcreatedb`, `rolcanlogin`, `rolbypassrls`, for the roles you believe are the owner, the application role, and any role the host's HTTP APIs use.
- List memberships: query `pg_auth_members` joined to `pg_roles`, to see which roles the application role belongs to.
**Pass (design, physical-design §6):** the application role is not a superuser, cannot create roles or databases, does not bypass row-level security, is not a member of the owner role or of any API-facing role, and the owner and application are different roles.
**Record:** the attribute table with role names replaced by `owner`, `app`, `api-role-n`.

### Check 5 — Least-privilege grants *(only after the tables exist in a test database)*
**Run as the owner:** query `information_schema.table_privileges` and `information_schema.column_privileges` for the authentication schema, filtered to the application role and to the API-facing roles.
**Pass:** the application role's privileges match physical-design §6 exactly (for example, no `UPDATE` or `DELETE` on the events table; column-restricted updates on the others), and the API-facing roles have **none**.
**Record:** the grant listing compared with §6, with differences marked.

### Check 6 — Exposure through the host's HTTP APIs
**Steps:**
1. In the Supabase dashboard, open the API settings and note the list of **exposed schemas**. Confirm the authentication schema is **not** in it.
2. *(After the tables exist.)* For each API-facing role (the publishable/anonymous role and the authenticated role), try to read each authentication table through the project's REST address using that role's **publishable** key (never the secret key). Take the key from an environment variable.
**Pass:** the schema is not exposed, and every attempt is refused or returns nothing. **Record:** exposed-schema list, and the status code for each attempt (no body, no keys). Also confirm the Data API is unused by the application (`USER-SD-AC-043`).

### Check 7 — Row-level security state
**Run as the owner (after the tables exist):** query `pg_class` joined to `pg_namespace` for `relrowsecurity`, `relforcerowsecurity` and the table owner of each authentication table, and query `pg_policies` for the schema.
**Pass:** row-level security is enabled on every table; the application role has an explicit policy and does not depend on owner bypass; API-facing roles have no policy; **grants are still the main control**.
**Record:** one row per table.

### Check 8 — Indirect exposure: Realtime, functions, views
**Run as the owner:**
- Query `pg_publication_tables` and confirm **none** of the authentication tables appears.
- List functions in schemas that are exposed to the HTTP APIs (`pg_proc` joined to `pg_namespace`) and confirm none reads the authentication tables or runs with elevated rights over them.
- List views and materialized views (`pg_views`, `pg_matviews`) in exposed schemas and confirm none selects from the authentication tables.
**Pass:** all three lists show no access path. **Record:** counts and names of anything found.

### Check 9 — Migration and ownership model
**Written statement, no queries:** state (a) who runs migrations, (b) with which role, (c) from where, and (d) confirmation that the owner credential is **not** stored on Render. **Pass:** all four answered and (d) is yes.

### Check 15 — UUID generation on this PostgreSQL version
**Run as any role (read-only):**
1. `select version();`
2. `select gen_random_uuid();` run a few times. Confirm the function exists and the value's third group starts with `4` (version 4).
**Pass:** the function works and returns version-4 values. **Record:** the server version string and the pass or fail. If the function is missing, report it: this design assumes it and the physical design would change.

## 3. Checks that wait
| Check | Why it waits |
|---|---|
| 10 Browser, cookie, CSRF and Origin behavior | Needs the frontend and API running on the intended origins |
| 11 Trusted-proxy behavior | Needs a running API behind Render's proxy |
| 12 Rate-limit enforcement | After implementation |
| 13 Measured Google token and request sizes | Needs the real sign-in flow |
| 14 Secret storage on the host and startup validation | After implementation |
| 16 No client IPs in logs | After implementation |
| 17 Independent detection of event-write failure | After implementation |
| Ambiguous commit outcome | Application-level test after implementation |

## 4. What to send back
For each check: **number, pass or fail or not possible, date, who ran it, and the requested "Record" item.** Examples of what to send: "Check 2: verified connection succeeded, wrong certificate refused" or the role-attribute table with role names replaced as described.

**Never send:** any connection string, password, API key, token, certificate private key, or the contents of any environment file.

## 5. After the results
- The assistant updates security-design §20 statuses **only to what the evidence shows**, and keeps anything not evidenced as NOT PERFORMED or UNVERIFIED.
- Results decide the connection method (check 1 to 3), confirm or change the privilege design (4 to 8), and confirm the UUID assumption (15).
- No check result closes RISK-001 to RISK-004 or satisfies a production gate by itself; those need their own remediation evidence.
