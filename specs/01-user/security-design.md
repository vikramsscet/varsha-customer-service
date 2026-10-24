# 01 — User & Authentication: Security Design

**Status:** Draft — pending project owner review

This document addresses the security design questions assigned to the security review by the approved requirements, API contracts, and data model (notably USER-Q-001, USER-Q-002, USER-Q-003, USER-Q-004, and USER-DM-DEC-004 to 013). It records the security decisions **approved by the project owner in Step 5** (marked **Approved** in §3) and **recommendations for review** for everything else. A recommendation is not approved unless §3 says so, and no open decision is resolved by this document.

It contains no application code, SQL, migrations, schema files, or configuration. Mentions of Node.js capabilities or libraries are design considerations, not selections.

**Labels used for every significant item**
- **[Required]** — mandated by an approved specification (requirements, approved API contract, or approved data model).
- **[Proposed]** — a recommendation for review; not approved.
- **[Unresolved]** — a decision not yet made; recorded, not decided.
- **[Approved]** — approved by the project owner for this design (see §3). Approval of a decision does not approve any other decision, and does not finalize the physical database schema.

Keywords follow the requirements: MUST / SHOULD / MAY.

---

## 1. Scope and Source Specifications

**In scope:** security design for Google identity validation, access tokens, refresh-token strategy and storage, rotation and reuse detection, concurrency safety, session revocation and logout, token transport, cookie/CORS/CSRF protection, rate limiting and abuse protection, secret management, and security-event handling and retention dependencies.

**Out of scope:** Cart, Checkout, Order, Payment, Delivery, Feedback; product browsing; administrative identities; other authentication providers; account deletion/recovery (USER-DEC-009, 011, 012); physical schema and DDL.

| Source | Status | Used for |
|---|---|---|
| [requirements.md](requirements.md) | Approved (business decisions USER-DEC-001–014 approved) | USER-FR, USER-SEC, USER-AC, USER-Q |
| [api-contracts.md](api-contracts.md) | Approved for implementation planning (owner statement); decisions in its §9 and §10 and §11 carry their own statuses | Endpoints, errors, nonce, CSRF and cookie contract |
| [data-model.md](data-model.md) | Approved for implementation planning; decisions in its §11, §13 and §14 carry their own statuses | Entities, constraints, retention dependencies |
| [frontend-contract.md](frontend-contract.md) | Draft — pending project owner review (created Step 5b) | Browser behavior |

## 2. Project Constraints

**[Required]** Node.js, Express, JavaScript (no TypeScript), PostgreSQL, REST, Google as the only authentication provider in the first release, JWT access tokens plus refresh tokens, configuration through environment variables, no Docker, no unit tests at this stage. No additional technologies or dependencies without approval (api-contracts and requirements constraints).

**Deployment context confirmed so far (Step 5).** Details and what is still missing are in §10.2.

| # | Fact | Status |
|---|---|---|
| F1 | Frontend technology React, hosted on Render | Confirmed |
| F2 | Frontend origin `https://varsha-homemade.onrender.com`; public API origin `https://varsha-customer-service.onrender.com` | **Both confirmed.** No other domains are assumed. |
| F3 | The two origins are different, both HTTPS, both subdomains of `onrender.com` | Different origins confirmed. **Site relationship evaluated in §10.2: cross-site**, because `onrender.com` is on the Public Suffix List. |
| F4 | Gateway and proxy topology | **Confirmed:** no Apigee or other gateway is in the request path; the API is directly reachable at its public URL; no separate BFF exists. Render's own platform proxy still sits in front of the service, so its trusted-proxy configuration is still to be established. |
| F5 | Single instance planned | Confirmed as the current plan |
| F6 | Browser clients only for the first release; no native mobile | Confirmed |
| F7 | Custom domains, for example frontend `https://www.varsha-homemade.com` and API `https://api.varsha-homemade.com` | **Examples only.** Not configured. Ownership, DNS, and Render support are **not confirmed**. |
| F8 | PostgreSQL hosting | **Confirmed:** hosted on Supabase. Project identifiers, keys and connection strings are not recorded in the specifications. See §2.1. |

**BFF.** Confirmed: no separate BFF exists. `varsha-customer-service` is the backend API that the frontend calls directly.

This document does not assume anything marked unconfirmed.

### 2.1 Database hosting and access: Supabase (F8; USER-SD-DEC-025, 026)
**Approved (Step 5b, 2026-10-24):**
- Supabase hosts **PostgreSQL only**. Express handles customer authentication and issues and verifies its own access JWTs. **Supabase Auth is not used** for customer authentication in the first release. The application does **not** depend on the Supabase Data API unless the owner explicitly approves a change.
- Express uses a **standard PostgreSQL driver** (such as `pg`), **not** the Supabase client library. This is a driver *choice*; the individual package approval is still pending (§14.1).

**Inspection result (the repository is the source of truth).** The repository holds no configuration files. No requirements, API-contract or data-model text assumes Supabase Auth, Supabase-issued tokens or Data API use. The earlier open questions about Supabase Auth and a Supabase JWKS URL are resolved by this decision: the JWKS URL is **not used**. A Supabase URL alone was never treated as evidence of intent.

**Database access controls (approved in principle; the schema, role and privilege design is completed before implementation).**
1. **Database permissions come first. RLS is not a replacement for them.** The Express login role holds **only the privileges the application needs** (for example specific data-manipulation rights on specific tables, and no schema-change rights, superuser rights or row-security bypass). The **database owner** (which owns the schema and tables and runs approved schema changes) and the **application login** are **separate roles**. Privileges are decided and documented at physical design; no names or credentials are invented here.
2. The authentication tables live in a schema that is **not exposed** by Supabase's HTTP API settings. If the Data API is unused, consider disabling it. Nothing is granted on these tables to the roles that the HTTP APIs use.
3. **Row-level security is enabled as defense in depth, not as the main control.** Its implications must be designed: a table owner bypasses it unless it is forced, and a role with no policy sees nothing, so the application role needs a deliberate policy or ownership arrangement.
4. Verify that none of these tables is in a Realtime publication and that no exposed function or view reads them.
5. Restrict who can use the Supabase dashboard and SQL editor, with strong sign-in.
6. The Supabase publishable and secret keys and the JWKS URL are **not needed** by the application. They are never placed in React, source control, logs or specifications.

**Limits.** A non-exposed schema, revoked grants and RLS do not protect against a leaked application database password, dashboard access, or a flaw in code that holds the application role. A compromised application role can read whatever it can reach.

**Connection method (approved as a plan: test first; the result decides).** Compared against Supabase's connection documentation (checked 2026-10-24):

| Mode | Network | Behavior | Notes |
|---|---|---|---|
| Direct (port 5432) | IPv6, or IPv4 with a paid add-on | Full session behavior, prepared statements; recommended for persistent backends | **Preferred if Render can reach it** |
| Pooler, session mode (port 5432) | IPv4 | Prepared statements supported; the alternative for persistent backends on IPv4-only networks | **Used if direct is not reachable** |
| Pooler, transaction mode (port 6543) | IPv4 | No prepared statements; session state does not persist; recommended for serverless | Not needed for one Render instance |

- **Test first:** Render-to-Supabase connectivity is tested before the connection method is finalized. **No test has been run;** Render's outbound IPv6 support is **unverified**. Running the test is an action for you or an authorized step; this document changes no configuration.
- **Atomic rotation is unaffected in every mode:** consuming the old refresh token and creating its successor is one explicit transaction on a single connection. A pooler adds a hop where a commit can succeed but its acknowledgment is lost, which the approved ambiguous-outcome rule (§8.1) already handles. Transaction mode would forbid session-level features and named prepared statements.
- **TLS** is required for the database connection, with certificate verification recommended (to confirm in physical design).
- **Database role setup:** a dedicated least-privilege Express role is created through the approved database setup process. **No role name, password or connection string is invented or recorded.** Whether such a role already exists is unconfirmed.

## 3. Decision Register

**Status vocabulary:** *Approved* · *Approved (conditional)* — approved by the owner, in principle or dependent on a stated condition · *Approved (delegated)* — decided by the assistant under the owner's Step 5d delegation (2026-10-25), reversible by the owner · *Open* · *Blocked by Deployment Details* (none currently).

The project owner has approved **21 decisions outright and 6 conditionally or in principle**; **2 remain open** (017 rate limiting and 020 retention). **Approved:** 003 to 012, 015, 016, 018 (approach), 019 (principle), 021 (process), 022, 023 (single-tab), 024 (policy), 025, 028, 029. **Approved (conditional):** 001 (ID-token flow with a server-issued nonce, in principle), 002 (HS256 if Express is the only issuer and verifier), 013 and 014 (deployment profiles, no production profile selected), 026 (connection plan), 027 (page-held binder, in principle). Where only part of a decision was approved, the approved part is stated in **Approved choice** and the rest is listed as still open.

An approved decision does not approve any other decision and does not finalize the physical database schema. Each decision shows an **Approval** field: the owner approval where given, otherwise "Not approved."

Decision IDs USER-SD-DEC-001 to 024 are unchanged. Decisions 025 to 029 were added in Step 5b (Supabase role, connection and role, nonce binder, CSRF token, Profile X fallback).

### Summary

| ID | Topic | Status | Still open or conditional |
|---|---|---|---|
| USER-SD-DEC-001 | Google authentication flow and replay protection | Approved (conditional) | Approved in principle, subject to documenting the complete flow (§4.4, now documented for your final review) and your approval of the new endpoint and nonce record as written in api-contracts §11 and data-model §14. |
| USER-SD-DEC-002 | Access-token signing algorithm and key type | Approved (conditional) | Condition: if another service needs to verify application JWTs independently, the architectural implications are presented to you before the algorithm changes. No gateway verifies the application's JWTs. |
| USER-SD-DEC-003 | Access-token claims and clock skew | Approved | — |
| USER-SD-DEC-004 | Strict single-use refresh-token rotation | Approved | — |
| USER-SD-DEC-005 | Refresh-token format and generation | Approved | — |
| USER-SD-DEC-006 | Refresh-token storage representation (digest strategy) | Approved | — |
| USER-SD-DEC-007 | Concurrency mechanism and transaction-outcome handling | Approved | — |
| USER-SD-DEC-008 | Effect of presenting a consumed token (reject only vs revoke session) | Approved | — |
| USER-SD-DEC-009 | Acceptance of logout from lost responses and races | Approved | — |
| USER-SD-DEC-010 | Maximum absolute session lifetime | Approved | — |
| USER-SD-DEC-011 | Multiple concurrent sessions; logout from all devices | Approved | — |
| USER-SD-DEC-012 | Logout semantics | Approved | — |
| USER-SD-DEC-013 | Refresh-token transport and client storage | Approved (conditional) | Condition: feasibility is confirmed for the profile to be used. Domain ownership is unconfirmed, and proxy feasibility is only assessed (a feasibility assessment, not a deployment change, is authorized). Profiles are documented in §10.3. |
| USER-SD-DEC-014 | Cookie attributes | Approved (conditional) | Conditional on the profile chosen and on browser behavior supporting it (§10.3, §10.6). |
| USER-SD-DEC-015 | CORS policy and allow-list | Approved | Revisit the allow-list if custom domains are configured (§10.4). |
| USER-SD-DEC-016 | CSRF protection | Approved | The exact endpoint list and how the frontend obtains and sends the token are documented in §10.5. The status and code are recorded in api-contracts §11. |
| USER-SD-DEC-017 | Rate-limiting scope, thresholds, and storage | Open | Thresholds are deferred pending operational review. Proposed limit targets, storage, client-IP identification, trusted-proxy assumptions, and store-failure behavior are in §11.1. Forwarded client-IP headers are not trusted until a trusted-proxy configuration is explicitly established. Reassess if the service runs multiple instances. |
| USER-SD-DEC-018 | Secret management, key rotation, and compromise response | Approved | Still open: the named accountable owner for secret management and emergency rotation, and confirmation of how Render exposes secrets. The exposed Google client secret is kept for now by owner decision and recorded as an unresolved security risk (§12, RISK-001). |
| USER-SD-DEC-019 | Security-event types and storage | Approved | Exact event definitions, retention periods, and operational details remain Open. The nonce-replay event is proposed in addition (§4.4). |
| USER-SD-DEC-020 | Retention of events, sessions, and consumed-token records | Open | Retention periods are not chosen. Needs privacy, legal, and operational review (§13.3). |
| USER-SD-DEC-021 | New dependencies | Approved | Each individual dependency remains Open until approved; candidates are in §14.1. The driver choice (a standard PostgreSQL driver, not the Supabase client library) is approved in principle under decision 026. |
| USER-SD-DEC-022 | Immediate access-token revocation mechanism | Approved | — |
| USER-SD-DEC-023 | Client-side refresh coordination requirements | Approved | Cross-tab coordination is recommended (frontend-contract.md) and remains Open pending your approval. |
| USER-SD-DEC-024 | Event-persistence failure policy and monitoring | Approved | The monitoring mechanism remains Open pending review of operational tooling. |
| USER-SD-DEC-025 | Supabase role and HTTP API restriction (R1, R2) | Approved | Schema, roles, privileges and RLS design are completed at physical design. RLS is not a replacement for database permissions (§2.1). |
| USER-SD-DEC-026 | Database connection method, driver, and application role | Approved (conditional) | Condition: the connectivity test result, and confirmation of the connection method and role setup before the physical design is finalized. No connection string, role or credential is invented. The `pg` package approval remains pending (§14.1). |
| USER-SD-DEC-027 | Browser binding of the sign-in nonce (page-held binder) | Approved (conditional) | Condition: the limitations listed in §4.4 are validated before this decision is final. A nonce alone does not prove browser identity, and the binder does not either. |
| USER-SD-DEC-028 | Session-scoped CSRF token | Approved | Details in §10.5. The session record gains a CSRF-token digest (data-model §14). |
| USER-SD-DEC-029 | Profile X third-party-cookie fallback (re-sign-in design) | Approved | The exact user-interface behavior is for the frontend review; the structure is in §10.6 and frontend-contract.md. |

### Decision details

Each entry gives: Recommendation · Rationale · Trade-offs · Information required from the owner · Dependencies · Impact if unresolved · Approval.

**USER-SD-DEC-001 — Google authentication flow and replay protection** *(API-DEC-001; USER-SEC-001; USER-FR-002)* — **Approved (conditional)**
- *Approved choice:* Google sign-in uses **ID-token verification protected by a server-issued, single-use nonce**, with a **five-minute nonce lifetime**. The complete flow is documented in §4.4.
- *Condition:* Approved in principle, subject to documenting the complete flow (§4.4, now documented for your final review) and your approval of the new endpoint and nonce record as written in api-contracts §11 and data-model §14.
- *Recommendation:* ID-token flow with a server-issued, single-use nonce, five-minute lifetime, page-held binder (decision 027); complete flow in §4.4.
- *Rationale:* The ID-token flow is the recommended proposal in the API contract and adds no endpoint. Signature and standard-claim validation do not by themselves prove the credential belongs to the current attempt (§4.1).
- *Trade-offs:* Binding via a server-issued nonce is strongest but needs a new endpoint and short-lived state; a client-generated nonce adds little replay protection; the authorization-code flow adds a callback endpoint and a client secret.
- *Information required:* the frontend sign-in flow; whether new API surface or short-lived server state is acceptable; whether the residual risk of no binding is acceptable.
- *Dependencies:* API-DEC-001; frontend and deployment facts (§10.2); data model (any nonce state is not currently modeled).
- *Impact if unresolved:* the sign-in request schema and replay posture cannot be finalized; endpoint 1 cannot be implemented.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (in principle)

**USER-SD-DEC-002 — Access-token signing algorithm and key type** *(USER-FR-026, 072; USER-SEC-002)* — **Approved (conditional)**
- *Approved choice:* Sign application access JWTs with **HS256 provided Express is the only application JWT issuer and verifier** in the first release. Controls in §5.1.
- *Condition:* Condition: if another service needs to verify application JWTs independently, the architectural implications are presented to you before the algorithm changes. No gateway verifies the application's JWTs.
- *Recommendation:* HS256 with the controls in §5.1, conditional on Express being the only issuer and verifier.
- *Rationale:* A single service holding one secret is the simplest sound design; any party that can verify an HMAC token can also forge one.
- *Trade-offs:* Simplicity versus a wider blast radius if the secret must be shared; asymmetric adds key-management cost.
- *Information required:* service architecture; whether gateways or other services will verify tokens; where secrets may be held.
- *Dependencies:* USER-SD-DEC-018; deployment topology.
- *Impact if unresolved:* token issuing and validation cannot be built.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (conditional)

