# Decision Record 0001 — Authentication Architecture Decisions

**Status:** Recorded 2026-10-24. This record preserves decision history; it does not replace the detailed specifications.

**Source of each decision:** the project owner's decision messages in the Step 5 and Step 5b workflow. Where a decision was approved only in principle or conditionally, that is stated. A recommendation is not a decision.

## 1. Architecture
| Fact or decision | Status |
|---|---|
| Render hosts the React frontend and the Express backend; Supabase hosts PostgreSQL; Google provides sign-in | Confirmed |
| Supabase is used for PostgreSQL only. Supabase Auth is not used. The application does not depend on the Supabase Data API unless the owner approves a change. | Approved in principle (USER-SD-DEC-025) |
| Express issues and verifies its own access JWTs | Approved |
| Standard PostgreSQL driver (such as `pg`), not the Supabase client library | Approved in principle (USER-SD-DEC-026); package approval pending |
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
| 020 | Retention | Open |
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
| ID | Risk | Decision |
|---|---|---|
| RISK-001 | The previously exposed Google client secret remains in use for now. Anyone holding it could impersonate the application in an authorization-code exchange. What depends on it is unknown. | **Owner decision (Step 5b): keep it temporarily.** Rotation is recommended before production. No secret value is recorded anywhere. |
| RISK-002 | A Supabase secret key was exposed. It is unused under the approved design. | The owner has **not** decided. Rotation before production is recommended. |
| RISK-003 | Profile X may require repeated sign-in in cookie-blocking browsers. | Accepted as a design constraint (decision 029); Profile Y is the remedy. |

The credential types are classified in security-design.md §12. **No named owner is recorded.** An accountable owner for secret management and emergency rotation must be assigned before production.

## 5. Open items
Connection method and role confirmation (connectivity test); the Supabase project settings (exposed schemas, Data API, Realtime); whether the frontend service type supports rewrites; domain ownership; rate-limit thresholds and trusted-proxy configuration; retention periods; named owners; dependency approvals; cross-tab coordination; the API-DEC items listed in api-contracts.md §11.6; proposed requirements USER-SEC-014 to 016 (not added; awaiting explicit approval).

## 6. History
| Date | Event |
|---|---|
| 2026-10-24 | Step 5: refresh-token, session, access-token, event and CORS/CSRF principles approved; absolute session lifetime 90 days; deployment facts recorded |
| 2026-10-24 | Step 5b: Supabase role, HS256, nonce, binder, CSRF token, header and status, profiles, secrets risk, and the two owner clarifications recorded; frontend contract created |
| 2026-10-25 | Step 5c: identity and profile decisions (UUID v4, unique provider and subject, nullable name, required validated email, **case-insensitive email uniqueness**, profile updates, stored session expiry); events without foreign keys (conditional on retention); mandatory events atomic with state changes; IP logging off by default; individual dependency approvals; production verification gates; AC-026 and AC-036 restored and marked NOT SATISFIED |
| 2026-10-25 | Step 5d: the owner delegated the open decisions; they were decided and recorded as **Approved (delegated)**, reversible by the owner (security-design §18.8) |

## 7. Step 5c notes (2026-10-25)
- **Email uniqueness:** the owner chose case-insensitive uniqueness over the recommended alternative (duplicates allowed). Implications are documented in data-model.md §15.4, including that a legitimate new owner of a reassigned address is locked out with no recovery path in the first release.
- **Acceptance criteria:** the original wording of USER-SD-AC-026 and USER-SD-AC-036 was restored from an unedited copy of the file taken before they were rewritten, because git history contains only the rewritten text. Both are marked NOT SATISFIED.
- **Not approved in Step 5c:** USER-SEC-014 to 016 and USER-FR-087; API-DEC-005, 006, 008, 009, 010, 014; the event enumeration; any dependency; named owners; the Supabase key rotation.

## 8. Delegated decisions (Step 5d, 2026-10-25)
The owner asked for the open decisions to be decided. They are recorded as **Approved (delegated)** in security-design.md §18.8, api-contracts.md §13, data-model.md §15 and requirements.md §12.7. The owner may override any of them. Still open, because only the owner can supply the facts: the connection method, rate-limit numbers and trusted-proxy hops, retention periods, the production cookie profile, named persons, and numeric request-size limits. Production gates added: a support procedure for the email conflict (`USER-SD-AC-045`), rotation of the exposed Supabase and Google secrets, and the verification evidence in `USER-SD-AC-044`.
