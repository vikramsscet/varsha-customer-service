# Decision Record 0002 — SDD Implementation-Readiness Review

**Status:** Persisted from the readiness review of 2026-10-24, updated for Step 5b. This record preserves the review; it does not authorize implementation.

## 1. Basis
Reviewed: specs/README.md, 01-user/requirements.md, api-contracts.md, data-model.md, security-design.md and frontend-contract.md. No source code and no configuration files exist in the repository. Modules 02 to 08 have no specifications.

## 2. Readiness by module (updated after Step 5b)
| Module | Status | Main gaps |
|---|---|---|
| 01 Foundation (configuration, errors, logging, database connection) | NOT READY | Physical database design; shared conventions; connectivity test and role setup; dependency approvals |
| 01 Persistence and token lifecycle | READY WITH EXPLICIT ASSUMPTIONS (logical level only) | Needs the physical design first. Assumptions: digest algorithm, identifier format, absolute expiry storage, event types |
| 01 Security-event recording | READY WITH EXPLICIT ASSUMPTIONS (logical level only) | Event list and monitoring mechanism |
| 01 Access tokens (HS256) | NOT READY | HS256 conditional; dependency approval |
| 01 Google sign-in | NOT READY | Nonce flow documented for final review; API-DEC-004, 006, 013; dependency approval |
| 01 `GET /users/me` | NOT READY | Nullability decisions (DM-DEC-001, 010, 011); API-DEC-011 |
| 01 Refresh and logout | NOT READY | No production profile selected; cookie and CORS depend on it; API-DEC-008 |
| 01 Rate limiting | NOT READY | Decision 017 |
| 02 to 08 | NOT READY | No specifications. A product-catalog domain is also missing from the sequence. |

## 3. Findings
- **Cross-module gaps:** product catalog; the frontend contract (now drafted); Cart merge on sign-in; the Checkout authentication prompt; order retention; integration verification; shared conventions (configuration, errors, logging, validation); a deployment and monitoring specification; a verification plan.
- **Physical PostgreSQL decisions open:** connection method and role setup; schema, privilege and RLS design; identifier format; column types; digest algorithm; constraints and indexes; transaction boundaries against the pooling mode; migration approach; nonce cleanup; retention.
- **Recommended implementation order (not approved):** foundation, persistence, access tokens, Google sign-in, `GET /users/me`, refresh and logout with the CSRF controls, then rate limiting and monitoring; then product catalog, Cart, Checkout, Order, Payment, Delivery, Feedback, Integration.
- **Minimum approvals before the first module:** an approved physical database design; shared conventions; individual dependency approvals; the connectivity test and role setup.

## 4. Resolved since the review
Stale labels in the data model and security design; the API header-versus-introduction mismatch; the `Path` inconsistency; the Supabase role; the missing frontend contract and decision records.

## 5. Not resolved
See decision record 0001 §5.