**USER-SD-DEC-003 — Access-token claims and clock skew** *(USER-FR-023, 053; USER-DEC-010)* — **Approved**
- *Approved choice:* Claims: issuer, audience, subject, issued-at, expiry. A small, bounded, configurable clock-skew allowance that never bypasses expiry.
- *Recommendation:* issuer, audience, subject (internal customer identifier), issued-at, expiry; no personal data. Any clock-skew tolerance must be small, bounded, explicitly configured, and must never bypass expiration (§5).
- *Rationale:* Data minimization; avoids a configuration default that quietly extends token life.
- *Trade-offs:* Zero skew can reject valid tokens on drifting clocks; larger skew extends exposure.
- *Information required:* owner confirmation of the claim set; the skew value (if any) is an operational input.
- *Dependencies:* DEC-002.
- *Impact if unresolved:* token format and validation rules cannot be finalized.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-004 — Strict single-use refresh-token rotation** *(USER-FR-080–083; USER-Q-001; USER-DM-DEC-013)* — **Approved**
- *Approved choice:* Strict single-use refresh-token rotation. Each successful refresh consumes the existing token and issues a successor. No grace period unless the requirements are formally revised and approved.
- *Recommendation:* rotating, single-use refresh tokens (Option A, §6.2). Refresh tokens are single-use **only if rotation is approved**.
- *Rationale:* The only option that meets USER-FR-081 and 083 without changing requirements.
- *Trade-offs:* Legitimate clients can occasionally present a consumed token (lost responses, tabs, ambiguous outcomes) and be signed out (§6.4).
- *Information required:* owner approval of strict single-use rotation.
- *Dependencies:* DEC-007, 008, 009, 023; data model USER-DM-DEC-013.
- *Impact if unresolved:* the data model's physical design and refresh endpoint remain blocked.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-005 — Refresh-token format and generation** *(USER-FR-075; USER-DM-DEC-004)* — **Approved**
- *Approved choice:* Opaque, cryptographically random refresh tokens. Not JWTs.
- *Recommendation:* opaque random value from a cryptographically secure random source with entropy that makes guessing infeasible; not a JWT.
- *Rationale:* Carries no information if disclosed; meets USER-FR-075.
- *Trade-offs:* Requires server-side state per refresh.
- *Information required:* owner approval.
- *Dependencies:* DEC-006.
- *Impact if unresolved:* token issuance and storage cannot be designed.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-006 — Refresh-token storage representation (digest strategy)** *(USER-SEC-011; USER-AC-029; USER-DM-DEC-004)* — **Approved**
- *Approved choice:* Store a one-way digest, never the raw value. Begin with an unkeyed digest provided the final design meets the approved security requirements. Reassess a keyed digest if the threat model requires protection against database write access. Never log or persist raw refresh tokens.
- *Recommendation:* the baseline satisfying USER-SEC-011 is a one-way digest of a high-entropy token. Unkeyed versus keyed digest remains open and is compared in §7; a keyed digest is recommended if the owner's threat model includes database write access.
- *Rationale:* See §7 comparison.
- *Trade-offs:* The keyed option adds a secret lifecycle and likely a key identifier stored alongside digests.
- *Information required:* threat model for database compromise (read-only versus write); capacity to manage another secret.
- *Dependencies:* DEC-005, DEC-018; data model verifier definition.
- *Impact if unresolved:* the physical token store cannot be finalized.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-007 — Concurrency mechanism and transaction-outcome handling** *(USER-FR-083; API-DEC-007; USER-DM-AC-013, 024)* — **Approved**
- *Approved choice:* Atomic conditional update or an equivalently safe transaction strategy. Consume the existing token and create its successor in the same transaction. Concurrent requests cannot consume the same token twice. An ambiguous transaction outcome is never treated as a confirmed rollback. A losing request gets the generic refresh error without revealing token state.
- *Recommendation:* atomic conditional state transition; consuming the old token and issuing the successor in one transaction; ambiguous outcomes never treated as confirmed rollback (§8.1).
- *Rationale:* One winner by construction; avoids claiming a token is still valid when its state is unknown.
- *Trade-offs:* Ambiguous outcomes can sign a legitimate user out.
- *Information required:* owner approval; the losing request's response (proposed: generic refresh failure).
- *Dependencies:* DEC-004.
- *Impact if unresolved:* refresh endpoint cannot be implemented safely.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-008 — Effect of presenting a consumed token** *(USER-FR-082; USER-AC-026; USER-DM-AC-026)* — **Approved**
- *Approved choice:* Reject a request presenting a consumed token, record the security event, and revoke the entire affected session. Legitimate concurrent requests or lost responses may also cause sign-out.
- *Recommendation:* detection (reject and record) is required. Whether to **also** revoke the entire session is a separate policy; recommendation is to revoke it, with the benign-collision cost accepted under DEC-009.
- *Rationale:* Reuse can indicate token theft; revoking cuts off every holder.
- *Trade-offs:* A legitimate lost-response or tab race is indistinguishable from reuse and signs the user out.
- *Information required:* owner choice between reject-only and reject-and-revoke.
- *Dependencies:* DEC-004, 009, 023.
- *Impact if unresolved:* USER-DM-AC-026 and the reuse path remain undefined.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-009 — Acceptance of logout from lost responses and races** *(USER-FR-081; api-contracts §6.2)* — **Approved**
- *Approved choice:* Accept occasional sign-outs caused by lost responses and concurrent refresh requests. No grace period and no automatic retry of a failed refresh.
- *Recommendation:* accept the logout as a trade-off for strict single-use; no grace period.
- *Rationale:* A grace period would contradict USER-FR-081 and requires formally revised, approved requirements.
- *Trade-offs:* Occasional unexpected sign-outs versus strict consumed-token rejection.
- *Information required:* whether the owner accepts this user-experience cost.
- *Dependencies:* DEC-004, 008, 023.
- *Impact if unresolved:* rotation cannot be approved on an informed basis.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-010 — Maximum absolute session lifetime** *(USER-DM-DEC-005; USER-DM-AC-027)* — **Approved**
- *Approved choice:* Absolute session lifetime is **90 days**. A session must not exceed 90 days even when actively refreshed. Refreshing never extends the absolute session expiry. A token's own expiry (7 days by default) is capped at the session's absolute expiry (§8.3).
- *Recommendation:* an absolute, configurable cap on total session life. **The owner must select and approve the value before implementation**; none is proposed.
- *Rationale:* With rotation, active sessions would otherwise never expire (§8.3).
- *Trade-offs:* Periodic forced re-authentication versus bounded exposure of a kept-alive stolen session.
- *Information required:* the maximum absolute lifetime; whether a separate inactivity timeout is wanted beyond the per-token lifetime.
- *Dependencies:* DEC-004; USER-DEC-007 (7-day default per-token lifetime).
- *Impact if unresolved:* session expiry rules and token issuance cannot be finalized.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-011 — Multiple concurrent sessions; logout from all devices** *(USER-Q-003; USER-DM-AC-008)* — **Approved**
- *Approved choice:* Multiple active sessions per customer are allowed. Logout ends only the current session. No logout-all endpoint in the first release.
- *Recommendation:* allow multiple sessions; logout affects the current session only; logout-from-all-devices is not provided in the first release (it would need a new endpoint and its own approval).
- *Rationale:* Matches the approved API and data model.
- *Trade-offs:* More live sessions to protect; no self-service way to end all sessions.
- *Information required:* product input on multi-device expectations.
- *Dependencies:* api-contracts §6.3.
- *Impact if unresolved:* session cardinality rules stay undecided.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-012 — Logout semantics** *(USER-FR-027, 073, 074, 084, 085; API-DEC-008)* — **Approved**
- *Approved choice:* Logout revokes the applicable refresh session. Already-issued access tokens may stay valid until expiry, up to the configured 15-minute lifetime. This residual window is documented (§9).
- *Recommendation:* revoke the session; accept the residual access-token window (§9).
- *Rationale:* Consistent with approved USER-DEC-008.
- *Trade-offs:* A copied access token works until it expires.
- *Information required:* owner approval; API-DEC-008 (204 versus 401 for stale tokens).
- *Dependencies:* DEC-022.
- *Impact if unresolved:* logout behavior cannot be finalized.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-013 — Refresh-token transport and client storage** *(API-DEC-002; USER-Q-004)* — **Approved (conditional)**
- *Approved choice:* Keep **Profile X** (current Render origins, cross-site) and **Profile Y** (future custom domains or a same-origin proxy) as **alternative designs, not simultaneously active production configurations**. **No production profile is selected** until deployment feasibility is confirmed. S2 custom domains remain the preferred direction. The refresh token is a `Secure`, `HttpOnly` cookie and the access token stays in memory.
- *Condition:* Condition: feasibility is confirmed for the profile to be used. Domain ownership is unconfirmed, and proxy feasibility is only assessed (a feasibility assessment, not a deployment change, is authorized). Profiles are documented in §10.3.
- *Recommendation:* Two alternative profiles (§10.3); no production profile selected until feasibility is verified.
- *Rationale:* Keeps the long-lived credential out of script-readable storage.
- *Trade-offs:* Introduces cookie, CORS, and CSRF obligations; may not suit non-browser clients.
- *Information required:* §10.2 deployment facts.
- *Dependencies:* DEC-014, 015, 016.
- *Impact if unresolved:* endpoints 2 and 3 cannot be finalized.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (conditional)

**USER-SD-DEC-014 — Cookie attributes** *(API-DEC-003)* — **Approved (conditional)**
- *Approved choice:* Baseline for the same-site profile (Y): `Secure`, `HttpOnly`, `Path=/`, `SameSite=Lax`, no `Domain`, and `__Host-` only if its requirements are met. Profile X necessarily uses `SameSite=None` (§10.3).
- *Condition:* Conditional on the profile chosen and on browser behavior supporting it (§10.3, §10.6).
- *Recommendation:* Per-profile cookie attributes in §10.3.
- *Rationale:* Cookie, SameSite, CORS, and CSRF decisions must be made together.
- *Trade-offs:* See §10.3.
- *Information required:* §10.2 deployment facts.
- *Dependencies:* DEC-013, 015, 016.
- *Impact if unresolved:* cookies cannot be set safely.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (conditional)

**USER-SD-DEC-015 — CORS policy and allow-list** *(API-DEC-003)* — **Approved**
- *Approved choice:* The sole allowed origin for direct browser-to-API calls is the frontend origin `https://varsha-homemade.onrender.com`. No wildcard origins. Credentialed requests only if required by the selected cookie strategy. Only the required methods and headers. The public API origin is not listed as the frontend's allowed origin.
- *Still open:* Revisit the allow-list if custom domains are configured (§10.4).
- *Recommendation:* Allowed origin `https://varsha-homemade.onrender.com` only; revisit when custom domains are configured (§10.4).
- *Rationale:* Origins are unknown.
- *Trade-offs:* Too broad exposes credentialed requests; too narrow blocks the frontend.
- *Information required:* actual frontend and API origins.
- *Dependencies:* DEC-013, 014, 016.
- *Impact if unresolved:* browser clients cannot call the API safely.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-016 — CSRF protection** *(API-DEC-003)* — **Approved**
- *Approved choice:* For browser requests that use cookie authentication: the custom header `X-CSRF-Protection`; rejected requests return **HTTP 403 with error code `REQUEST_REJECTED`**; **Origin validation** against the configured frontend origin; a missing `Origin` is **rejected on refresh and logout**; and a **session-scoped CSRF token** (decision 028). Origin validation or the header alone is not treated as a complete CSRF strategy.
- *Still open:* The exact endpoint list and how the frontend obtains and sends the token are documented in §10.5. The status and code are recorded in api-contracts §11.
- *Recommendation:* The approved baseline in §10.5, with a session-scoped CSRF token under both profiles.
- *Rationale:* Cookie-authenticated refresh and logout are CSRF-relevant.
- *Trade-offs:* See §10.5.
- *Information required:* §10.2 deployment facts; status code for a failed CSRF check.
- *Dependencies:* DEC-013, 014, 015.
- *Impact if unresolved:* refresh and logout stay exposed to cross-site forgery if cookies are used.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-017 — Rate-limiting scope, thresholds, and storage** *(USER-Q-002; USER-SEC-009; USER-DEC-013)* — **Open**
- *Open:* Thresholds are deferred pending operational review. Proposed limit targets, storage, client-IP identification, trusted-proxy assumptions, and store-failure behavior are in §11.1. Forwarded client-IP headers are not trusted until a trusted-proxy configuration is explicitly established. Reassess if the service runs multiple instances.
- *Recommendation:* The proposed limit targets and storage strategy in §11.1; thresholds set by the owner after operational review; no trust in forwarded headers without explicit trusted-proxy configuration.
- *Rationale:* In-process counters are per-instance and proxies change which address the service sees.
- *Trade-offs:* See §11.
- *Information required:* instance count; proxy and gateway topology; thresholds; fail-open versus fail-closed policy.
- *Dependencies:* §10.2 facts; DEC-024.
- *Impact if unresolved:* USER-SEC-009 cannot be implemented correctly.
- *Approval:* Not approved.

**USER-SD-DEC-018 — Secret management, key rotation, and compromise response** *(USER-SEC-003, 004)* — **Approved**
- *Approved choice:* Server-side environment secrets provided through the hosting platform's secret-management mechanism. Google client secrets and token-signing secrets stay out of source control and frontend code. Required secrets are validated at startup. Key identifiers and a safe rotation procedure where applicable. **An accountable owner is assigned before production** (no name is invented).
- *Still open:* Still open: the named accountable owner for secret management and emergency rotation, and confirmation of how Render exposes secrets. **The exposed Google client secret is kept for now by owner decision and recorded as an unresolved security risk (§12, RISK-001).**
- *Recommendation:* Server-side environment secrets via the hosting platform; startup validation; key identifiers and rotation; accountable owner to be assigned before production (§12).
- *Rationale:* Secrets handling depends on the unknown hosting and operations model.
- *Trade-offs:* See §12.
- *Information required:* hosting environment; who holds and rotates secrets; who authorizes emergency revocation; rotation cadence.
- *Dependencies:* DEC-002, 006.
- *Impact if unresolved:* no secure deployment procedure.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (approach)

**USER-SD-DEC-019 — Security-event types and storage** *(USER-DM-DEC-006; USER-FR-011, 065, 066, 082)* — **Approved**
- *Approved choice:* Persist mandatory security events. Use redacted operational logs. Never include raw tokens, token digests, passwords, credentials, client secrets or other secrets in logs.
- *Still open:* Exact event definitions, retention periods, and operational details remain **Open**. The nonce-replay event is proposed in addition (§4.4).
- *Recommendation:* the mandatory event types in §13.2; database records plus redacted structured logging.
- *Rationale:* Requirements say events are "recorded" and "recorded for review".
- *Trade-offs:* Database storage adds write load and a retention duty.
- *Information required:* owner approval; any additional event types wanted.
- *Dependencies:* DEC-020, 024.
- *Impact if unresolved:* USER-FR-082 recording cannot be implemented.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (principle only)

**USER-SD-DEC-020 — Retention of events, sessions, and consumed-token records** *(USER-DM-DEC-007, 012; USER-PRE-001)* — **Open**
- *Open:* Retention periods are not chosen. Needs privacy, legal, and operational review (§13.3).
- *Recommendation:* No periods. Retention requirements are documented in §13.3 for review.
- *Rationale:* Privacy and legal obligations are not yet reviewed.
- *Trade-offs:* Longer retention improves detection but increases data held.
- *Information required:* outcome of the privacy/legal/security/operational review.
- *Dependencies:* USER-PRE-001; Order retention analysis.
- *Impact if unresolved:* cleanup and recognition limits cannot be defined.
- *Approval:* Not approved.

**USER-SD-DEC-021 — New dependencies** *(requirements §11.1)* — **Approved**
- *Approved choice:* Each dependency is approved individually before installation, with its purpose, security implications, and alternatives documented. Nothing is installed.
- *Still open:* Each individual dependency remains **Open** until approved; candidates are in §14.1. The driver choice (a standard PostgreSQL driver, not the Supabase client library) is approved in principle under decision 026.
- *Recommendation:* decide each candidate in §14 individually; Node.js's built-in cryptographic module needs no new dependency.
- *Rationale:* The project forbids unapproved dependencies.
- *Trade-offs:* Hand-written security code is riskier than established libraries.
- *Information required:* per-dependency approval.
- *Dependencies:* DEC-001, 002, 017.
- *Impact if unresolved:* any needed library cannot be installed.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (process only)

**USER-SD-DEC-022 — Immediate access-token revocation mechanism** *(USER-FR-085)* — **Approved**
- *Approved choice:* Immediate access-token revocation is deferred in the first release. Access tokens may remain valid until expiry.
- *Recommendation:* no mechanism proposed. Deferred: revoking a refresh session does not automatically invalidate an issued JWT access token (§9).
- *Rationale:* USER-DEC-008 accepts this behavior.
- *Trade-offs:* See §9.
- *Information required:* none unless the owner decides immediate invalidation is needed.
- *Dependencies:* DEC-012.
- *Impact if unresolved:* none for the first release as approved.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-023 — Client-side refresh coordination requirements** *(USER-FR-083; api-contracts §6.2)* — **Approved**
- *Approved choice:* One refresh request at a time per browser tab. No automatic retry of a failed refresh. The possibility of sign-out after a lost response is documented.
- *Still open:* Cross-tab coordination is **recommended** (frontend-contract.md) and remains **Open** pending your approval.
- *Recommendation:* the client allows at most one refresh in flight per client context and does not auto-retry a refresh after an ambiguous failure; whether cross-tab coordination is required is the owner's choice (§6.4).
- *Rationale:* Reduces collisions that would be mistaken for reuse.
- *Trade-offs:* Reduces but cannot eliminate races, lost responses, or ambiguous outcomes.
- *Information required:* frontend architecture; whether cross-tab coordination is required.
- *Dependencies:* DEC-004, 008, 009; frontend facts.
- *Impact if unresolved:* unnecessary sign-outs; the frontend contract is unspecified.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (single-tab coordination only)

**USER-SD-DEC-024 — Event-persistence failure policy and monitoring** *(USER-FR-082; USER-SEC-008)* — **Approved**
- *Approved choice:* A request stays rejected if a required security-event write fails. Event-write failures must be detectable and monitored. Logging and monitoring failures must not expose sensitive information.
- *Still open:* The monitoring mechanism remains **Open** pending review of operational tooling.
- *Recommendation:* a rejection never depends on event persistence; failures are detected, surfaced, and monitored (§13.2). Any policy that fails closed on event-write failure needs a specific, justified approval.
- *Rationale:* Audit availability must not turn a rejection into an acceptance or block unrelated operations.
- *Trade-offs:* Possible loss of an event if no fallback sink exists.
- *Information required:* existing monitoring and alerting capability; who responds to alerts.
- *Dependencies:* DEC-019, 020.
- *Impact if unresolved:* event-write failures may go unnoticed.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (policy only)

