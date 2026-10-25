# Decision Record 0001 — Authentication Architecture Decisions

**Status:** Recorded 2026-10-24. This record preserves decision history; it does not replace the detailed specifications.

**Source of each decision:** the project owner's decision messages in the Step 5 and Step 5b workflow. Where a decision was approved only in principle or conditionally, that is stated. A recommendation is not a decision.

## 1. Architecture
| Fact or decision | Status |
|---|---|
| Render hosts the React frontend and the Express backend; Supabase hosts PostgreSQL; Google provides sign-in | Confirmed |
| Supabase is used for PostgreSQL only. Supabase Auth is not used. The application does not depend on the Supabase Data API unless the owner approves a change. | Approved in principle (USER-SD-DEC-025) |
| Express issues and verifies its own access JWTs | Approved |
| Standard PostgreSQL driver (such as `pg`), not the Supabase client library | Approved in principle (USER-SD-DEC-026); `pg` approved under delegation (Step 5d), not installed |
| Frontend `https://varsha-homemade.onrender.com`; API `https://varsha-customer-service.onrender.com`; cross-site | Confirmed; unchanged |
| No gateway, no BFF; API directly reachable; one instance; browser clients only | Confirmed |

## 2. Decision register summary (details in security-design.md §3)
| ID | Decision | Status |
|---|---|---|
| 001 | ID-token verification with a server-issued single-use nonce, five-minute lifetime | Approved (conditional, in principle) |
| 002 | HS256 while Express is the only issuer and verifier | Approved (conditional) |
| 003–012 | Claims and skew; opaque refresh tokens; digest storage; atomic rotation; reuse revokes the session; no grace period; 90-day cap; multiple sessions; logout | Approved |
| 013, 014 | Profile X and Profile Y as alternatives; no production profile selected; cookie attributes per profile | Approved (conditional) |
| 015, 016 | CORS allow-list; CSRF baseline (`X-CSRF-Protection`, `403 REQUEST_REJECTED`, Origin validation, missing Origin rejected on refresh and logout) | Approved |
| 017 | Rate limiting thresholds and storage | Open |
| 018 | Secret handling approach; accountable owner to be assigned before production | Approved (approach); owner open |
| 019, 021, 022, 023, 024 | Event persistence principle; dependency process; access-token revocation deferred; single-tab coordination; event-write failure policy | Approved (in part) |
| 020 | Retention | Open (180-day event default proposed, provisional) |
| 025 | Supabase role and HTTP API restriction | Approved in principle |
| 026 | Connection method (test first; direct, otherwise session-mode pooler), driver, dedicated least-privilege role | Approved (conditional) |
| 027 | Page-held binder, limitations validated | Approved (conditional, in principle) |
| 028 | Session-scoped CSRF token in both profiles | Approved |
| 029 | Profile X sign-in fallback where third-party cookies are blocked | Approved |

## 3. Owner clarifications recorded
1. **RLS is not a replacement for database permissions.** The Express login role has only the privileges the application needs, and the database owner and the application login are separate roles.
2. **Profile X has a browser-compatibility risk.** Because the Render origins are cross-site, browsers that block third-party cookies may prevent silent refresh. A CSRF token does not solve that. The design must account for users needing to sign in again.
3. No custom-domain ownership is confirmed. Keep the current origins until a deployment change is approved. A same-origin proxy **feasibility assessment** is authorized; no deployment change is.

## 4. Unresolved security risks (owner decisions; not secure or approved production states)
States are the five defined in security-design.md §21: documented, approved for remediation, remediated, verified, production gate satisfied. **No risk is resolved.**
| ID | Risk | Accountable role | State | Decision and remediation | Evidence of remediation | Blocks |
|---|---|---|---|---|---|---|
| RISK-001 | The previously exposed Google client secret remains in use for now. Anyone holding it could impersonate the application in an authorization-code exchange. What depends on it is unknown. | Project owner (interim) | Documented. Gate `USER-SD-AC-026` NOT SATISFIED | **Owner decision (Step 5b): keep it temporarily.** Rotation before production is a delegated production gate (Step 5d). No secret value is recorded anywhere. | None | Production |
| RISK-002 | A Supabase secret key was exposed. It is not required by the approved direct-PostgreSQL architecture. | Project owner (interim) | Documented; remediation approved (delegated): rotate or revoke before production. Gate `USER-SD-AC-036` NOT SATISFIED | Rotate or revoke | None | Production |
| RISK-003 | Profile X may require repeated sign-in in cookie-blocking browsers. | Project owner (interim) | Documented | Accepted as a design constraint (decision 029); Profile Y is the remedy but is not selected | None | Production on Profile X |
| RISK-004 | **No individual is named as accountable** for secret management, emergency rotation, the infrastructure checks, or the support procedure. The project owner is the interim accountable role. **Impact:** a compromise could be answered slowly or by no one, and checks and the support procedure could go unowned. **Mitigation:** the owner names a person or delegate and records their authority. **Relationship:** distinct from RISK-001 and RISK-002, which are credential exposures; RISK-004 is the governance gap that affects how they are remediated. It is not a duplicate. | Project owner (interim) | Documented (Step 5e); the repository audit found it was not previously recorded | None approved | None | Production |

