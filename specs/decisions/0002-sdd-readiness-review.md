# Decision Record 0002 — SDD Implementation-Readiness Review

**Status:** Persisted from the readiness review of 2026-10-24; readiness table and findings refreshed after the audit of 2026-10-25 (Step 5e). This record preserves the review; it does not authorize implementation.

## 1. Basis
Reviewed: specs/README.md, 01-user/requirements.md, api-contracts.md, data-model.md, security-design.md and frontend-contract.md. No source code and no configuration files exist in the repository. Modules 02 to 08 have no specifications.

## 2. Readiness by module (actual status, audit of 2026-10-25)
| Module | Status | Main gaps |
|---|---|---|
| 01 Specifications (requirements, API, data model, security design, frontend contract) | **Substantively complete; not final** | Open decisions and unsatisfied gates (below) |
| 01 Physical database design | **DRAFTED, NOT APPROVED** ([physical-design.md](../01-user/physical-design.md)) | Owner decisions accepted (Step 5h); infrastructure evidence not produced; retention provisional |
| 01 Foundation (configuration, errors, logging, database connection) | NOT READY | Physical design; shared conventions; connectivity evidence |
| 01 Persistence and token lifecycle | NOT READY (logical specification is sufficient; blocked by the physical design) | Physical design |
| 01 Security-event recording | NOT READY (logical specification sufficient) | Physical design; retention |
| 01 Access tokens (HS256) | NOT READY | Physical design; implementation approval |
| 01 Google sign-in with nonce | NOT READY | Request-size verification; physical design |
| 01 `GET /users/me` | NOT READY | Physical design |
| 01 Refresh and logout | NOT READY | Production cookie profile; physical design |
| 01 Rate limiting | NOT READY | Thresholds and trusted-proxy evidence |
| 02 to 08 | NOT READY | No specifications; a product-catalog domain is also missing |
**No implementation is authorized.**

## 3. Findings
- **Cross-module gaps:** product catalog; the frontend contract (now drafted); Cart merge on sign-in; the Checkout authentication prompt; order retention; integration verification; shared conventions (configuration, errors, logging, validation); a deployment and monitoring specification; a verification plan.
- **Physical PostgreSQL decisions open:** connection method and role setup; schema, privilege and RLS design; identifier format; column types; digest algorithm; constraints and indexes; transaction boundaries against the pooling mode; migration approach; nonce cleanup; retention.
- **Recommended implementation order (not approved):** foundation, persistence, access tokens, Google sign-in, `GET /users/me`, refresh and logout with the CSRF controls, then rate limiting and monitoring; then product catalog, Cart, Checkout, Order, Payment, Delivery, Feedback, Integration.
- **Minimum approvals before the first module:** an approved physical database design; shared conventions; dependency installation (approved under delegation, not installed); the connectivity test and role setup.

## 3A. Step 5f update (2026-10-25)
Owner decisions on retention, event references, email uniqueness, USER-SEC-017/018 and API-DEC-004 are recorded (decisions/0001 §10). Readiness is unchanged: the physical design is still NOT READY because retention is provisional, the connection method and infrastructure evidence are missing, and API-DEC-012 and RISK-004 are open.

## 3B. Step 5g update (2026-10-25)
Audit contradictions were corrected and three delegated decisions were recorded (decisions/0001 §11). The physical design is still NOT READY: the connection method, infrastructure evidence, digest algorithm and encoding, the unique-violation handling and the provisional retention remain open. Drafting the design document can be considered once you authorize it.

## 4. Resolved since the review
Stale labels; the API header mismatch; the `Path` inconsistency; the Supabase role; the frontend contract and decision records; the identity and email decisions; the delegated API decisions; the Step 5e corrections (stale text, traceability, consolidated retention table, persisted infrastructure checklist and risk-state model, strengthened `USER-SD-AC-045`, RISK-004).

## 5. Not resolved
See decision record 0001 §5.