**USER-SD-DEC-025 — Supabase role and HTTP API restriction (R1, R2)** *(R1, R2; USER-SEC-011; USER-SEC-004)* — **Approved**
- *Approved choice:* Supabase hosts PostgreSQL only; Express handles customer authentication and issues and verifies its own JWTs; Supabase Auth is not used in the first release; no dependence on the Data API unless you approve a change. The authentication tables must not be exposed through Supabase's HTTP APIs, and the access controls are designed before implementation.
- *Still open:* Schema, roles, privileges and RLS design are completed at physical design. **RLS is not a replacement for database permissions** (§2.1).
- *Recommendation:* See §2.1: separate owner and application roles with least privilege, a non-exposed schema, RLS as defense in depth.
- *Rationale:* Stated in the section referenced above.
- *Trade-offs:* Stated in the section referenced above.
- *Information required:* none beyond the open items listed.
- *Dependencies:* DEC-013, 014, 016 (profiles) and the physical database design where relevant.
- *Impact if unresolved:* the dependent elements cannot be finalized.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (in principle)

**USER-SD-DEC-026 — Database connection method, driver, and application role** *(USER-SEC-004; USER-SD-DEC-007)* — **Approved (conditional)**
- *Approved choice:* Use a **standard PostgreSQL driver** (such as `pg`), not the Supabase client library. **Test Render-to-Supabase connectivity first**; prefer a direct connection if supported, otherwise the session-mode pooler. Define a **dedicated least-privilege Express database role**, separate from the database owner, through the approved database setup process.
- *Condition:* Condition: the connectivity test result, and confirmation of the connection method and role setup before the physical design is finalized. No connection string, role or credential is invented. The `pg` package approval remains pending (§14.1).
- *Recommendation:* See §2.1: driver, connection method, role.
- *Rationale:* Stated in the section referenced above.
- *Trade-offs:* Stated in the section referenced above.
- *Information required:* none beyond the open items listed.
- *Dependencies:* DEC-013, 014, 016 (profiles) and the physical database design where relevant.
- *Impact if unresolved:* the dependent elements cannot be finalized.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (conditional)

**USER-SD-DEC-027 — Browser binding of the sign-in nonce (page-held binder)** *(USER-SEC-001; USER-SD-DEC-001)* — **Approved (conditional)**
- *Approved choice:* Approve the **page-held binder (B2)** in principle: the page keeps a random secret in memory, sends only its hash when requesting a nonce, and sends the raw value at sign-in. **Its limitations must be validated explicitly** (§4.4).
- *Condition:* Condition: the limitations listed in §4.4 are validated before this decision is final. A nonce alone does not prove browser identity, and the binder does not either.
- *Recommendation:* B2 in principle, with explicit validation of its limitations (§4.4).
- *Rationale:* Stated in the section referenced above.
- *Trade-offs:* Stated in the section referenced above.
- *Information required:* none beyond the open items listed.
- *Dependencies:* DEC-013, 014, 016 (profiles) and the physical database design where relevant.
- *Impact if unresolved:* the dependent elements cannot be finalized.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24). (in principle)

**USER-SD-DEC-028 — Session-scoped CSRF token** *(USER-SD-DEC-016; USER-SEC-007)* — **Approved**
- *Approved choice:* A **session-scoped CSRF token** is used for cookie-authenticated requests under Profile X, and the same contract is retained under Profile Y. Only its digest is stored. It is sent in `X-CSRF-Protection`.
- *Still open:* Details in §10.5. The session record gains a CSRF-token digest (data-model §14).
- *Recommendation:* Session-scoped CSRF token in both profiles (§10.5).
- *Rationale:* Stated in the section referenced above.
- *Trade-offs:* Stated in the section referenced above.
- *Information required:* none beyond the open items listed.
- *Dependencies:* DEC-013, 014, 016 (profiles) and the physical database design where relevant.
- *Impact if unresolved:* the dependent elements cannot be finalized.
- *Approval:* Approved by the project owner in the Step 5b decision message (2026-10-24).

**USER-SD-DEC-029 — Profile X third-party-cookie fallback (re-sign-in design)** *(USER-SD-DEC-013, 014; USER-FR-070)* — **Approved**
- *Approved choice:* The design accounts for the **browser-compatibility risk of Profile X**: because the Render origins are cross-site, a browser that blocks third-party cookies may prevent silent refresh. **A CSRF token does not solve that.** Users may need to sign in again, and the design and frontend contract handle it (§10.6).
- *Still open:* The exact user-interface behavior is for the frontend review; the structure is in §10.6 and frontend-contract.md.
- *Recommendation:* See §10.6.
- *Rationale:* Stated in the section referenced above.
- *Trade-offs:* Stated in the section referenced above.
- *Information required:* none beyond the open items listed.
- *Dependencies:* DEC-013, 014, 016 (profiles) and the physical database design where relevant.
- *Impact if unresolved:* the dependent elements cannot be finalized.
- *Approval:* Raised by the project owner in the Step 5b clarification (2026-10-24).

## 4. Google Identity Validation and Trusted Identity Handling
**Decision:** USER-SD-DEC-001 · **Traces:** USER-FR-001–011, 060, 061, 069; USER-SEC-001, 005, 006; API-DEC-001, 004

**[Required]**
- Google credentials MUST be validated on the server; client-supplied identity claims are not trusted (USER-FR-002, USER-SEC-001).
- Validation MUST confirm authenticity, issuer, intended audience, and expiry (USER-SEC-001), and MUST reject identities whose email is not verified (USER-FR-006).
- The stable Google account identifier is the identity basis; email is not the sole key (USER-FR-005).
- Failures create no session and reveal no Google error detail (USER-FR-007, 061, USER-SEC-006).

**[Proposed] validation checks** for the ID-token flow (recommended proposal in api-contracts §6.1):
1. Verify the signature with Google's published public signing keys, obtained over HTTPS and cached according to Google's published caching signals.
2. Verify the issuer is a Google issuer value.
3. Verify the audience equals this application's configured Google client identifier (from configuration, never from the request).
4. Verify the token is not expired (small, configurable clock tolerance).
5. Verify the email-verified claim is true; otherwise treat as `AUTH_GOOGLE_FAILED` (API-DEC-004).
6. Use the subject claim as the Google account identifier.
7. Take name and email only from the validated token, never from request fields.

**[Proposed]** All failures map to the same generic client outcome; the specific reason is recorded internally only, without the credential value (USER-SEC-008).

**[Unresolved] replay protection.** Validating the credential (the checks above) is a different question from ensuring the credential belongs to the *current* authentication attempt. See §4.1 and §4.2. The Google flow, including replay protection, stays open under API-DEC-001 and USER-SD-DEC-001 until the owner approves it.

**[Unresolved] library.** Implementing signature verification by hand is error-prone. An established verification library is the usual approach, but no dependency is approved (USER-SD-DEC-021). Node.js's built-in cryptographic module cannot by itself fetch and rotate Google's keys.

**Trade-offs.** Using Google's verified ID token avoids storing or handling Google passwords or long-lived Google tokens. Not requesting additional scopes keeps stored data minimal (USER-FR-048). The authorization-code flow (a possible alternative if the frontend requires it) would add a callback endpoint and a client secret to protect.

**Failure behavior [Required]** If Google's keys cannot be obtained, sign-in fails safely (503, no session) and never grants access (requirements §7).

### 4.1 Two separate questions: validity and attempt binding

| Question | What it establishes | Addressed by |
|---|---|---|
| **A. Is this a genuine, current, Google-issued credential for this application, for a verified email?** | Signature, issuer, audience, expiration, and the verified-email claim | The validation checks above — **[Required]** for the ID-token flow (USER-SEC-001, USER-FR-006) |
| **B. Does this credential belong to the current authentication attempt, and has it not been improperly replayed or injected?** | Freshness and binding to the attempt | A nonce (ID-token flow) or state and PKCE (authorization-code flow) — **[Unresolved]** |

Passing A does **not** prove B. A genuine credential obtained from another context (a log, a proxy, a malicious page, an earlier sign-in) still passes A until it expires. This design does **not** claim that validating the signature and standard claims alone provides complete replay protection.

### 4.2 Options for attempt binding (USER-SD-DEC-001; API-DEC-001 open)

| Option | How it works | Protection | Costs and requirements |
|---|---|---|---|
| **N0. No binding** | Validate only (A) | None beyond expiry and HTTPS | **Residual risk:** a captured, still-valid credential can sign in as that customer. Requires **explicit owner acceptance**. |
| **N1. Client-generated nonce, checked for match** | The client creates a random value, has Google embed it in the credential, and sends it so the service checks the credential's nonce claim equals it | Defeats injection of a credential minted for a different attempt | The service cannot tell if the value was used before, so a captured credential with its nonce is still replayable until expiry. Changes the sign-in request schema. |
| **N2. Server-issued, single-use nonce** | The service issues a nonce, remembers it briefly, and the credential must carry it; consumed on first use | Strongest for the ID-token flow | Needs a way for the client to obtain the nonce (**a new endpoint, not in the approved API inventory**, api-contracts §1) and short-lived server state (**not in the approved data model**, data-model §1). Needs approval of both. |
| **C. Authorization-code flow** | Redirect-based; the service exchanges a code with Google | State and PKCE bind the response to the attempt; codes are single-use at Google | Needs a callback endpoint (**not in the approved inventory**), exact redirect-URI registration, a client secret to protect, and state storage. Needs approval of API changes. |

**If a nonce is used (N1 or N2), the design requires** *(all [Proposed]; values left to the owner)*:
- **Generation:** from a cryptographically secure random source with entropy that makes guessing infeasible.
- **Binding:** tied to one sign-in attempt; for N2, also to the requesting client context, with the mechanism unresolved.
- **Validation:** the credential's nonce claim must exactly equal the expected value; absent or mismatched means failure.
- **Expiration:** short and bounded, configured by the owner; no value is proposed here.
- **Replay handling:** single use; consumed on first presentation whether it succeeds or fails; a second presentation fails generically and is recorded as a security event that contains no credential.
- **Secrecy:** never logged.

**If the authorization-code flow (C) is chosen, the design requires** *(all [Proposed])*: a state value that is unguessable, bound to the user's browser, single-use, short-lived and validated on return; PKCE; exact redirect-URI matching; failures generic and recorded.

**Owner decision (Step 5b):** the ID-token flow with a server-issued, single-use nonce (option N2) is approved in principle (USER-SD-DEC-001). The complete flow is in §4.4.

### 4.3 Flow comparison in the confirmed context (USER-SD-DEC-001, Approved (conditional))
Context: React frontend at `https://varsha-homemade.onrender.com`; backend API at `https://varsha-customer-service.onrender.com`; a Google OAuth *web application* client in the `varsha-homemade` Google Cloud project with the frontend origin as an authorized JavaScript origin; browser clients only. The flow is **not approved**.

| Aspect | ID-token flow | Authorization-code flow |
|---|---|---|
| What the browser sends to the API | A Google-signed identity credential | A short-lived, single-use code, which the backend exchanges with Google |
| Backend validation | Signature, issuer, audience, expiry, verified email, subject (§4) | The exchange with Google returns the identity; the backend still validates the returned ID token the same way |
| Google client secret | **Not needed** by the backend | **Needed** on the backend only; must never reach React or the browser |
| Replay and binding | Needs a nonce (§4.2). A client-generated nonce alone is **not** sufficient: the service cannot tell if a captured credential and nonce were already used. Strong protection needs a server-issued single-use nonce, which means a new endpoint and short-lived server state. | State and PKCE bind the response to the attempt, and Google's code is single-use |
| API implications | Same endpoint with a `credential` field; a server-issued nonce would add an endpoint | The sign-in request schema changes (code rather than credential); a redirect-based variant also needs a callback endpoint and an authorized redirect URI registered for the API origin `https://varsha-customer-service.onrender.com` (none is known to be registered) |
| Complexity | Lower | Higher |
| API origin `https://varsha-customer-service.onrender.com` (now confirmed) | Relevant to CORS and cookies, because the frontend and API are different origins | Relevant to redirect URIs and secret handling; a redirect-based variant needs a callback endpoint that is not in the approved API |

**Approved in principle (Step 5b):** the ID-token flow with a **server-issued, single-use nonce** (§4.4). It needs no client secret on the backend. If the nonce were dropped, the residual replay risk would need your explicit acceptance. Google's sign-in configuration documents an optional `nonce` described as a random string used to prevent replay attacks; verify where the claim appears in the token when building.

**Exposed secret.** The previously shared Google client secret must be treated as exposed and rotated (§12). It is not reproduced in any specification.

### 4.4 Complete flow: ID-token verification with a server-issued, single-use nonce (USER-SD-DEC-001, 027)
**Status:** approved in principle (decision 001, five-minute nonce lifetime) and the page-held binder is approved in principle (decision 027), **subject to your review of this documented flow**. The new endpoint and the nonce record are explained below and written into api-contracts §11 and data-model §14.

**Why a new endpoint and a nonce record are needed.** Only the server can know that a nonce was issued and used once. A signed nonce with no stored record cannot prove "once", and a cookie-held nonce would not stop replay within its lifetime. So one endpoint issues it and one short-lived record remembers it.

**The flow**
1. The page generates a **binder**: a random secret kept only in memory. It computes the binder's hash.
2. The page calls the nonce endpoint (`POST /api/v1/auth/google/nonce`, public, rate-limited) with the binder hash.
3. The server generates a random **nonce** from a cryptographically secure source and stores a record: the nonce's digest, the binder hash, creation time, expiry (five minutes), and an unconsumed marker. It returns the nonce and its lifetime.
4. The page starts Google sign-in passing the nonce. The customer signs in, and Google returns a signed **ID token** carrying that nonce.
5. The page posts the credential and the raw binder to `POST /api/v1/auth/google`.
6. The server **verifies the ID token**: signature against Google's published public keys, issuer, audience equal to our configured Google client identifier, expiry, and verified email. It takes Google's stable `sub` as the identity key.
7. The server **atomically consumes the nonce**: one conditional update marks the nonce used only if it exists, is unused and is unexpired, and reports how many rows changed. Exactly one request can win.
8. The server checks that the **binder** hashes to the stored binder hash.
9. On success it finds or creates the customer, creates the session with its CSRF token (decision 028), and issues the access token and refresh token.

**Properties**
| Question | Answer |
|---|---|
| How is the nonce tied to the initiating attempt? | The nonce appears inside the Google-signed token and is stored with the binder hash for the page that requested it. A token minted for another nonce cannot match. |
| How is it consumed atomically? | A single conditional update, as with refresh-token consumption (§8.1). The nonce is consumed on the **first presentation of a validly signed Google token carrying it**, whether or not sign-in then succeeds, so repeated guessing cannot probe it. |
| Expiry | Five minutes, configurable, with an upper bound. An expired nonce means the user starts sign-in again with a fresh nonce; there is no automatic retry loop. It is separate from Google's own token expiry. |
| Replay | A consumed, expired, unknown or mismatched nonce, or a wrong binder, fails with the generic `AUTH_GOOGLE_FAILED`, creates no session, and records a security event that contains no credential or binder. |
| Is a PostgreSQL record necessary? | Yes. In-process memory would be lost on restart or redeploy and would not be shared if the service scales. Expired records are cleaned up opportunistically at issuance, so no scheduler is required (none is decided). |
| Use by an unrelated attempt | It requires the matching Google-signed token, the single-use nonce, and the binder. The nonce alone is worthless without a token carrying it. |

**Browser binding (B2) and its limitations (to be validated explicitly).** A nonce alone does not prove it belongs to the same browser, and the binder does not prove browser identity either; it is a secret known to the page.
| It helps against | It does **not** help against |
|---|---|
| A token and nonce leaked by a route that did not include the sign-in request, such as a log, a referrer, or another site | Script injection on the page, which can read the binder from memory |
| An attacker replaying a token on another device without the binder | Capture of the sign-in request itself, which carries both the credential and the binder |
| | A token stolen and used **before** the legitimate page uses it, if the binder is also stolen |
It needs no cookie, so it works under Profile X even when third-party cookies are blocked.
**Validation required before decision 027 is final:** (a) a threat-model review that confirms the table above and records any gap; (b) negative checks at acceptance: a wrong binder is rejected, a missing binder is rejected, a reused nonce is rejected, an expired nonce is rejected, and a token carrying a different nonce is rejected; (c) confirmation that the binder is never logged.

**API contract and data changes** (explained and written into the other documents): one new endpoint, a `binder` field and a `csrfToken` response field on sign-in, the endpoint inventory growing from four to five, a Sign-in Nonce entity, a CSRF-token digest on the session, and a new security-event type for nonce replay. See api-contracts §11 and data-model §14.

**Abuse:** the issuance endpoint is unauthenticated and creates state, so it is rate-limited and caps outstanding nonces per client (§11.1).

## 5. Access Tokens
**Decisions:** USER-SD-DEC-002, 003 · **Traces:** USER-FR-020–026, 072, 074, 084; USER-SEC-002, 007, 010; USER-DEC-006, 007