The credential types are classified in security-design.md §12. **No named owner is recorded.** An accountable owner for secret management and emergency rotation must be assigned before production.

## 5. Open items (refreshed Step 5e, 2026-10-25)
- **Infrastructure evidence (security-design §20, none performed):** connectivity and TLS; roles, grants, schema exposure, RLS, Realtime publications, exposed functions and views; the migration and ownership model; browser and cookie behavior; trusted-proxy behavior; measured request sizes.
- **Decisions waiting on evidence or review:** the connection method; rate-limit thresholds and trusted-proxy hops; retention periods (event references decided as plain identifiers, Step 5f; the 180-day event default awaits privacy and legal review); the production cookie profile; the request-size values (PROVISIONAL — NOT VERIFIED).
- **Ownership:** a named accountable person (RISK-004); named owners for the checks.
- **Event-failure detection (USER-SEC-017):** the independent mechanism, minimum an operational log entry and a monitored indicator outside the database; the alerting tool and responder are open.
- **Production gates not satisfied:** `USER-SD-AC-026`, `036`, `044`, `045`.
- **Still open in the API contract:** API-DEC-012 (API-DEC-004 approved, Step 5f).
- **Dependencies:** the approved packages are not installed; deferred or unselected: `jsonwebtoken`, `helmet`, `dotenv`, `pino`.
- **Not yet specified:** a physical database design, a verification plan, shared conventions, deployment and operational specifications, a product-catalog domain.

## 6. History
| Date | Event |
|---|---|
| 2026-10-24 | Step 5: refresh-token, session, access-token, event and CORS/CSRF principles approved; absolute session lifetime 90 days; deployment facts recorded |
| 2026-10-24 | Step 5b: Supabase role, HS256, nonce, binder, CSRF token, header and status, profiles, secrets risk, and the two owner clarifications recorded; frontend contract created |
| 2026-10-25 | Step 5c: identity and profile decisions (UUID v4, unique provider and subject, nullable name, required validated email, **case-insensitive email uniqueness**, profile updates, stored session expiry); events without foreign keys (conditional on retention); mandatory events atomic with state changes; IP logging off by default; individual dependency approvals; production verification gates; AC-026 and AC-036 restored and marked NOT SATISFIED |
| 2026-10-25 | Step 5d: the owner delegated the open decisions; they were decided and recorded as **Approved (delegated)**, reversible by the owner (security-design §18.8) |
| 2026-10-25 | Step 5e: audit corrections applied (see §9): stale text refreshed, traceability added, retention table consolidated, infrastructure checklist and risk-state model persisted, AC-045 strengthened, RISK-004 recorded, request-size defaults documented as PROVISIONAL — NOT VERIFIED |
| 2026-10-25 | Step 5f: owner decisions on retention and event references, email-uniqueness requirement, USER-SEC-017/018, API-DEC-004 approved, RISK-004 kept open, API-DEC-012 kept open (§10) |
| 2026-10-25 | Step 5h: owner accepted the five decisions in physical-design.md §10; a database connection string was pasted into the conversation (see §12) |
| 2026-10-25 | Step 5g: audit corrections; delegated decisions on reuse revocation and existing-subject collision when the event cannot be stored; clarification of "identity conflict" (§11) |

## 7. Step 5c notes (2026-10-25)
- **Email uniqueness:** the owner chose case-insensitive uniqueness over the recommended alternative (duplicates allowed). Implications are documented in data-model.md §15.4, including that a legitimate new owner of a reassigned address is locked out with no recovery path in the first release.
- **Acceptance criteria:** the original wording of USER-SD-AC-026 and USER-SD-AC-036 was restored from an unedited copy of the file taken before they were rewritten, because git history contains only the rewritten text. Both are marked NOT SATISFIED.
- **Not approved in Step 5c:** USER-SEC-014 to 016 and USER-FR-087; API-DEC-005, 006, 008, 009, 010, 014; the event enumeration; any dependency; named owners; the Supabase key rotation.