**[Required]**
- Access tokens are short-lived JWTs; initial lifetime 15 minutes, configurable and enforced (USER-FR-026, 072).
- Every protected request validates signature, expiry, and approved claims (USER-FR-021, USER-SEC-002).
- A valid token resolves to exactly one customer (USER-FR-023). Authorization uses the identity derived from it, never a client-supplied identifier (USER-FR-053).
- Access tokens are stateless; no record exists in the data model (data-model §1, USER-DM-AC-020). Logout does not necessarily invalidate them (USER-FR-074).

**[Approved (conditional)] Algorithm and key type (USER-SD-DEC-002).** HS256 is approved provided Express is the only application JWT issuer and verifier in the first release.

| Option | Benefits | Costs / risks |
|---|---|---|
| HMAC-based (HS256) with a strong shared secret | Simple; one secret; sufficient when this single service both issues and verifies | Any party able to verify can also forge; sharing the secret with another service widens exposure |
| Asymmetric (for example ES256 or RS256) | Verifiers need only the public key; private key stays with the issuer | More key management; larger tokens or slower operations |

*Approved (conditional):* HS256, **only while** this Express service is the sole issuer and verifier of application access tokens, no other component needs to verify them, and the secret-sharing implications are accepted. The verifier accepts only the configured algorithm and rejects unsigned tokens, and the secret meets a minimum strength checked at startup (§12). If another service must verify application JWTs independently, the implications are presented to you **before** the algorithm changes (§5.1).

**[Approved] Claims (USER-SD-DEC-003).** Include only: issuer, audience, subject (the internal customer identifier), issued-at, and expiry. Do not include name, email, or Google identifier (data minimization; USER-DEC-010; USER-SEC-006). No per-token identifier is needed because no access-token revocation list is proposed. Validate issuer and audience on every request so tokens meant for another context are refused.

**[Approved] Clock skew (USER-SD-DEC-003).** The skew allowance is small, bounded, configurable, and never bypasses expiry. The value is not chosen here. Any allowed clock skew MUST be small, bounded, and explicitly configured; no default larger than zero is proposed, and a value is an owner/operational input. Clock-skew tolerance MUST NOT be used to bypass token expiration: a token that is expired beyond the configured tolerance is rejected, and tolerance may never exceed a documented maximum. A distinct expired-token error code (API-DEC-005) does not bypass validation (api-contracts §4).

**Consequence of statelessness.** A stolen access token is usable until it expires (up to 15 minutes by default). This is the accepted behavior under USER-DEC-008; shortening the lifetime reduces exposure at the cost of more refresh traffic.

### 5.1 Signing algorithm: HS256 controls and the verification topology (USER-SD-DEC-002, Approved (conditional))
**Two different key sets (not to be confused):**
| | Google's ID-token verification keys | Our application's JWT signing key |
|---|---|---|
| Purpose | Verify Google-issued ID tokens at sign-in | Sign and verify our own access tokens |
| Who owns it | Google; we fetch the public keys and never hold or rotate them | This service |
| Our responsibility | Fetch, cache, validate issuer, audience, expiry | Generate, store, protect, rotate |

**Topology (confirmed):** Express is the only issuer and verifier. No Apigee or other gateway is in the request path and none verifies the application's JWTs. Supabase Auth and the Data API are not used, so Supabase verifies nothing.

**Required controls for HS256**
- A strong random signing secret held **only in server-side secret configuration**. It is never in React, source control, logs or specification files.
- Validate signature, **issuer, audience, subject, issued-at and expiry** on every request. Accept only the configured algorithm; reject unsigned tokens.
- Clock skew is small, bounded, configurable, and never bypasses expiry.
- A key identifier in the token header, and rotation with a short overlap during which the previous key verifies only (about one access-token lifetime plus the skew). Emergency compromise response: replace the key at once with no overlap, so every outstanding access token stops validating.
- Startup validation refuses to start if the secret is missing or too weak.

**If another service must verify application JWTs.** Presented to you before any change:
| Aspect | HS256 (shared secret) | Asymmetric (for example ES256) |
|---|---|---|
| Verifier needs | The same secret that signs, so it can also forge | Only the public key; cannot forge |
| Secret sharing | Widens the compromise surface | Private key stays with Express |
| Key distribution | Not suitable | Public key by configuration; a published key-set endpoint would be a **new endpoint** needing API-contract approval |
| Rotation | Coordinated secret change everywhere | Rotate the key pair and distribute the new public key first |
| Cost | Lowest | Higher |
Because access tokens live 15 minutes and are stateless, switching later invalidates tokens for at most one refresh cycle.

## 6. Refresh-Token Strategy
**Decisions:** USER-SD-DEC-004, 005 · **Traces:** USER-FR-070–083; USER-SEC-011; USER-AC-015–016, 026–029; USER-Q-001; USER-DM-DEC-013; api-contracts §6.2

### 6.1 Required outcomes (any strategy)
**[Required]**
- A valid, unexpired, unrevoked refresh token yields a new access token without Google sign-in (USER-FR-070).
- Lifetime: 7 days by default, configurable (USER-FR-071, 072).
- A consumed refresh token is not reusable (USER-FR-081).
- Concurrent requests using the same token: at most one succeeds (USER-FR-083).
- Presenting an invalidated token is rejected and recorded as a security event, without logging the token or disclosing prior state (USER-FR-082).
- A revoked session cannot refresh (USER-FR-073, 077).
- A disclosed token store yields no usable refresh tokens (USER-SEC-011).
- Refresh tokens are not assumed to be JWTs (USER-FR-075).
- Rotation is the **recommended** initial policy, subject to security-design confirmation (USER-FR-080).

### 6.2 Options analysis
| Option | How it works | Evaluation against required outcomes |
|---|---|---|
| **A. Rotating, single-use tokens** | Each successful refresh consumes the presented token and issues a replacement | Directly satisfies USER-FR-081 and 083 and enables USER-FR-082 detection. Cost: a client that loses the response cannot refresh (see §6.4). |
| **B. Non-rotating, reusable token** | One token valid for its whole life; presented repeatedly | **Conflicts with USER-FR-083 as written**, which permits at most one successful use of a token by concurrent requests, and a reusable token has no "consumed" state, so USER-FR-081/082 cannot detect theft. A non-rotating design must not be approved merely on token validation (data-model §5.4, USER-DM-AC-023). No viable variant is identified here. |
| **C. Rotation with a bounded grace/retry window** | A consumed token may be reused briefly | Would permit consumed-token reuse, which contradicts USER-FR-081 and api-contracts §6.2. **Requires an approved requirements change**; not recommended and not proposed. |

**Approved (USER-SD-DEC-004): Option A, strict single-use rotation.** It is the only option that meets the required outcomes without requirement changes. This resolves USER-DM-DEC-013 for the rotation question; session lifetime and the other items of that data-model decision are handled in §8.

### 6.3 Token format and generation (USER-SD-DEC-005)
- **[Approved]** Opaque random value, long enough to be infeasible to guess (at least 256 bits of entropy is a common practice), generated by a cryptographically secure random source (Node.js's built-in cryptographic module provides one; no new dependency).
- **[Approved]** Not a JWT: nothing about identity or expiry is embedded; the server holds all state. This meets USER-FR-075 and means a disclosed token carries no information.
- **Trade-off:** server-side state is required for every refresh (a database lookup), unlike self-contained tokens; this is needed anyway for revocation and single use.

### 6.4 Lost responses, multiple tabs, and client coordination (USER-SD-DEC-009, 023)
**[Approved]** Refresh tokens are single-use (rotation approved, USER-SD-DEC-004). A *legitimate* client can therefore present an already-consumed token in several situations, all of which are indistinguishable from malicious reuse:
- a refresh succeeded on the server but its response was lost;
- two browser tabs or windows refresh at nearly the same moment;
- a request was resent after a timeout, by the client or by a network component;
- the database outcome of a refresh was ambiguous (§8.1).

- **[Required]** No grace period or reuse of a consumed token is permitted unless the requirements are formally revised and approved (USER-FR-081; api-contracts §6.2; data-model §5.4).
- **[Approved]** Client-side coordination (USER-SD-DEC-023, single tab): one refresh request at a time per browser tab, with the result shared among callers in that tab; **no automatic retry** of a failed refresh; a failed refresh ends the session in that client. **[Open]** Cross-tab coordination is deferred until the frontend implementation approach is reviewed. The possibility of sign-out after a lost response is accepted (USER-SD-DEC-009).
- **Limit of coordination.** Single-flight within one tab reduces concurrency but does not eliminate races. Multiple tabs would need cross-tab coordination, which is a frontend responsibility not specified here. Lost responses and ambiguous database outcomes cannot be prevented by client coordination at all.
- **Reuse detection and revocation of the whole session are separate policies** (§8.2). Revocation on reuse is approved (USER-SD-DEC-008), so each of the situations above signs the user out of that session.

**Approved by the project owner (Step 5):** strict single-use rotation (USER-SD-DEC-004); reuse of a consumed token rejects the request, records the event, and revokes the session (USER-SD-DEC-008); the resulting sign-outs from lost responses and concurrent requests are accepted (USER-SD-DEC-009); one refresh at a time per tab and no automatic retry (USER-SD-DEC-023, single-tab).

**Still open:** cross-tab coordination (USER-SD-DEC-023).

A grace or retry window is **not** introduced. It would require a formal change to USER-FR-081 and api-contracts §6.2.

### 6.5 Demonstration: how recommended Option A satisfies the approved requirements
This shows the recommendation meets each applicable approved requirement. Option A (strict rotation) is approved under USER-SD-DEC-004; the mechanisms in §7 and §8 follow their own decisions.

| Approved requirement | How Option A (with §7 storage and §8 mechanisms) meets it | Depends on |
|---|---|---|
| USER-FR-070: valid token yields new access token without Google sign-in | A valid, unconsumed, unexpired token in an unrevoked session is exchanged for an access token and a successor | — |
| USER-FR-071, 072: 7-day configurable lifetime | Each issued token's expiry is computed from configuration at issue time | USER-SD-DEC-010 for the absolute cap |
| USER-FR-073, 077: revoked session cannot refresh | Atomic transition (§8.1) requires the session to be unrevoked; revoked sessions fail it | — |
| USER-FR-075: not assumed to be a JWT | Opaque random value (§6.3) | — |
| USER-FR-076: generic rejection | All failure causes return the same generic outcome | — |
| USER-FR-080: rotation recommended | Option A is rotation | Approved (USER-SD-DEC-004) |
| USER-FR-081: consumed token never reusable | Consumption is a one-way, durable state change made by the atomic transition; a consumed token can never pass it again | Consumed record retained (§13) |
| USER-FR-082: invalidated token rejected and recorded, no token logged, no state disclosed | Presentation of a consumed or revoked token fails the transition, an event is recorded using opaque identifiers only, and the response is the generic one | Retention of consumed records (§13) |
| USER-FR-083: concurrent requests, at most one success | The atomic conditional transition has one winner by construction; consume and successor-issue commit together | USER-SD-DEC-007 |
| USER-SEC-011 / USER-AC-029: no usable tokens on store disclosure | Only a one-way digest of a high-entropy token is stored (§7) | USER-SD-DEC-006 |
| USER-SEC-008, USER-FR-028: no tokens in logs or events | Raw token, digest, and credentials are excluded from logs and events (§10.1, §13) | — |
| USER-SEC-009: abuse limiting on refresh | Per-client and per-session limits (§11) | USER-SD-DEC-017 |

**Residual limitation:** the lost-response case (§6.4) is a consequence of satisfying USER-FR-081, not a gap in it. It is a usability cost for owner acceptance.

## 7. Refresh-Token Storage and Disclosure Protection
**Decision:** USER-SD-DEC-006 · **Traces:** USER-SEC-011; USER-AC-029; USER-FR-028; USER-DM-DEC-004; USER-DM-AC-009

**[Required]** Raw refresh tokens are never stored or logged; disclosure of the token store must yield no usable refresh tokens.

**Security invariants [Required unless noted]** (USER-SEC-011; USER-AC-029; USER-FR-028; USER-FR-075; USER-DM-AC-009, 023):
1. Refresh tokens are generated with a cryptographically secure random source.
2. Entropy makes guessing infeasible.
3. Raw refresh tokens are never persisted in the database or logs.
4. The stored representation does not enable an attacker with database-only access to recover or feasibly guess usable refresh tokens.
5. Verification and lookup follow the approved token-storage design (USER-SD-DEC-006): a one-way digest of each token, never the raw value.
6. Rotation, revocation, expiration, and the token-family relationship stay consistent with the approved data model. In the data model the token family is the Refresh Session; a session's tokens share its revocation state (data-model §5, §6).

| Approach | Disclosure result | Cost |
|---|---|---|
| Unkeyed one-way digest (for example SHA-256) of a high-entropy random token | The store contains values that cannot be reversed or guessed in practice because the token is long and random | None beyond hashing; computable with Node.js's built-in cryptographic module; lookup is by digest |
| Keyed digest (HMAC with a server secret held outside the database) | Same, plus an attacker needs the key to verify guesses | A new secret to manage; rotating the key invalidates stored digests unless key identifiers are kept |
| Reversible encryption | Store plus key yields working tokens | Worst fit for USER-SEC-011; not recommended |
| Slow password hashing (bcrypt-style) | Strong for low-entropy secrets | Unnecessary for 256-bit random values and adds per-request cost; not recommended |

**Unkeyed versus keyed digest (USER-SD-DEC-006, open):**

| Aspect | Unkeyed digest | Keyed digest (secret held outside the database) |
|---|---|---|
| Database-only **read** access | Digests cannot be reversed; guessing is infeasible because tokens are long and random → no usable tokens | Same |
| Database **write** access | An attacker can compute the digest of a token they choose and insert a session record that the service would accept | The attacker also needs the secret, so cannot forge a session |
| If token entropy proves weaker than intended (implementation flaw) | Offline guessing against stolen digests is possible | Offline guessing needs the key as well |
| Key management | None | A new secret to generate, store, and protect; rotation needs a key identifier stored alongside each digest (a data-model follow-up) and a dual-verification period, or all tokens become unverifiable |
| Impact of losing the key | — | If lost, all stored digests are unverifiable and sessions end; if exposed, protection reverts to unkeyed |
| Lookup | By digest | By keyed digest; the service needs the key at lookup time |

**Approved (USER-SD-DEC-006):** store a one-way digest of each refresh token, never its raw value. **Begin with an unkeyed digest**, provided the final design satisfies the approved security requirements (USER-SEC-011). **Reassess a keyed digest if the threat model requires protection against database write access**, since an unkeyed digest lets an attacker with write access forge a session (see the comparison above). A keyed digest would add a second secret and likely a stored key identifier, both of which would need their own approval and a data-model amendment. Raw refresh tokens are never logged or persisted.

**[Proposed]** Digest values are treated as sensitive: never logged, never put in security events (data-model §5.5, USER-DM-AC-018), and compared via the database lookup rather than ad-hoc string comparison in application code.

**[Open]** The exact digest algorithm, and the point at which a keyed digest is adopted (USER-SD-DEC-006 / USER-DM-DEC-004). No physical schema or storage implementation is decided here.

## 8. Rotation, Reuse Detection, Concurrency, and Session Revocation
**Decisions:** USER-SD-DEC-007, 008, 010, 011 · **Traces:** USER-FR-073, 077, 080–083; USER-AC-026–028; USER-DM-AC-011–015, 024–027; API-DEC-007

### 8.1 Atomic state transition (USER-SD-DEC-007)
**[Required]** At most one of several concurrent requests with the same token succeeds, and the session is never left in an undefined state (USER-FR-083).

**[Approved] (USER-SD-DEC-007):** an atomic conditional update **or an equivalently safe transaction strategy**. The two candidates below are both acceptable if they meet the required outcome; the choice is made at physical design.

| Option | Behavior | Notes |
|---|---|---|
| **Atomic conditional update** | A single statement marks the token consumed only if it is currently unconsumed, unexpired, and its session is unrevoked; the number of rows changed tells the request whether it won | Only one request can win by construction; no explicit lock held across application logic |
| Row locking in a transaction | The token (and session) row is locked, checked, then updated | Also correct; holds a lock for the duration of the transaction; more room for mistakes if the check and the update are separated |

**Approved:** consume the old token and create its successor **in the same transaction**, using the atomic conditional update (or an equivalent), so no committed state exists in which the old token is consumed and no successor was created (data-model §9, USER-DM-AC-024). Concurrent requests cannot consume the same token twice.

**Transaction outcomes (applies if rotation is approved).**
| Outcome | Meaning | Required handling |
|---|---|---|
| **Confirmed commit** | The database confirmed the transaction committed | Old token is consumed; return the successor |
| **Confirmed rollback** | The database confirmed the transaction did **not** commit (for example, an error reported before commit, or an acknowledged rollback) | The old token remains unconsumed and still valid; no successor exists; the client may present the old token again |
| **Ambiguous outcome** | The result is unknown (for example, the connection was lost or timed out during or after commit) | **Not** treated as a confirmed rollback. The service must not assert that the old token is still valid, must not tell the client a retry is safe, and must not return tokens it cannot show were persisted. Return the generic dependency failure. The true state is discovered only when the old token is next presented: if it is consumed it is rejected as consumed (and the reuse policy of USER-SD-DEC-008 applies), if it is not consumed it is still valid. |

The system can claim that the old token is still valid **only** where it can establish that the transaction did not commit. An ambiguous outcome can therefore produce the same sign-out as a lost response (§6.4); this is part of the trade-off the owner must accept (USER-SD-DEC-009).

**Loser's response (API-DEC-007).** **[Approved]** the losing request receives the same generic `AUTH_REFRESH_TOKEN_INVALID` as any invalid token (USER-FR-076). The alternative of a distinct retry-safe response would disclose token state and is rejected under USER-FR-076/082.

### 8.2 Reuse detection and session response (USER-SD-DEC-008)
**[Required]** Rejection and a recorded security event on presentation of an invalidated token (USER-FR-082).

**[Approved] (USER-SD-DEC-008):** presenting a consumed token rejects the request, records the security event, **and revokes the entire affected session**. USER-AC-026 / USER-DM-AC-026 therefore apply.

| Option | Benefit | Cost |
|---|---|---|
| Reject and record only | A benign race or double-submit does not end the session | If the token was stolen, the thief's successor chain stays valid, and the genuine user is the one who gets rejected |
| **Reject, record, and revoke the whole session** | Reuse is a strong sign of theft or a copied token; revoking the session cuts off both parties until re-sign-in | A benign concurrent refresh (for example two tabs refreshing at once) looks identical to reuse and would sign the user out |

**Two separate policies, both now approved.**
1. *Reuse detection* — a presented consumed or revoked token is rejected and recorded as a security event. **[Required]** (USER-FR-082).
2. *Revocation of the entire session on reuse* — approved by the project owner (USER-SD-DEC-008). The requirements do not mandate it, but the approved decision applies.

**Documented consequence:** the system cannot tell a thief's reuse from a legitimate lost response, tab race, or ambiguous outcome without a grace period, so those legitimate cases also sign the user out. The owner accepted this (USER-SD-DEC-009). The session-revocation write is a security state change, independent of the event record (§13.2): it is attempted regardless of whether the event write succeeds.

### 8.3 Absolute session lifetime (USER-SD-DEC-010)
**[Approved]** With rotation, each refresh issues a token with a fresh 7-day life, so an active session could otherwise live indefinitely; the approved 90-day absolute cap prevents that (data-model USER-DM-DEC-005).
- **[Approved] (USER-SD-DEC-010):** the absolute session lifetime is **90 days** (project owner, Step 5, 2026-10-24). A session must not exceed 90 days even when it is actively refreshed.
- **How absolute session expiry interacts with refresh-token expiry:**
  1. The **absolute session expiry** is fixed at session creation (sign-in): creation time plus 90 days. It never changes afterwards, and **refreshing never extends it**.
  2. Each refresh token has its own **refresh-token expiry**: issue time plus the refresh-token lifetime (7 days by default, configurable; USER-FR-071, 072). A successor token's expiry is the **earlier** of (now plus the refresh-token lifetime) and the session's absolute expiry, so a token never outlives its session.
  3. A refresh attempted at or after the absolute session expiry is rejected with the generic refresh error and the session ends. The customer signs in with Google again.
  4. **Inactivity:** a session whose newest token goes unused for the refresh-token lifetime (7 days by default) ends by token expiry, well before 90 days. No separate inactivity timer is introduced. The absolute cap bounds an *actively used* session at 90 days in total.
  5. **Access tokens** (15 minutes by default) are independent of session expiry. One issued shortly before the absolute expiry can remain valid until its own expiry, up to 15 minutes later, the same residual window as after logout (§9).
  6. **[Proposed]** The 90-day value is read from configuration like the other lifetimes (USER-FR-072), and the service refuses a configured value above the approved 90 days at startup unless a larger value is separately approved.
  7. Under a cookie transport (if approved), the cookie lifetime never exceeds the token's own expiry.

**Terminology used across the specifications**

| Term | Meaning |
|---|---|
| Session (refresh session) | The record created at one sign-in; the unit of revocation; the family of that sign-in's refresh tokens |
| Absolute session expiry | Session creation + 90 days; fixed; never extended by refresh |
| Refresh-token expiry | Per-token expiry, 7 days by default, never later than the absolute session expiry |
| Access-token expiry | 15 minutes by default; stateless; independent of the session |
| Inactivity window | The refresh-token lifetime; there is no separate inactivity timer |
- **[Proposed]** Store the computed expiry at issue time so that later configuration changes do not silently alter tokens already issued.
- Trade-off: a cap forces periodic re-authentication but bounds the damage of a stolen, kept-alive session.

### 8.4 Multiple sessions (USER-SD-DEC-011)
**[Approved] (USER-SD-DEC-011)** The data model does not limit a customer to one session (USER-DM-AC-008).
- **Approved:** multiple active sessions per customer; logout acts on the current session only (api-contracts §6.3). No logout-all endpoint in the first release; adding one would be a new endpoint requiring its own approval. This resolves USER-Q-003.
- Trade-off: more sessions mean more tokens to protect and a larger set to review if an account is compromised.

## 9. Logout and Session Revocation
**Decision:** USER-SD-DEC-012 · **Traces:** USER-FR-027, 073, 074, 084, 085; USER-SEC-012; USER-AC-017, 018; API-DEC-008; USER-DM-AC-016, 020

**[Required]**
- Logout revokes the applicable refresh token or session; a revoked token cannot obtain new access tokens (USER-FR-073).
- Logout does not necessarily invalidate already-issued access tokens; they may remain valid until natural expiry; protected APIs continue to validate every access token (USER-FR-074, 084). The security design documents this (USER-SEC-012) — **it is documented here (§5 and this section)**.
- Immediate invalidation of issued access tokens requires a separate approved mechanism (USER-FR-085). None is part of this specification.
- A revocation that cannot be persisted is never reported as success (api-contracts §6.3; USER-DM-AC-016).
- A missing refresh token identifies no session; nothing is revoked (api-contracts §6.3).

**[Approved] (USER-SD-DEC-012)** Logout revokes the **session** (not just the single presented token), so every token in the session is invalid. Revocation is permanent (data-model §6.3).

**Residual window.** For up to the configured access-token lifetime (15 minutes by default) after logout, a copied access token still works. Options if that is unacceptable, **none proposed or approved here** (USER-FR-085): a short server-side list of revoked access-token identifiers (adds per-request state and a token identifier claim), a shorter lifetime, or checking the session on every request (removes statelessness). Each changes an approved behavior and needs its own decision.

**Residual-risk statement (preserved).** Revoking a refresh session does not automatically invalidate an already-issued, self-contained JWT access token. Unless an additional revocation mechanism is separately approved (USER-SD-DEC-022, deferral approved), the access token may remain valid until it expires, subject to normal validation rules (signature, expiry, claims). Protected APIs continue to validate access tokens on every request.

## 10. Token Transport, Cookies, CORS, and CSRF
**Decisions:** USER-SD-DEC-013–016 · **Traces:** USER-Q-004; API-DEC-002, 003; USER-SEC-006, 008, 010; api-contracts §5

### 10.1 Transport and client storage (USER-SD-DEC-013, Approved (conditional))
**Approved direction:** a `Secure`, `HttpOnly` refresh-token cookie, and the access token held in memory on the frontend. **No production profile is selected** until deployment feasibility is confirmed (§10.3, §10.7).

| Credential | Transmission | Client storage |
|---|---|---|
| Google credential | Request body over HTTPS (`POST /auth/google`), with the binder (§4.4) | Not stored |
| Access token | Response body; `Authorization: Bearer` on protected requests | Page memory only; not `localStorage` or `sessionStorage`; lost on reload, recovered by one refresh call |
| Refresh token | `Secure`, `HttpOnly` cookie | Held by the browser; not readable by page scripts |
| CSRF token | Sign-in response body (§10.5) | Browser storage shared by the tabs of the frontend origin; not an authentication credential on its own |

**[Required]** Credentials and tokens are sent only over encrypted transport in production (USER-SEC-010), and must not appear in URLs, logs, or error responses (USER-FR-028, USER-SEC-006, 008). Logging of `Authorization`, `Cookie` and `X-CSRF-Protection` headers and of request bodies for the sign-in endpoint MUST be redacted.

**Trade-offs.** Keeping the refresh token out of script-readable storage limits what script injection can steal. The cost is that refresh and logout become browser-automatic cookie requests, hence the CSRF controls (§10.5), and that **a browser that blocks the cookie cannot refresh silently** (§10.6). A refresh token in the response body avoids both but leaves the long-lived credential readable by page scripts; it is not recommended.

### 10.2 Deployment facts and same-site evaluation
| # | Fact | Status |
|---|---|---|
| F1 | Frontend technology and hosting | **Confirmed:** React on Render. The exact Render service type is **not** confirmed. |
| F2 | Origins | **Confirmed:** frontend `https://varsha-homemade.onrender.com`; API `https://varsha-customer-service.onrender.com` |
| F3 | Origin relationship | Different origins, both HTTPS, both under `onrender.com`. **Cross-site** (below). |
| F4 | Gateway and proxy topology | **Confirmed:** no Apigee or other gateway; the API is directly reachable; no separate BFF. Render's own platform proxy remains; its trusted-proxy configuration is still to be established. |
| F5 | Instances | **Single instance** |
| F6 | Clients | **Browser only**; no native mobile |
| F7 | Custom domains | **Not confirmed.** Ownership is unconfirmed. |

**Same-site evaluation.** Different origins do not by themselves decide whether two hosts are the same site; the browser compares the *registrable domain*, which is one label plus a public suffix. The Public Suffix List includes `onrender.com` as a Render-submitted private entry (checked against the published list on 2026-10-24; re-check before relying on it). Therefore:
- The frontend's site is `varsha-homemade.onrender.com` and the API's site is `varsha-customer-service.onrender.com`. They are **different sites: the deployment is cross-site**, not merely cross-origin.
- A request from the React app to the API is a cross-site request. A cookie set by the API is a third-party cookie in that context.
- `SameSite=Lax` and `Strict` cookies from the API are **not sent** on cross-site `fetch` requests.
- A cookie cannot be shared across the two hosts with `Domain=onrender.com`, because browsers refuse cookies scoped to a public suffix. Cookies are host-only.
- This is evaluated for the current default hostnames only. Custom domains under one registrable domain would change the result, and none are assumed.

**Cookie transport, SameSite policy, CORS, and CSRF protection must be designed together,** and cookie settings must reflect the verified deployment architecture.

**Constraints that apply whatever the deployment:**
- `SameSite=None` requires `Secure`, and does **not** itself provide CSRF protection.
- A `__Host-` prefixed cookie requires `Secure`, `Path=/`, and no `Domain` attribute. A restricted cookie `Path` is therefore incompatible with the prefix.
- CORS controls whether a browser lets a page read a cross-origin response. It is **not** a substitute for CSRF protection, because a cross-site form can still send credentialed requests.

### 10.3 Two alternative deployment profiles (USER-SD-DEC-013, 014)
**These are alternative designs, not simultaneously active production configurations. Neither is production-ready until its prerequisites are verified, and no production profile is selected.** Nothing here changes any deployment configuration.

**Cookie attributes and their purpose**
| Attribute | Purpose |
|---|---|
| `Secure` | The browser sends the cookie only over HTTPS |
| `HttpOnly` | Page scripts cannot read the cookie |
| `SameSite` | Controls whether the cookie travels on cross-site requests (`Lax`, `Strict` keep it home; `None` allows it and requires `Secure`) |
| `Path` | Scope of requests that carry the cookie; `/` sends it with every request to that host |
| `Domain` | Omitted: a host-only cookie. Setting it widens sharing across subdomains |
| `__Host-` prefix | Optional hardening: needs `Secure`, `Path=/`, no `Domain`; stops sibling subdomains overwriting it |
| `Max-Age` | Never later than the token's own expiry, which is capped at the 90-day absolute session expiry |

**Profile X: the current Render origins (cross-site)**
| Aspect | Design |
|---|---|
| Topology | Frontend `https://varsha-homemade.onrender.com` calls API `https://varsha-customer-service.onrender.com` directly. Cross-site. |
| Cookie | `Secure; HttpOnly; SameSite=None`, host-only. `Path=/` allows the `__Host-` prefix; a restricted path is also possible. Optionally `Partitioned` (browser support varies; verify). |
| CORS | Credentialed: exact allowed origin `https://varsha-homemade.onrender.com`, credentials allowed, allowed headers `Authorization`, `Content-Type`, `X-CSRF-Protection`, `Vary: Origin` |
| CSRF | Origin validation, the custom header, **and the session CSRF token** (§10.5). `SameSite` gives no protection here. |
| Browser compatibility | **Risk:** browsers that block or partition third-party cookies may not store or send the refresh cookie, so silent refresh fails. **A CSRF token does not fix this.** Users may need to sign in again (§10.6). Behavior differs by browser and changes over time; test the target browsers. |
| Deployment dependencies | None beyond testing |
| Frontend changes | Credentials on refresh and logout only; the CSRF header; the nonce step with the binder; the sign-in fallback (§10.6) |
| Trade-offs | No deployment change and fastest; fragile; degraded experience in cookie-blocking browsers |

**Profile Y: a future same-site or same-origin deployment**
| Aspect | Y1: custom domains | Y2: same-origin proxy |
|---|---|---|
| Topology | Frontend and API as subdomains of a domain the owner controls (same site, different origins) | The frontend host forwards API paths to the API (one origin) |
| Cookie | `Secure; HttpOnly; SameSite=Lax`, host-only; `Path=/` allows `__Host-` | Cookie lives on the frontend host: a restricted `Path` for the API path is advisable, which rules out `__Host-`; or `Path=/` with `__Host-` |
| CORS | Credentialed, exact new frontend origin | None needed |
| CSRF | Origin check, the custom header, the session CSRF token (same contract as X) | Same |
| Browser compatibility | The cookie is not third-party; much more reliable | Same |
| Deployment dependencies | Domain ownership (**unconfirmed**), DNS, Render custom domains and TLS (**unverified**), updated Google authorized origin and CORS allow-list | The frontend must be a service type that supports rewrites (**unconfirmed**), and the rewrite must forward `POST`, cookies and headers correctly (**not documented**) |
| Frontend changes | Update the configured API origin; otherwise the same | API base path becomes relative |
| Trade-offs | Cost and DNS work; best long-term | The API stays directly reachable and must still enforce Origin, CSRF and CORS; the proxy hides client addresses, so per-address rate limiting is ineffective unless a trusted forwarded header is configured; one more hop |

### 10.4 CORS (USER-SD-DEC-015, Approved)
**Approved:**
- The **sole allowed origin** for direct browser-to-API calls is the frontend origin `https://varsha-homemade.onrender.com`.
- No wildcard origins; never a reflected arbitrary origin. The public API origin is not the frontend's allowed origin.
- **Credentialed requests only if the selected cookie strategy requires them** (Profile X and Y1 do; Y2 needs no CORS).
- Only the required methods (`GET`, `POST`, `OPTIONS`) and headers (`Authorization`, `Content-Type`, `X-CSRF-Protection`).
- Origin values come from configuration. **Revisit the allow-list if custom domains are configured.**

**Limitation:** CORS controls which pages may read responses. It is not CSRF protection (§10.5).

### 10.5 CSRF protection (USER-SD-DEC-016, 028, Approved)
**Approved baseline** for browser requests that rely on cookie authentication:
- Custom header `X-CSRF-Protection`.
- Rejected request: **HTTP 403 with error code `REQUEST_REJECTED`**, one generic response for every cause, with no token state revealed.
- **Origin validation:** the request `Origin` must exactly equal the configured frontend origin.
- A **missing `Origin` is rejected** on refresh and logout.
- A **session-scoped CSRF token**, in both profiles.
- Origin validation or a custom header alone is **not** treated as a complete strategy.

**Exactly which endpoints**
| Endpoint | Cookie-authenticated | Origin check | `X-CSRF-Protection` with the CSRF token | Missing `Origin` |
|---|---|---|---|---|
| `POST /auth/refresh` | Yes | Required | Required | Reject |
| `POST /auth/logout` | Yes | Required | Required | Reject |
| `POST /auth/google` | No (credential in the body) | Not required | Not required | Not applicable |
| `POST /auth/google/nonce` | No | Not required | Not required | Not applicable |
| `GET /users/me` | No (bearer token) | Not required | Not required | Not applicable |
Any future endpoint that relies on the refresh cookie adopts the same controls.

**How the token reaches the frontend and is supplied**
1. At sign-in the server creates a **random CSRF token** for the new session and stores only its **digest** on the session record.
2. The raw value is returned **once**, in the sign-in response body (`csrfToken`).
3. The frontend keeps it in **browser storage shared by the tabs of the frontend origin**, so a reload, a new tab or a restored tab still has it. It is not an authentication credential by itself: without the `HttpOnly` cookie it authorizes nothing, and script injection could issue requests anyway.
4. The frontend sends it in `X-CSRF-Protection` on every refresh and logout.
5. The token is **session-scoped and constant for the session**; it is not rotated on refresh, which keeps multiple tabs consistent. Logout or session end clears it.
6. A cookie-only "double-submit" scheme cannot work cross-site, because a page cannot read the API's cookie, which is why the token is delivered in the response body.
7. If the frontend has lost the token while the cookie remains, refresh is rejected and the customer signs in again, which creates a new session.

**Order of checks, all before any token is consumed:** Origin present and exact → header present → find the session from the refresh cookie (read only) → compare the token's digest → then the atomic consume (§8.1). A request failing a CSRF check never consumes or revokes a token, and the client does not retry it. If the cookie is absent, the response is the generic `401 AUTH_REFRESH_TOKEN_INVALID`.

**Limitations**
| Control | Does not protect against |
|---|---|
| Origin validation | Requests lacking `Origin` unless rejected (it is, on these endpoints); an allowed origin that is itself compromised |
| Custom header and CSRF token | Script injection on the frontend, which can read the token and set the header |
| `SameSite=Lax` (Profile Y) | Requests from other subdomains of the same site |
| All of them | A stolen refresh token used directly (covered by rotation and reuse detection, §8) |
A strict content-security policy on the frontend is recommended.

### 10.6 Profile X third-party-cookie risk and the sign-in fallback (USER-SD-DEC-029, Approved)
**The risk.** Under Profile X the refresh cookie is a third-party cookie. A browser that blocks or partitions third-party cookies may refuse to store it, or not send it on the frontend's refresh call. Then silent refresh fails. **The CSRF token does not solve this; it only guards the request.**

**The design must account for users needing to sign in again:**
- The sign-in response still returns the access token in the body, so the customer is signed in for the access-token lifetime (15 minutes by default). After that, refresh fails with `401 AUTH_REFRESH_TOKEN_INVALID` and the customer must sign in again. In a cookie-blocking browser this can be as often as every 15 minutes and on every reload.
- The frontend treats a failed refresh as the end of the session and shows the sign-in prompt. It does **not** retry the refresh and does not loop (decision 023).
- **Session accumulation:** each such sign-in creates a session whose refresh cookie was never stored. Those sessions are never used again and sit until they expire. Sign-in and nonce rate limits (§11.1) and the retention decision (020) matter for this.
- The frontend may remember in browser storage that refresh failed immediately after a sign-in, and then skip the silent refresh attempt on later loads, to avoid repeated pointless requests and to show a clear message. This is a frontend-contract detail for review.
- Profile Y removes the cause. **A same-site or same-origin deployment is the real remedy.**
- Behavior differs by browser and changes over time, so the supported browsers must be tested before Profile X is used in production.

### 10.7 Deployment alternatives, prerequisites and the same-origin proxy assessment
Only a **feasibility assessment** of the proxy is authorized, not a deployment change. Nothing has been tested and no configuration was changed.

**Mapping to earlier labels:** Profile X is the earlier option S1; Y1 is S2; Y2 is S3; storing the refresh token in the response body is S4 (not recommended).

**Y1 prerequisites:** domain ownership or purchase; DNS records for both subdomains; Render custom domains and TLS for both services; the new frontend origin added to the Google authorized origins and the CORS allow-list; issuer and audience configuration; a re-run of the same-site evaluation for the final hostnames.

**Y2 proposed topology:** the browser requests `/api/...` on the frontend host; the host rewrites to the API origin; the browser sees a single origin. Security implications: the API remains reachable directly and must still enforce Origin, CSRF and CORS for direct calls; the proxy hides client addresses so per-address rate limits need a trusted forwarded header or per-session limits; the cookie is first-party but `Path=/` would also send it with static-asset requests to the frontend host; the proxy is a new trust point.

**Evidence (Render documentation, checked 2026-10-24):** rewrites exist and serve the destination content without redirecting the browser, and the rule syntax says the destination can be "a path or a full, publicly accessible URL". The documentation does **not** say whether `POST` requests, cookies (including `Set-Cookie`) or headers are forwarded, and it does not confirm proxying to another service. The frontend's service type is **not** confirmed by any configuration. **Proxy feasibility is therefore unverified.**

**Proposed assessment (requires your authorization to execute):** confirm the frontend service type; confirm in Render's documentation or a test that `POST`, request and response headers, `Origin` and `Set-Cookie` pass through; confirm the API sees a usable client address; confirm the cookie behaves as first-party. Any test must not change production configuration.

## 11. Rate Limiting and Authentication Abuse Protection
**Decision:** USER-SD-DEC-017 · **Traces:** USER-SEC-009; USER-FR-078; USER-DEC-013; USER-Q-002; USER-AC-014

**[Required]** The service MUST include controls to limit repeated failed or abusive authentication attempts, including sign-in and refresh requests; controls SHOULD be configurable; error responses MUST NOT expose sensitive account information (USER-SEC-009).

**[Unresolved]** thresholds, scope, and state store.

| Question | Options | Recommendation |
|---|---|---|
| Scope | Per client address; per session or token on refresh; per customer after identity is validated | Per client address on all four endpoints, plus per session on refresh. Per-customer limits on the sign-in endpoint are only possible after identity validation, and the Google credential already gates abuse |
| Thresholds and windows | Numbers | **Not proposed.** No values are invented; the owner or operations sets them as configuration |
| State store | In-process memory; PostgreSQL; a separate store such as Redis | In-process memory is sufficient for a single instance but is not shared across instances and resets on restart. A shared store needs either PostgreSQL (extra write load on the sign-in path) or a new technology, which is not approved. Choice depends on the unknown instance count |
| Client address | Direct connection address, or forwarded address when behind a proxy | Depends on the unknown proxy topology; trusting forwarded headers incorrectly lets attackers spoof their address and bypass limits |
| Response | `429` with `RATE_LIMITED` and a retry hint | As in api-contracts §4; generic, no account detail |

**Trade-offs.** Per-address limits can affect users sharing an address (offices, mobile carrier networks); per-session limits do not help against unauthenticated floods. Limits do not replace Google's own protections for the sign-in itself.

**Single-instance implications (F5).** The current plan is one instance. In-process rate-limit counters would work for one process but are lost on every restart or redeploy and do not survive scaling to several instances; they are not durable. Durable state (sessions, tokens, security events) must live in PostgreSQL, never in memory, and the atomic refresh transition (§8.1) is still required, because a single Node.js process handles many requests concurrently. Nothing here assumes in-memory state is suitable for durable session management.

The design must be reassessed if the service later runs more than one instance (in-process counters and any in-memory coordination would not be shared across instances).

**Client-address identification.** Forwarded client-address headers MUST NOT be trusted until the trusted-proxy configuration is explicitly established (F4). Abuse and refresh-token limits rely on a correct client address, and a wrong trust setting lets attackers pick their own.

**Dependence on topology (USER-SD-DEC-017, Open).** The rate-limit implementation depends on the confirmed deployment topology, the number of application instances, and the trusted-proxy configuration (F4, F5). With several instances, in-process counters are kept separately per instance, so the effective limit multiplies; with a proxy, the service sees the proxy's address unless a trusted forwarding header is configured, and a misconfiguration either lets attackers choose their own address or makes every client share one limit. No implementation choice is made until these facts are known, and the behavior when the limit store is unavailable (fail open or fail closed) is part of the decision (USER-SD-DEC-024).

Rate-limit state is not part of the data model (data-model §1). If the owner selects a database-backed store, that is a data-model change needing its own approval.

### 11.1 Proposed rate-limit targets and strategy (USER-SD-DEC-017, Open)
Thresholds are **deferred** until operational requirements are reviewed. No numeric value is proposed. The structure below is for review.

| Target | Proposed scope | Purpose |
|---|---|---|
| Google sign-in (`POST /auth/google`) | Per client address, and failed attempts per client | Authentication abuse and credential stuffing |
| Nonce issuance (only if the nonce design, §4.4, is approved) | Per client address, with a cap on outstanding nonces per client | The operation is unauthenticated and creates state |
| Refresh (`POST /auth/refresh`) | Per client address and per session | Refresh-token abuse; a flood of invalid tokens |
| Logout (`POST /auth/logout`) | Per client address | Forced-logout abuse |
| Identity retrieval (`GET /users/me`) | Per client address; optionally per customer | General abuse |

**Sign-in volume and orphaned sessions (Profile X).** When third-party cookies are blocked, every re-sign-in creates a session whose refresh cookie is never stored. Sign-in and nonce limits and session retention matter more in that case (§10.6).

**Storage strategy.** *Proposed:* in-process counters are acceptable for the first release because a single instance is planned, with the known consequences that limits reset on every restart or redeploy and are not shared across instances. A PostgreSQL-backed store is the alternative and adds write load. Reassess the design if the service later runs more than one instance.

**Client-IP identification and trusted-proxy assumptions.** No Apigee or other gateway is in the request path (confirmed), and the API is directly reachable, but Render's platform proxy sits in front of the service (how it forwards the client address, and how many hops to trust, is to be verified in Render's documentation). **Forwarded client-address headers MUST NOT be trusted without an explicit trusted-proxy configuration.** If none is established, the connection address would be the proxy's address and every client would share one per-address limit. Because the API is directly reachable, a client can also reach it without any intermediary, so the trusted-proxy setting must not allow a forged forwarded header to be accepted.