## 8. Delegated decisions (Step 5d, 2026-10-25)
The owner asked for the open decisions to be decided. They are recorded as **Approved (delegated)** in security-design.md §18.8, api-contracts.md §13, data-model.md §15 and requirements.md §12.7. The owner may override any of them. Still open, because only the owner can supply the facts: the connection method, rate-limit numbers and trusted-proxy hops, retention periods, the production cookie profile, named persons, and numeric request-size limits. Production gates added: a support procedure for the email conflict (`USER-SD-AC-045`), rotation of the exposed Supabase and Google secrets, and the verification evidence in `USER-SD-AC-044`.

## 9. Step 5e notes (2026-10-25)
*Superseded by §10 (Step 5f): the proposed IDs below were approved. Kept for history.*
**Proposed new requirement IDs (not assigned; awaiting your approval):**
- **USER-SEC-017 (proposed):** "A mandatory security event MUST be persisted in the same database transaction as the state change it describes. If it cannot be persisted, the operation MUST fail (fail closed)." Proposed acceptance criterion: **USER-AC-040 (proposed)**.
- **USER-SEC-018 (proposed):** "Client IP addresses MUST NOT be written to operational logs by default. They MAY be recorded only for a documented security need that states the purpose, who may read them, how long they are kept, and the privacy implications." Proposed acceptance criterion: **USER-AC-041 (proposed)**.
- **Alternative using existing IDs:** none fits; both are new policies of general scope.
**Not proposed:** an email-uniqueness requirement, until the database constraints can be checked against the approved collision policy.
**Amended with the original wording preserved:** `USER-AC-031` (requirements), `USER-SD-AC-019` and `USER-SD-AC-045` (security design).

## 10. Step 5f owner decisions (2026-10-25)
- **Approved:** USER-SEC-017 with USER-AC-040; USER-SEC-018 with USER-AC-041 (the §9 proposals, now in requirements §12.9); a new email-uniqueness requirement `USER-FR-088` with `USER-AC-042` (numbers assigned by the specification author); API-DEC-004 (generic `AUTH_GOOGLE_FAILED`).
- **Proposed, provisional:** security-event retention of 180 days (pending privacy and legal review; not a legal requirement); events use plain nullable identifiers instead of foreign keys; session and token cleanup defined separately, with consumed-token records kept at least until the token's own expiry. USER-SD-DEC-020 stays Open.
- **Still open:** RISK-004 (no person named; the project owner is the interim accountable role; `USER-SD-AC-035` and `USER-SD-AC-045` not satisfied); API-DEC-012 (a separate post-sign-in Cart call is the preferred option under evaluation; waits for the Cart specification).
- No physical database design or implementation was created.

## 11. Step 5g (2026-10-25): audit corrections and delegated decisions
Decided under the owner's delegation; the owner may override.
- **Reuse revocation vs fail closed:** the session is still revoked if the event cannot be stored (Exception 1 to USER-SEC-017); failure detected independently.
- **Existing-subject email collision:** sign-in succeeds even if the conflict event cannot be stored (Exception 2); same event type, distinct reason code, no email in the event.
- **"Identity conflict"** in USER-FR-011, USER-FR-066 and USER-AC-013 is clarified (original wording kept) as an unknown subject with a colliding email.
- Corrected: the UUID check (PostgreSQL-generated), stale retention, dependency, nonce and ID-approval text, traceability rows, and checks 16 and 17 in security-design §20.
- Added to the physical-design prerequisites: the transaction handling of the unique violation, and the transaction boundaries for the two exceptions. **Not decided.**
- Unchanged: RISK-001 to RISK-004, `USER-SD-AC-026`, `035`, `036`, `044`, `045` (NOT SATISFIED), API-DEC-012 (Open), retention periods (provisional). No physical design or implementation exists.

## 12. Step 5h (2026-10-25)
- **Accepted by the owner:** the five decisions in physical-design.md §10 (SHA-256 unkeyed digests and the encoding rule; the lower-cased-email expression index; the privilege and RLS design; the reason-code and revocation-reason enumerations; the retention proposal and the owner-run event-removal path). Retention remains provisional pending privacy and legal review. The physical design stays DRAFT until the infrastructure evidence exists.
- **Credential exposure:** a database connection string that includes a password was pasted into the conversation. It is **not recorded in any file**. It must be treated as **exposed** and rotated. It names a privileged host-level role, which the approved design says must stay off the runtime (security-design §2.1, §14.1). This adds to RISK-002 (exposed Supabase credentials; state still 1, Documented) and to RISK-004 (no named accountable person). **No remediation is recorded as done.**
- **Evidence:** no check in security-design §20 was performed or has a supplied result.