**Behavior when the rate-limit store is unavailable.** Not applicable to in-process counters. For a database-backed store, *proposed:* fail closed for sign-in, nonce issuance and refresh (those operations already depend on the database), never silently disable limits. This is open.

## 12. Secret Management and Signing-Key Lifecycle
**Decision:** USER-SD-DEC-018 · **Traces:** USER-SEC-003, 004, 008; USER-FR-072

**[Required]** Secrets (signing keys, any Google client secret) are supplied through configuration, are never committed to the repository, never exposed to clients (USER-SEC-003), and follow least privilege (USER-SEC-004). Configuration uses environment variables.

**[Proposed] practices**
- Required configuration (signing key, Google client identifier, token lifetimes, allowed origins, rate-limit settings) is validated at startup; the service refuses to start if a required secret is missing or below a minimum strength, rather than falling back to a default.
- Different secrets per environment; secrets never reused between development and production; never logged, including at startup.
- Environment files containing secrets are excluded from version control; no secret appears in example files.
- The Google client identifier is not secret; a client secret is needed only if the authorization-code flow is chosen (API-DEC-001).

**[Proposed] signing-key lifecycle**
- Each key carries an identifier. Rotation introduces a new key for signing while the previous key remains valid for verification only until all tokens signed with it have expired. Given the 15-minute lifetime, the overlap needed is about one access-token lifetime plus clock tolerance.
- **Compromise response:** replace the key immediately with no overlap. All outstanding access tokens become invalid and users refresh to obtain new ones; refresh tokens are unaffected unless the keyed-digest option for refresh storage is used (§7), in which case rotating that key needs its own identifier scheme.
- Rotation frequency is not proposed (owner/operational decision).

**Approved approach (USER-SD-DEC-018):**
- Secrets are **server-side environment secrets**, provided through the hosting platform's secret-management mechanism, never in source control, configuration files in the repository, or frontend code.
- The Google client secret and the token-signing secret are never placed in React code or any browser-delivered configuration.
- **Startup validation:** the service refuses to start if a required secret is missing or too weak, if configured lifetimes exceed the approved maxima (access token 15 minutes, refresh token 7 days, absolute session 90 days), or if the allowed origins are not exact HTTPS origins or include a wildcard.
- **Key identifiers and rotation** where applicable (§5.1, below).
- **The previously exposed Google client secret is treated as compromised and rotated before implementation.**

**Operational ownership (an accountable owner must be assigned before production; no name is invented)**
| Responsibility | Named owner |
|---|---|
| Provision and hold secrets in the hosting platform | ______ |
| Rotate the token-signing secret or keys | ______ |
| Rotate the Google client secret | ______ |
| Respond to a suspected compromise | ______ |
| Authorize emergency revocation, including mass session revocation | ______ |
| Monitor and alert on secret-related failures | ______ |

How Render exposes secrets for this service (who can read them, whether secret files or environment values are used) is to be confirmed.

**Credential classification (values are never recorded in any specification).**
| Credential | What it is | Exposure | Recommendation |
|---|---|---|---|
| Supabase project URL | An identifier, not a secret | Shown | None |
| Supabase publishable key | Client-facing key for Supabase's HTTP APIs; safe only where row-level security is enforced; **unused** under the approved design | Shown | Not required for secrecy; not shipped to React or used |
| **Supabase secret key** | Elevated server key for Supabase's HTTP APIs that bypasses row-level security; **unused** under the approved design | **Exposed** | **Rotation recommended before production**, and revoke it if not needed. The owner has not decided (RISK-002). |
| Supabase JWKS URL | Public metadata for Supabase Auth tokens; **unused** (Supabase Auth is not used) | Shown | None |
| **Database role and connection string** | The credential Express needs to reach PostgreSQL. A different credential from the keys above. Not among the values shown. | No evidence of exposure | Create a fresh least-privilege role through the approved setup; never record it |
| **Google client secret** | OAuth client credential for the authorization-code exchange; **not needed by the backend** under the ID-token flow | **Exposed** | **Rotation recommended before production.** See RISK-001. |
| HS256 signing secret | To be generated; server-side only | Not yet created | Per §5.1 |

**RISK-001 (unresolved security risk; owner decision, Step 5b).** The previously exposed Google client secret **remains in use for now** by the owner's decision. **This is not a secure or approved production state.** Anyone holding it could impersonate the application in an authorization-code exchange. What currently depends on it is unknown. While it remains: it is never copied into specifications, logs, source control or React code; if the backend holds it, it is server-side only; and rotation before production is recommended. Gating criterion USER-SD-AC-026 is **NOT SATISFIED** while this risk is open.

**RISK-002 (Supabase secret key).** The Supabase secret key was exposed. The owner has not decided whether to rotate it. Rotation before production is recommended, and it is unused by the design.

**Accountable owner.** An accountable owner for secret management and emergency rotation must be assigned **before production**. No name is invented; the owner is asked to assign one.

**Render secret management (to confirm, USER-SD-DEC-018).** How Render stores and exposes environment values, who has access to them, and the operational ownership of rotation and compromise response are not confirmed here. Startup validation, signing-key identification, and rotation procedures are specified below; secrets stay out of configuration files, logs, and source control.

**Operational responsibilities (USER-SD-DEC-018).** The approach is approved (server-side environment secrets); the named owners below are **not yet assigned**:
- **Ownership:** who generates, holds, and has read access to each secret in each environment.
- **Rotation:** the schedule and who performs it, and how the overlap window for the access-token signing key is managed.
- **Compromise response** (who may authorize, and the procedure):
  - *Access-token signing key exposed:* replace the key immediately with no overlap; all outstanding access tokens stop validating, and clients refresh. An attacker with the key could mint access tokens for any customer until it is replaced, so the owner decides whether to also review activity from the exposure window. Refresh sessions are not affected by this key.
  - *Refresh-token digest key exposed (only if a keyed digest is chosen):* rotate the key; the attacker could forge sessions during the exposure window, so the owner decides whether to revoke all sessions or those created in that window.
  - *Database credentials or contents exposed:* stored digests do not yield usable tokens (USER-SEC-011), but the owner decides whether to revoke all sessions and rotate credentials.
  - *Google client secret exposed (only if the authorization-code flow is chosen):* rotate it with Google.
- **Mass revocation of affected sessions or credentials:** no administrative role or interface exists (USER-DEC-012), so revoking many sessions would be an operational action outside the approved API. Whether such a procedure is acceptable, and who may run it, needs a decision and probably a data-model note.
- **Detection and notification:** how a suspected compromise is detected and who is told.

**[Unresolved]** Where production secrets are kept (plain environment variables, an operating-system level service, or a managed secret store) depends on the unknown hosting environment, and Docker is not used. Plain environment variables are the approved configuration mechanism; their handling on the host (file permissions, who can read process environments) is an operational control to be defined at deployment.

## 13. Security Events, Retention Dependencies, and Privacy
**Decisions:** USER-SD-DEC-019, 020 · **Traces:** USER-FR-011, 065, 066, 082; USER-SEC-008; USER-DM-DEC-006, 007, 012; USER-PRE-001; data-model §5.5, §8.4

**[Required]**
- Record: presentation of an invalidated refresh token (USER-FR-082); identity conflict (USER-FR-011, 066); unresolvable customer identity (USER-FR-065).
- Events never contain raw tokens, token digests, credentials, secrets, or unnecessary personal data (USER-FR-028, USER-SEC-008; USER-DM-AC-018).
- Retaining an event does not require retaining the token record indefinitely (data-model §8.4; USER-DM-AC-019).

**[Proposed] event types** (list is open, USER-DM-DEC-006): refresh-token reuse of an invalidated token; session revoked due to reuse (if adopted); identity conflict; unresolvable identity; repeated-failure/rate-limit trigger (optional, to support USER-SEC-009 review).

**[Approved: Step 5c] IP logging.** Ordinary logging of client IP addresses is **disabled by default**. It may be enabled only for a documented security need, and the justification must first state the purpose (for example rate-limit abuse analysis), who may read the logs, how long they are kept, and the privacy implications. None is approved. **[Proposed] event content:** event type, time, customer identifier if known, session or token-record identifier if known, request identifier. No client network address and no user-agent string is recorded by default.

**[Proposed] storage:** database records for the required events, so they survive log rotation and can be reviewed (USER-FR-011 "recorded for review"), plus redacted structured application logging for operational monitoring. **[Unresolved]** (USER-DM-DEC-006).

**Retention dependency (recognition limit).** The ability to identify a presented token as *previously consumed* (rather than unknown) depends on keeping its consumed record (data-model §8.4). **[Proposed]** retain consumed-token records at least until the token's own expiry, because after that the token is rejected as expired in any case. If records are removed earlier, reuse can only be rejected, not recognized, and the requirement to record it cannot be fully met (data-model §8.4 limitation). **[Unresolved]** the actual retention periods for sessions, consumed records, and events: owner/privacy decisions pending USER-PRE-001 and the Order retention analysis. No periods are proposed.

**Retention stays open (USER-SD-DEC-020, Open).** No retention period is chosen until privacy, legal, security, and operational requirements are reviewed. The review must consider that removing consumed or revoked token records can leave the system unable to recognize a presented token as previously consumed or revoked; it would then look the same as an unknown token (data-model §8.4).

### 13.1 Privacy and failure handling
**Privacy [Required]:** store only the internal identifier, Google account identifier, and necessary name and email (USER-FR-048, USER-DEC-010); no unnecessary Google profile data; access tokens carry no personal data; error responses and logs carry no unnecessary personal information (USER-SEC-006, 008). Deletion and recovery are deferred but privacy and retention obligations must be reviewed before launch (USER-DEC-011, USER-PRE-001).

**Failure handling [Required]:** any failure must end in a safe refusal, never a granted session (requirements §7; USER-FR-068). **[Proposed]** behavior per failure:

| Failure | Behavior | Traces |
|---|---|---|
| Google keys or service unavailable | Sign-in fails with the generic dependency error (503); no customer or session created; existing sessions unaffected | USER-FR-060; api-contracts §4 |
| Database unavailable | Sign-in, refresh, identity lookup, and logout fail with 503; no success is reported; logout never reports success if revocation was not persisted | api-contracts §6.3 |
| Failure while committing refresh | Consume and successor issuance commit together (§8.1). On a **confirmed** rollback the old token stays valid; on an **ambiguous** outcome it is not assumed valid and the generic dependency failure is returned | USER-FR-083; USER-DM-AC-024 |
| Security-event write fails | The suspicious or invalid request **remains rejected** regardless; event persistence failure never changes a rejection into an acceptance (§13.2) | USER-FR-082; USER-SD-DEC-024 |
| Missing or weak secrets at startup | Service refuses to start (§12) | USER-SEC-003 |
| Rate-limit store unavailable | Open: fail open (risks abuse) or fail closed (blocks sign-in). Recommendation: decide with the store choice; do not silently disable limits | **[Unresolved]** USER-SD-DEC-017 |

### 13.2 Security-event persistence and failure handling (USER-SD-DEC-019, 024)

**Mandatory events [Required]** (USER-FR-011, 065, 066, 082):
- presentation of an invalidated (consumed, revoked, or superseded) refresh token;
- identity conflict;
- customer identity that cannot be resolved.

**[Proposed]** Additional events (session revoked due to reuse, rate-limit trigger) are optional and open under USER-SD-DEC-019.

**Principle [Approved] (USER-SD-DEC-024; Step 5c):** mandatory security events are persisted, and **critical state changes are atomic with their required events**: a mandatory event is written in the same database transaction as the state change it describes, so a failed event write fails the operation (fail closed). A request that is already rejected stays rejected whether or not its event is stored. Events that describe no state change are operational logs unless made durable (§13.4). Revocation of a session on reuse commits together with its event.

**Detecting persistence failures [Approved: must be detectable and monitored; mechanism Open]:** the result of every event write is checked. A failure produces an operational error entry (event type, request identifier, failure class; no credentials) and increments a monitored failure indicator. Where a fallback sink such as the application log is available, the event is emitted there so it is not silently lost.

**Monitoring and alerting [Open]:** what monitoring and alerting exist, and who responds, is not established. The owner supplies this (USER-SD-DEC-024). Persistent event-write failure SHOULD raise an operational alert.

**Which event-write failures affect other operations [Approved with the principle]:** a failure to write a mandatory event fails the operation it belongs to (sign-in, rotation, logout, session revocation), because the event and the state change are one transaction. A failure to write an operational log does not change an outcome; it is detected and surfaced. If the database is unavailable altogether, those operations already fail with a dependency error.

**Content [Required]** (USER-FR-028; USER-SEC-008; USER-DM-AC-018): events contain only fixed fields taken from server-side state (event type, time, opaque customer, session, or token-record identifiers, request identifier). They never contain raw credentials, raw refresh tokens, token digests, secrets, or free text copied from requests. **[Proposed]** the set of fields is an allow-list, and event types use enumerated reason codes.

**Operational logs [Approved] (USER-SD-DEC-019):** logs are redacted. They never contain raw tokens, token digests, passwords, credentials, client secrets, or other secrets. A failure in logging or monitoring must not expose sensitive information. Mandatory security events are persisted; the exact event definitions, retention periods, and operational details remain open.

**Open:** event storage technology, retention periods, and detailed failure policies, pending operational and privacy input.

### 13.3 Retention requirements for review (USER-SD-DEC-020, Open — deferred)
Retention is deferred pending privacy, legal, and operational review. **No period is proposed.** The review must decide the following.

| Data | Why it is kept | Decision the review must make | Effect of deleting it |
|---|---|---|---|
| **Active and revoked sessions** | Active sessions authorize refresh. Revoked and expired sessions let the service reject their tokens and attribute events. | How long revoked and expired session records are kept | After deletion the session's tokens are unknown. They are still rejected, but a later presentation can no longer be tied to a known session |
| **Consumed refresh-token records** | Needed to recognize a presented token as previously consumed and so to detect reuse (USER-FR-082) and revoke the session (USER-SD-DEC-008) | How long consumed records are kept. At minimum they are needed until the token's own expiry, because after that the token is rejected as expired anyway | If a consumed record is deleted before the token's own expiry, the token looks **unknown**. It is still rejected, but it **cannot be recognized as reuse**, so no reuse event is recorded and the session is not revoked on that basis. The reuse protection is silently weakened unless the retention decision addresses this |
| **Security events and audit records** | Review and investigation of mandatory events | How long events are kept, and whether they are anonymized or removed | Loss of the audit trail for the affected period. Events must not depend on token records being kept (USER-DM-AC-019) |

Considerations for the review: privacy law and data-minimization duties, legal holds, investigation needs, the order-history retention analysis (USER-PRE-001), and storage cost. Retention of a nonce record (if §4.4 is approved) is short-lived and part of the same review.

### 13.4 Event-type enumeration (persistence principle approved; the enumeration is proposed)
**Decided (Step 5d, delegated):** mandatory security events are persisted and critical state changes are atomic with their required events (approved, Step 5c). The enumeration below is **adopted**: nonce replay and invalid nonce are **mandatory** (the nonce flow is retained); `AUTH_SIGN_IN_FAILED` and `AUTH_RATE_LIMIT_REJECTED` are **operational logs and metrics only**, because making them durable would let an attacker multiply database writes; they become durable and aggregated only if abuse investigation later requires it.
| Event type | Durable | Atomic with the state change | Metadata | Purpose |
|---|---|---|---|---|
| `AUTH_SIGN_IN_SUCCEEDED` | Yes | Yes (session creation) | customer id, session id, request id | Audit |
| `AUTH_REFRESH_ROTATED` | Yes | Yes (rotation) | session id, token-record id, request id | Audit |
| `AUTH_REFRESH_REUSE_DETECTED` | Yes | Yes (session revocation) | session id, request id, reason code | Theft detection (mandatory) |
| `AUTH_SESSION_REVOKED` | Yes | Yes | session id, reason code | Audit |
| `AUTH_LOGOUT` | Yes | Yes (revocation) | session id, request id | Audit |
| `AUTH_NONCE_REPLAY`, `AUTH_NONCE_INVALID` | Yes | n/a (request rejected) | request id, reason code; no nonce or binder | Replay detection |
| `AUTH_IDENTITY_CONFLICT`, `AUTH_IDENTITY_UNRESOLVED` | Yes | n/a | customer id if known, reason code | Approved mandatory events |
| `AUTH_SIGN_IN_FAILED` | Open | n/a | request id, generic reason code | Abuse signal |
| `AUTH_RATE_LIMIT_REJECTED` | Open (a flood would multiply database writes) | n/a | request id, endpoint, key class | Abuse signal |
| `AUTH_EVENT_PERSISTENCE_FAILED` | **No** (the database is the failing component) | n/a | event type, failure class, operational log and metric | Detect audit loss |
Never recorded: raw tokens, token digests, nonces, binders, credentials, secrets, or client addresses by default.

## 14. Cross-Cutting Controls and Dependencies

**[Required]** Input validation on all authentication requests (USER-SEC-005); generic, consistent error responses (USER-SEC-006, USER-FR-068); least privilege (USER-SEC-004); HTTPS in production (USER-SEC-010).

**[Proposed]** Security-relevant HTTP response headers and removal of server-identifying headers are a general hardening measure; whether a middleware dependency is used is part of USER-SD-DEC-021.

**Dependencies that may be needed (USER-SD-DEC-021) — none approved.** Requirements prohibit adding technologies or dependencies without approval, so each is a decision, not an assumption:

| Need | Possible approach | Without a new dependency |
|---|---|---|
| Google ID-token verification | An established verification library | Hand-written verification against Google's key set: possible but higher risk |
| Access-token signing and verification | An established JWT library | Hand-written: not recommended |
| Random tokens and digests | Node.js built-in cryptographic module | Available now |
| Rate limiting | An established rate-limiting middleware | Hand-written, in-process counters |
| CORS and cookie handling | Established middleware | Hand-written header handling: not recommended |

### 14.1 Dependency approval register (USER-SD-DEC-021)
**Nothing is installed.** Each candidate needs its own approval before it is added to the project. Capabilities and maintenance status below must be verified against current documentation and advisories at approval time. `express` itself is the approved framework; its package and version are approved with the first dependency.

| # | Package | Purpose | Security benefit | Alternatives (including built-in) | Maintenance and operational implications | Approval status |
|---|---|---|---|---|---|---|
| D1 | `google-auth-library` | Verify Google ID tokens on the backend (and exchange codes if the authorization-code flow is chosen) | Avoids hand-written signature, key-rotation and audience handling | `jose` with Google's published key set plus our own issuer, audience and verified-email checks; hand-written verification with Node's built-in crypto (higher risk) | Maintained by Google; carries its own dependencies; track advisories | Approved (delegated, 2026-10-25); install only on explicit instruction |
| D2 | `jose` | Sign and verify access-token JWTs; key identifiers; algorithm pinning | Strict algorithm and claim validation; supports rotation | `jsonwebtoken` (D3); hand-written with Node crypto (not recommended). Choose **one** of D2 or D3 | Widely used; check maintainer and advisories | Approved (delegated, 2026-10-25), chosen over `jsonwebtoken`; subject to checking module-format compatibility with the Node.js version, advisories and licence before install |
| D3 | `jsonwebtoken` | Same purpose as D2 | Same | `jose` (D2) | Widely used; check maintainer and advisories | Not selected (`jose` chosen); not approved |
| D4 | `express-rate-limit` | Rate limiting middleware (§11.1) | A tested limiter instead of hand-written counters | Hand-written in-process counters; a database-backed limiter | Default store is per-process memory; shared stores would be additional packages | Approved (delegated, 2026-10-25); install only on explicit instruction |
| D5 | `cors` | CORS headers with the approved allow-list (§10.4) | Correct preflight and credentialed handling | Hand-written header handling (not recommended) | Small; configuration must match the allow-list | Approved (delegated, 2026-10-25); install only on explicit instruction |
| D6 | `cookie-parser` | Read the refresh-token cookie (Express does not parse cookies by itself) | Avoids ad-hoc cookie parsing | Parse the `Cookie` header manually; setting cookies needs no extra package | Only needed under the cookie transport | Approved (delegated, 2026-10-25); install only on explicit instruction |
| D7 | `helmet` | Security response headers | Standard hardening headers | Set headers manually | Optional; review default headers for an API | Deferred: not needed for the first release; revisit later |
| D8 | `pg` | PostgreSQL client with parameterized queries and transactions (needed for the atomic refresh transition) | Parameterized queries guard against injection when used consistently | The `postgres` client; an ORM would introduce another technology and is not approved | Connection pooling and error handling matter for ambiguous outcomes (§8.1) | **Driver choice approved in principle** (a standard PostgreSQL driver, not the Supabase client library; decision 026). Package approved under delegation (2026-10-25); install only on explicit instruction |
| D9 | `dotenv` | Load a local `.env` in development | Keeps secrets out of code | Node.js built-in env-file support if the chosen Node.js version provides it (to verify) | Development only; production uses the platform's secrets | Deferred: production uses platform secrets and local development may use Node.js built-in env-file support (version to confirm) |
| D10 | `express` | The approved web framework package | Track advisories | None (approved framework) | Mature | **Approved (delegated, 2026-10-25)**; install only on explicit instruction |
| D11 | `node-pg-migrate` (development) | Version-controlled PostgreSQL schema changes as ordered migration files, run **only by the owner role** from a trusted machine, never by the Render runtime | Ordering and a tracking table; keeps the owner credential off the runtime | Plain SQL files applied by hand (no dependency, no tracking) | Verify maintenance and advisories | **Approved (delegated, 2026-10-25)**; install only on explicit instruction |
| D12 | `pino` | Structured logging with redaction | Helps redaction, but an **allow-list logging rule** applies regardless | A small allow-list logger over the console | Optional | **Deferred** |
| — | `@supabase/supabase-js` (Supabase client library) | Would access Supabase's HTTP APIs | — | Not used: the database is reached with a standard PostgreSQL driver | Would introduce the Data API surface | **Not approved; explicitly not to be used** (decision 026) |

## 15. Conflicts and Tensions Between Approved Requirements

| # | Tension | Handling in this document |
|---|---|---|
| 1 | USER-FR-083 (at most one successful use of a token) versus a reusable non-rotating design | Non-rotating design is judged to conflict; rotation recommended (§6.2). Flagged for owner approval; no requirement changed. |
| 2 | USER-FR-081 (consumed tokens never reusable) versus lost-response recovery | Accepted as a consequence; no grace period (§6.4). Relaxing it needs a requirements change. |
| 3 | USER-FR-082 (record reuse) versus retention of records needed to recognize reuse | Retention dependency documented (§13); periods unresolved. |
| 4 | USER-FR-074 / USER-DEC-008 (access tokens survive logout) versus the desire for immediate revocation | Accepted per approved decision; alternatives listed but not proposed (§9). |
| 5 | USER-FR-064 (expiry MAY be distinguishable) versus USER-SEC-006 (reveal little) | Distinguishing expiry is low-risk and permitted; left as API-DEC-005. |
| 6 | Reuse-triggered session revocation (§8.2) versus benign concurrent refresh | Flagged trade-off; mitigated by client single-flight. |
| 7 | Cookie transport for refresh tokens versus unknown client types | Recommendation conditional on browser-only clients (§10.1). |
| 8 | Earlier wording "the client may retry safely" after a failed refresh commit versus ambiguous database outcomes | Corrected: only a confirmed rollback leaves the old token valid (§8.1). |
| 9 | Approved API inventory (four endpoints) versus a server-issued nonce endpoint or an authorization-code callback endpoint | Neither is added. Either would need an API-contract amendment approved by the owner (§4.2). |
| 10 | Approved data model (no nonce, rate-limit, or key-identifier state) versus options that need such state | Flagged as data-model follow-ups, not added (§4.2, §7, §11). |
| 11 | No administrative role (USER-DEC-012) versus mass session revocation after a compromise | Flagged as an operational decision (§12); no admin interface is proposed. |
| 12 | Reuse-triggered revocation versus legitimate lost responses, tab races, and ambiguous outcomes | A grace period would fix it but contradicts USER-FR-081; left to the owner (§6.4, §8.2). |
| 13 | Approved reuse-triggered session revocation (USER-SD-DEC-008) with approved acceptance of resulting sign-outs (USER-SD-DEC-009) | Consistent. Accepted trade-off; documented in §6.4 and §8.2. |
| 14 | Confirmed cross-site deployment (`https://varsha-homemade.onrender.com` to `https://varsha-customer-service.onrender.com`, both under the public suffix `onrender.com`) versus browser third-party-cookie restrictions on a `SameSite=None` refresh cookie | Unresolved. Options S1 to S4 documented (§10.6); none decided. |
| 15 | Single-instance in-memory rate-limit counters versus restarts and future scaling | Flagged (§11); thresholds and storage deferred. |
| 16 | Recommended server-issued nonce (USER-SD-DEC-001) versus the approved four-endpoint API inventory and the approved data model | Not added. The required changes are documented for approval (api-contracts §10.4, data-model §13.6). |
| 17 | Approved `Path=/` cookie baseline versus least exposure | The cookie reaches every API request; the service accepts it only on refresh and logout and redacts it in logs (§10.3). A restricted path would be incompatible with `__Host-`. |
| 18 | Approved CORS origin (`https://varsha-homemade.onrender.com`) versus the approved custom-domain direction (Profile Y1) | The allow-list must be revisited and re-approved when custom domains are configured (§10.4). |
| 19 | Approved `SameSite=Lax` baseline versus the current cross-site `onrender.com` hostnames | The cookie flow cannot work until custom domains exist; any interim approach needs separate approval (§10.3). |
| 20 | A Supabase JWKS URL was provided versus the approved design, in which this service issues its own access tokens after Google sign-in | **Resolved (Step 5b):** Supabase Auth is not used; the JWKS URL is not used (§2.1). |
| 21 | Supabase's HTTP data API exposure versus token-store disclosure protection (USER-SEC-011) | Approved restriction and access-control design (§2.1). Schema, roles and privileges are decided at physical design; RLS is not a replacement for permissions. |
| 22 | Profile X cross-site cookies versus browser third-party-cookie blocking | Documented; users may need to sign in again; a CSRF token does not fix this (§10.6). Profile Y is the remedy once feasible. |
| 23 | The approved `X-CSRF-Protection` and `403 REQUEST_REJECTED` versus the API contract | Written into api-contracts §11. |
| 24 | The exposed Google client secret kept in use versus gating criterion USER-SD-AC-026 | Recorded as RISK-001; USER-SD-AC-026 keeps its original wording and is **NOT SATISFIED**. Not a secure state. |
| 25 | A nonce binder and session CSRF token versus the approved data model | Data-model §14 adds the nonce record and a CSRF-token digest on the session. |

No contradiction requires changing an approved requirement unless the owner chooses Option C (§6.2).

## 16. Requirements Traceability Matrix

| Area | Requirements | Sections |
|---|---|---|
| Google identity validation | USER-FR-002, 005, 006, 007, 061, 069; USER-SEC-001, 005 | §4 |
| Access tokens | USER-FR-020–026, 053, 072, 074, 084; USER-SEC-002, 007 | §5 |
| Refresh-token strategy | USER-FR-070–076, 080–083; USER-AC-015, 016, 026–029 | §6 |
| Refresh-token storage | USER-SEC-011; USER-AC-029; USER-FR-028 | §7 |
| Rotation, concurrency, revocation | USER-FR-073, 077, 080–083; USER-AC-026–028 | §8 |
| Logout | USER-FR-027, 073, 074, 084, 085; USER-SEC-012; USER-AC-017, 018 | §9 |
| Transport, cookies, CORS, CSRF | USER-SEC-006, 008, 010; USER-Q-004; API-DEC-002, 003 | §10 |
| Rate limiting | USER-SEC-009; USER-FR-078; USER-AC-014; USER-Q-002 | §11 |
| Secrets | USER-SEC-003, 004 | §12 |
| Events and retention | USER-FR-011, 065, 066, 082; USER-SEC-008; USER-PRE-001 | §13 |

## 17. Acceptance Criteria for the Security Design

IDs use `USER-SD-AC-nnn`. These are criteria for approving this design before implementation.

| ID | Criterion | Type | Traces to |
|---|---|---|---|
| USER-SD-AC-001 | The Google validation method checks signature, issuer, audience, expiry, and email verification on the server, and uses the Google account identifier as identity. | Required | USER-SEC-001; USER-FR-002, 005, 006 |
| USER-SD-AC-002 | The access-token algorithm is explicitly pinned, tokens signed with any other algorithm are rejected, and lifetime comes from configuration (15-minute default). | Required for lifetime and verification; algorithm choice depends on USER-SD-DEC-002 | USER-FR-026, 072; USER-SEC-002 |
| USER-SD-AC-003 | Access-token claims carry no personal data beyond the internal customer identifier. | Proposed | USER-DEC-010; USER-SEC-006 |
| USER-SD-AC-004 | The approved refresh strategy is demonstrated to satisfy consumed-token rejection, concurrency safety, event recording, session revocation, and disclosure protection. A non-rotating strategy is not approved on validation alone. | Required | USER-DM-AC-023; USER-FR-073, 081–083; USER-SEC-011 |
| USER-SD-AC-005 | The approved storage representation yields no usable refresh tokens if the store is disclosed, and no raw token, digest, or credential is logged. | Required | USER-SEC-011; USER-AC-029; USER-SEC-008 |
| USER-SD-AC-006 | The approved concurrency mechanism ensures at most one concurrent request consumes a token, and consumption and successor issuance cannot be left half-applied. | Required | USER-FR-083; USER-DM-AC-013, 024 |
| USER-SD-AC-007 | The decision on session revocation upon reuse is recorded, including its benign-collision cost. | Required (approved, USER-SD-DEC-008) | USER-FR-082; USER-DM-AC-026 |
| USER-SD-AC-008 | No grace period or consumed-token reuse exists unless a requirements change is approved. | Required | USER-FR-081 |
| USER-SD-AC-009 | The post-logout access-token window is documented, and no mechanism claims immediate invalidation. | Required | USER-FR-074, 085; USER-SEC-012 |
| USER-SD-AC-010 | Transport, cookie attributes, CORS allow-list, and CSRF protection are decided against stated deployment facts before implementation. | Conditional on API-DEC-002/003 | USER-Q-004 |
| USER-SD-AC-011 | Rate limits are configurable, cover sign-in and refresh, and return generic responses revealing no account information. | Required | USER-SEC-009; USER-FR-078 |
| USER-SD-AC-012 | Secrets are provided through configuration, validated at startup, and absent from the repository and logs. | Required | USER-SEC-003, 008 |
| USER-SD-AC-013 | A signing-key rotation and compromise procedure is defined. | Proposed | USER-SEC-003 |
| USER-SD-AC-014 | Security events cover the required types and contain no tokens, digests, credentials, secrets, or unnecessary personal data. | Required | USER-FR-011, 065, 066, 082; USER-SEC-008 |
| USER-SD-AC-015 | The retention decision explicitly addresses the loss of ability to recognize reuse when records are removed, and no retention period is chosen without approval. | Required | data-model §8.4; USER-PRE-001 |
| USER-SD-AC-016 | Any new dependency is approved before it is introduced. | Required | Requirements §11.1 |
| USER-SD-AC-017 | Before implementation, the owner either approves an attempt-binding design satisfying §4.2 or explicitly accepts the residual replay risk; signature and claim validation alone are not described as complete replay protection. | Required | USER-SD-DEC-001; API-DEC-001 |
| USER-SD-AC-018 | An ambiguous database outcome during refresh is never treated as a confirmed rollback, and the old token is not described as valid unless the system can establish the transaction did not commit. | Required (rotation approved) | USER-FR-081, 083; §8.1 |
| USER-SD-AC-019 | A rejected request stays rejected if security-event persistence fails; event-write failures are detected and surfaced for monitoring. | Required for the rejection; monitoring depends on USER-SD-DEC-024 | USER-FR-082; §13.2 |
| USER-SD-AC-020 | Deployment facts F1–F6 are confirmed before cookie transport, cookie attributes, CORS allow-list, CSRF strategy, and rate-limit storage are approved. | Required (gating) | USER-SD-DEC-013 to 017; §10.2 |
| USER-SD-AC-021 | Any clock-skew tolerance is small, bounded, explicitly configured, and cannot cause an expired token to be accepted beyond it. | Required | USER-FR-026, 072; USER-SEC-002 |
| USER-SD-AC-022 | The absolute session lifetime is 90 days (approved); a session cannot be extended beyond it by refresh; a successor token's expiry never exceeds the session's absolute expiry; a refresh at or after the absolute expiry is rejected. | Required (approved) | USER-SD-DEC-010; §8.3 |
| USER-SD-AC-023 | The statement that revoking a refresh session does not automatically invalidate an issued JWT access token is present and unchanged. | Required | USER-FR-074, 085; USER-SEC-012 |
| USER-SD-AC-024 | Refresh requests are coordinated one at a time per tab and never automatically retried after failure; the possibility of sign-out after a lost response is documented. | Required (frontend contract); cross-tab coordination Open | USER-SD-DEC-023, 009 |
| USER-SD-AC-025 | Presenting a consumed refresh token revokes the entire affected session and records the event. | Required (approved) | USER-SD-DEC-008; USER-FR-082; USER-DM-AC-026 |
| USER-SD-AC-026 | The previously exposed Google client secret has been rotated before any Google sign-in implementation, and no client secret appears in specifications, source control, logs, or browser-delivered code. | Required (gating). **NOT SATISFIED:** the prerequisite (rotation of the exposed Google client secret) is unresolved; RISK-001 is open by owner decision to keep the secret temporarily | USER-SD-DEC-018; §12 |
| USER-SD-AC-027 | No dependency is installed before it is individually approved, with its purpose, security implications, and alternatives documented. | Required | USER-SD-DEC-021 |
| USER-SD-AC-028 | Cookie attributes, the CORS allow-list, and the CSRF strategy are verified against the cross-site relationship of the confirmed origins before approval; refresh and logout work in the supported browsers when third-party cookie restrictions apply. | Required (gating) | USER-SD-DEC-013, 014, 015, 016 |
| USER-SD-AC-029 | The CORS allow-list contains only origins that the selected request and cookie strategy requires, uses no wildcard with credentialed requests, and does not list the API origin as the frontend's allowed origin. | Required | USER-SD-DEC-015 |
| USER-SD-AC-030 | The cookie baseline (`Secure`, `HttpOnly`, `Path=/`, `SameSite=Lax`, no `Domain`) is verified on the final domains in the supported browsers; `__Host-` is used only if its requirements are met. | Required (gating) | USER-SD-DEC-013, 014 |
| USER-SD-AC-031 | A cookie-authenticated, state-changing request with a missing or invalid `Origin` or required custom header is rejected **before** any token is consumed, with a generic response. | Required (approved) | USER-SD-DEC-016 |
| USER-SD-AC-032 | If the server-issued nonce is adopted, it is random, bound to the attempt, expires, and is consumed exactly once; the API and data-model changes are approved before implementation. | Conditional on USER-SD-DEC-001 | §4.4 |
| USER-SD-AC-033 | The rate-limit design states how the client address is identified and the trusted-proxy assumptions, and forwarded headers are not trusted without an explicit trusted-proxy configuration. | Required | USER-SD-DEC-017 |
| USER-SD-AC-034 | The retention decision documents the effect of deleting consumed-token records on the ability to recognize reuse; no period is set without approval. | Required | USER-SD-DEC-020; §13.3 |
| USER-SD-AC-035 | Startup validation covers required secrets and the approved lifetime maxima, and named owners exist for rotation, suspected compromise, and emergency revocation. | Required (gating) | USER-SD-DEC-018 |
| USER-SD-AC-036 | The exposed Supabase secret key is rotated before implementation. The database is verified not to expose the session, token-digest, nonce or event tables through any Supabase HTTP API, and the connection mode is verified against the atomic refresh transition and ambiguous-outcome handling. | Required (gating). **NOT SATISFIED:** the prerequisite (rotation of the exposed Supabase secret key) is unresolved; RISK-002 is open and the owner has not decided | USER-SEC-011; USER-SD-DEC-007; §2.1 |
| USER-SD-AC-037 | RLS is treated as defense in depth, not as a substitute for database permissions. The database owner role and the Express application login are separate, and the application role holds only the privileges it needs (no schema-change, superuser or row-security-bypass rights). | Required (approved) | USER-SD-DEC-025, 026; §2.1 |
| USER-SD-AC-038 | Render-to-Supabase connectivity is tested, and the connection method (direct, otherwise session-mode pooler) and role setup are confirmed by the owner, before the physical database design is finalized. TLS is required. | Required (gating) | USER-SD-DEC-026 |
| USER-SD-AC-039 | The nonce is single-use, expires in five minutes, and is consumed atomically on first presentation of a validly signed token. The binder's limitations are validated with the negative checks and threat-model review in §4.4. | Conditional on decisions 001 and 027 | §4.4 |
| USER-SD-AC-040 | The session CSRF token is random, only its digest is stored, it is never logged, and a request failing the Origin, header or token check is rejected with 403 `REQUEST_REJECTED` before any token is consumed. | Required (approved) | USER-SD-DEC-016, 028 |
| USER-SD-AC-041 | Under Profile X, a failed refresh caused by a missing cookie leads to a sign-in prompt, with no automatic retry loop, and the orphaned-session effect is accounted for in rate limits and retention. | Required (approved) | USER-SD-DEC-029 |
| USER-SD-AC-042 | No production profile is selected until the feasibility of the profile is verified, and the proxy assessment is documentation-only unless the owner authorizes a test. | Required | USER-SD-DEC-013; §10.7 |
| USER-SD-AC-043 | The Supabase client library and the Data API are not used. | Required (approved) | USER-SD-DEC-025, 026 |
| USER-SD-AC-044 | Before production, evidence is recorded that (a) database permissions are verified (role privileges, schema exposure through the HTTP APIs, RLS and grants), (b) authentication behavior is verified (all approved flows in the target browsers), and (c) secret handling is verified (secrets only server-side, startup validation, rotation procedure). | Required (gating; approved Step 5c); **NOT SATISFIED** until the evidence exists | USER-SD-DEC-018, 025, 026 |
| USER-SD-AC-045 | A manual, owner-handled support procedure for customers refused with `409 AUTH_IDENTITY_CONFLICT` is defined and documented before production. | Required (gating; Step 5d); **NOT SATISFIED** | USER-DM-DEC-002; data-model §15.4 |

## 18. Decisions, Statuses, and Remaining Approvals

### 18.1 Approved
003 to 012, 015, 016, 018 (approach), 019 (principle; atomicity and IP logging default recorded in Step 5c, §13), 021 (process), 022, 023 (single-tab), 024 (policy), 025 (Supabase role and HTTP API restriction, in principle), 028 (session CSRF token), 029 (Profile X sign-in fallback). **Step 5c** identity, profile and email decisions are recorded in data-model §15.

### 18.2 Approved (conditional or in principle)
| ID | Approved choice | Condition or limit |
|---|---|---|
| 001 | ID-token verification with a server-issued single-use nonce, five-minute lifetime | Approved in principle; documented in §4.4 for your final review; the new endpoint and record are in api-contracts §11 and data-model §14 |
| 002 | HS256 | Only while Express is the sole application JWT issuer and verifier |
| 013 | Profiles X and Y as alternatives; refresh cookie; access token in memory | **No production profile selected** until feasibility is verified |
| 014 | Per-profile cookie attributes (§10.3) | Conditional on the profile and on browser behavior |
| 026 | Standard PostgreSQL driver; test connectivity first; direct, otherwise session-mode pooler; dedicated least-privilege role | The connectivity test result and your confirmation of the connection method and role setup |
| 027 | Page-held binder (B2) | The limitations are validated explicitly (§4.4) |

### 18.3 Still open
- **017 rate limiting:** thresholds, storage, client-address identification, trusted-proxy configuration.
- **020 retention:** periods, after privacy, legal and operational review.
- **Remainders:** the named accountable owner for secrets (018); event definitions and the monitoring mechanism (019, 024); each dependency (021); cross-tab coordination (023); the Supabase secret key rotation decision (RISK-002); what depends on the Google client secret (RISK-001).

### 18.4 Items requiring inspection or confirmation of configuration (none performed by this document)
1. Render-to-Supabase connectivity (direct versus session-mode pooler; Render's outbound IPv6 support is unverified).
2. Whether a dedicated Express database role already exists, and the database setup process.
3. The Supabase project settings: which schemas the HTTP API exposes, whether the Data API can be disabled, Realtime publications.
4. The frontend's Render service type, and whether rewrites forward `POST`, cookies and headers.
5. Domain ownership (Profile Y1).
6. How Render exposes secrets for this service.
7. Browser testing of Profile X in the supported browsers.

### 18.5 Blockers for the physical PostgreSQL design
The connection method and role setup (items 1 and 2 above); the schema, privilege and RLS design; nonce and session CSRF-digest entities (approved in principle; data-model §14); DM-DEC-001, 002, 003, 008, 010, 011; the event-type enumeration; migration approach; retention (020).

### 18.6 Blockers for authentication implementation
Everything in 18.5, plus: API-DEC-004, 005, 006, 008, 009 to 015 (the API contract's own decisions); rate-limit thresholds and trusted-proxy configuration; a verified production profile; an assigned accountable owner; dependency approvals; and resolution of RISK-001 and RISK-002 before production.

### 18.7 Readiness
**Authentication module: NOT READY.** Nothing in this document authorizes implementation.

## 19. Cross-Document Consistency Report
**Confirmed alignments:** the approved refresh, rotation, session, token, logout and event decisions match requirements, the data model and the API contract. Supabase is PostgreSQL hosting only throughout. The nonce, binder, CSRF token, `X-CSRF-Protection` header and `403 REQUEST_REJECTED` are written into api-contracts §11, data-model §14 and frontend-contract.md.

**Resolved by Step 5b:** the Supabase Auth and JWKS open questions; the `Path` inconsistency between api-contracts §5.3 and the profiles; the header-versus-introduction status mismatch in api-contracts; stale labels in this document and the data model.

**Unresolved:** no production profile is selected; Profile X may require repeated sign-in in cookie-blocking browsers; RISK-001 and RISK-002; the physical-design blockers above.

### 18.8 Decisions taken under the owner's delegation (Step 5d, 2026-10-25)
The owner asked for the open questions to be decided. Everything below is **Approved (delegated)**: reasoned, recorded, and reversible by the owner. Items that need facts only the owner can supply stay open.

| Question | Decision | Reason |
|---|---|---|
| Email collision for an existing subject | Sign in, keep the stored email, record an event (data-model §15.4) | Identity is the subject; do not lock the customer out |
| Locked-out address owner | No self-service recovery; a manual, owner-handled support procedure must exist before production (`USER-SD-AC-045`) | A recovery flow is out of scope (USER-DEC-011); a procedure is still needed |
| `USER-DM-AC-003` | Amended wording, original preserved | Resolves the conflict-detection tension |
| Nonce replay; failed sign-in; rate-limit rejection | Replay is mandatory and durable; the other two are operational logs | Avoids write amplification |
| `USER-SEC-014` to `016`, `USER-FR-087` | Added, as worded, with criteria `USER-AC-031` to `039` (requirements §12.7) | Make the approved design testable |
| `API-DEC-005, 006, 008, 009, 010, 014, 015` | As recommended (api-contracts §13) | Consistent with the contract and requirements |
| Dependencies | `express`, `pg`, `jose`, `google-auth-library`, `express-rate-limit`, `cors`, `cookie-parser`, `node-pg-migrate` (development) approved individually; `jsonwebtoken` not selected; `helmet`, `dotenv`, `pino` deferred (§14.1). **Nothing is installed.** | Each is needed by the approved design; the rest are optional |
| Accountable owner for secrets, rotation and the infrastructure checks | **Defaults to the project owner (a role) until the owner names a person or delegate.** No name is invented. | An unowned role is itself a production risk |
| Supabase secret key | **Rotate or revoke before production** (production gate; unused by the design) | Elevated and exposed (RISK-002) |
| Google client secret | RISK-001 stays open and `USER-SD-AC-026` stays NOT SATISFIED; rotating it is a **production gate** | The owner's "temporarily" |
| Who runs the infrastructure checks | The project owner or a delegate with the relevant Render, Supabase or Google access; timing as in the checklist | Only the owner has the access |

**Not decided, because they need facts or a review only the owner can supply:** the connection method (needs the connectivity result), rate-limit numbers and the trusted-proxy hop count (need traffic and topology), retention periods (need a privacy, legal and operational review, and they also decide the no-foreign-key choice), which cookie profile is used in production, a named person for any role, and the numeric request-size limits.
