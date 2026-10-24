# 01 — User & Authentication: Requirements

**Status:** Approved for implementation planning, as confirmed by the project owner in the Step 3 and Step 4 instructions. Security-design decisions approved in Step 5 are recorded in §12.
- All 14 previously open business decisions (USER-DEC-001 to USER-DEC-014) have been approved by the project owner.
- This document has been updated to reflect those decisions.
- The project owner has since confirmed these requirements as approved for implementation planning. The Step 5 security-design decisions answer some of the open design questions (USER-Q-001, USER-Q-003) and are recorded in §12. **They do not change any requirement text.**

**Scope of this document:** business and functional requirements only. No API contracts, data model, or implementation.

Keyword usage: **MUST** = mandatory; **SHOULD** = recommended, deviation needs justification; **MAY** = optional.

---

## 1. Purpose

The User & Authentication domain is responsible for customer identity, authentication, authenticated sessions, and account-related behavior for Varsha Homemade. It establishes *who* a caller is (a guest or a specific authenticated customer) so that other domains (Cart, Checkout, Order, Payment, Delivery, Feedback) can decide what that caller may do.

## 2. Scope

### 2.1 In scope
- Customer identity.
- Google sign-up and sign-in.
- Application authentication using JWT access tokens and refresh tokens.
- Guest and authenticated access models.
- Authentication status and session handling (renewal, logout).
- Account-related behavior.
- Authentication failure handling.
- Secure handling of authentication credentials and tokens.
- Authentication-abuse protection.
- Authentication-related authorization requirements.

### 2.2 Out of scope
- Cart operations (including guest-cart lifecycle and merge details).
- Checkout processing.
- Order management.
- Payment processing.
- Delivery tracking.
- Customer feedback.
- Product browsing APIs (product/catalog domain).
- Authentication providers other than Google (first release).
- Administrative identities, administrator permissions, and role-based administration (USER-DEC-012).
- Dedicated account deletion and account recovery workflows (USER-DEC-011).

Other domains depend on this domain, but their business functionality is specified separately.

## 3. Customer Types and Access Categories

### 3.1 Guest customer
- A visitor who has not authenticated.
- MAY browse public products and MAY create and manage their own guest cart (USER-DEC-001, USER-DEC-002).
- MUST NOT complete checkout (USER-DEC-003).
- MUST NOT automatically receive the permissions of an authenticated customer.
- MUST NOT be treated as having a verified customer identity or as an authenticated customer.

### 3.2 Authenticated customer
- A customer who has successfully authenticated with Google and holds a valid application access token.
- Has a customer identity recognized by the application.
- MUST be able to access only the functionality permitted to authenticated customers, and only their own private data.

### 3.3 Access categories
| Category | Meaning | Examples (final lists are set in the API-contract step) |
|---|---|---|
| **Public** | No authenticated identity required. | Public product browsing. |
| **Guest-accessible** | Explicitly permitted to an unauthenticated visitor, limited to that visitor's own resources. | Managing the visitor's own guest cart. |
| **Authenticated-only** | Requires a valid authenticated customer identity. | Checkout; access to protected customer information. |

Guests MUST NOT be treated as authenticated customers in any category.

## 4. Functional Requirements

### 4.1 Google Authentication
| ID | Requirement |
|---|---|
| USER-FR-001 | A customer MUST be able to sign up or sign in using their Google account. Google is the only customer authentication provider in the first release (USER-DEC-009). |
| USER-FR-002 | The service MUST validate Google authentication responses or identity tokens on the server side; client-supplied identity claims MUST NOT be trusted without validation. |
| USER-FR-003 | On successful validation, the service MUST identify the customer associated with the Google account. |
| USER-FR-004 | If no customer exists for a validated Google account, the service MUST create one (sign-up); otherwise it MUST sign in the existing customer. |
| USER-FR-005 | The service MUST use the stable Google account identifier from validated Google identity information as the basis of customer identity. Email MUST NOT be the sole unique identity key. |
| USER-FR-006 | The service MUST reject Google authentication when the validated Google identity information indicates the email address is not verified (USER-DEC-014). No customer is created and no session is issued. |
| USER-FR-007 | Invalid, expired, or rejected Google credentials MUST be refused, and no authenticated application session MUST be created. |
| USER-FR-008 | A cancelled or failed Google authentication MUST result in no authenticated session and MUST leave the customer as a guest. |
| USER-FR-009 | Repeated sign-ins by the same Google account MUST resolve to the same internal customer identity. |
| USER-FR-010 | The service MUST NOT create duplicate customer identities for the same Google account. |
| USER-FR-011 | Identity conflicts MUST be handled safely: the sign-in MUST be refused rather than automatically merged or guessed, and the conflict MUST be recorded for review. |

### 4.2 Access Tokens, Refresh Tokens, and Logout
| ID | Requirement |
|---|---|
| USER-FR-020 | After successful authentication, the service MUST issue a short-lived JWT access token and a refresh token, both associated with the customer (USER-DEC-006). |
| USER-FR-021 | Protected functionality MUST validate the access token on every request. |
| USER-FR-022 | Invalid, expired, or improperly signed access tokens MUST be rejected. |
| USER-FR-023 | A valid access token MUST resolve to exactly one customer identity. |
| USER-FR-024 | A request to protected functionality without an access token MUST be rejected. |
| USER-FR-025 | A malformed authorization header MUST be rejected and MUST be treated the same as missing credentials for access purposes. |
| USER-FR-026 | Access-token expiration MUST be enforced. The initial access-token lifetime is **15 minutes** (USER-DEC-007). |
| USER-FR-027 | The service MUST support logout of the current session, which revokes the applicable refresh token or refresh session (see USER-FR-073). |
| USER-FR-028 | Credentials and tokens MUST NOT appear in logs or error responses. |
| USER-FR-070 | A valid, unexpired, unrevoked refresh token MUST allow the customer to obtain a new access token without repeating Google sign-in. |
| USER-FR-071 | The initial refresh-token lifetime is **7 days** (USER-DEC-007). Expired refresh tokens MUST be rejected, and the customer MUST sign in with Google again. |
| USER-FR-072 | Access-token and refresh-token lifetimes MUST be configurable through the application's configuration mechanism and MUST NOT be hard-coded in business logic. The configured lifetimes MUST be enforced. Changing the defaults beyond the approved 15 minutes / 7 days requires an approved decision. |
| USER-FR-073 | **Current-session logout:** on logout, the refresh token or refresh session applicable to the current session MUST be invalidated or revoked, and a revoked refresh token MUST NOT be usable to obtain new access tokens. Logout from other sessions or all devices is not defined here (see USER-Q-003). |
| USER-FR-074 | **Logout does not necessarily invalidate an already-issued access token.** Existing access tokens MAY remain valid until they expire naturally (at most the configured access-token lifetime, 15 minutes by default). This is accepted behavior under USER-DEC-008 and MUST be documented in the security design. |
| USER-FR-075 | Refresh-token handling MUST support secure storage, validation, expiration, revocation, and the lifecycle controls in USER-FR-080 to USER-FR-083. Refresh tokens MUST NOT be assumed to be JWTs; token format and storage approach are determined in the security and design review. |
| USER-FR-076 | Invalid, unknown, expired, malformed, or revoked refresh tokens MUST be rejected without issuing an access token and without revealing why beyond a generic reason. |
| USER-FR-080 | **Refresh-token rotation** is the recommended initial policy: each successful refresh SHOULD consume the presented refresh token and issue a replacement. Final adoption is confirmed in the security design (USER-Q-001). |
| USER-FR-081 | A refresh token that has been successfully consumed MUST NOT be independently reusable to obtain further access tokens. |
| USER-FR-082 | Presentation of an invalidated (consumed, revoked, or superseded) refresh token MUST be rejected under USER-FR-076, MUST be recorded as a security event without logging the token, and MUST NOT disclose the token's prior state to the caller. The exact response (for example, whether the related refresh session is also revoked) is an unresolved security design detail (USER-Q-001). |
| USER-FR-083 | Concurrent refresh requests using the same refresh token MUST be handled safely: at most one request may successfully consume the token, and the outcome MUST NOT yield more than one independently usable refresh token chain or leave the session in an undefined state. The handling mechanism, and any tolerance for benign client retries, is an unresolved security design detail (USER-Q-001). |
| USER-FR-084 | Protected APIs MUST continue to validate access tokens on every request (USER-FR-021) regardless of logout status; logout MUST NOT be relied upon to disable access-token validation. |
| USER-FR-085 | If immediate invalidation of already-issued access tokens is required (for example, after account compromise), a separate mechanism MUST be proposed and approved. No such mechanism is part of this specification. |

### 4.3 Guest Access
**Responsibility split:** User & Authentication is responsible for establishing customer identity after successful authentication. The Cart domain owns guest-cart creation, persistence, ownership, retention, and merge behavior. Integration verification (domain 08) validates the complete guest-to-authenticated-cart flow.

| ID | Requirement |
|---|---|
| USER-FR-030 | The service MUST distinguish guest callers from authenticated customers on every request. |
| USER-FR-031 | Every operation MUST be classified as public, guest-accessible, or authenticated-only (see §3.3 and USER-FR-050). Guests MUST NOT access an operation unless it is public or explicitly guest-accessible. |
| USER-FR-032 | Customers MUST be able to browse publicly available products without signing in (USER-DEC-001). Product browsing is a public capability; its APIs belong to the product/catalog domain. |
| USER-FR-033 | Guests MUST be allowed to create and manage a shopping cart without signing in (USER-DEC-002). Guest-cart lifecycle, ownership, and access mechanisms are defined by the Cart domain. |
| USER-FR-034 | When a guest signs in, the service MUST support transferring or merging the guest cart into the authenticated customer's cart (USER-DEC-005). Detailed merge behavior (duplicates, quantities, conflicts, errors) is defined by the Cart specification. The merge MUST NOT allow one customer to access or modify another customer's cart. |
| USER-FR-035 | Guests MUST NOT complete checkout; customers MUST authenticate before completing checkout (USER-DEC-003). How customers are prompted to authenticate is defined by the Checkout domain. |
| USER-FR-036 | Guest cart identifiers MUST be protected against unauthorized access so that one guest cannot access another guest's cart (USER-DEC-002). |
| USER-FR-037 | Guest carts SHOULD persist across browser restarts for a limited period (USER-DEC-004). The retention period is defined in the Cart specification. Guest cart contents MUST NOT be exposed to unauthorized users. |

### 4.4 Customer Identity
| ID | Requirement |
|---|---|
| USER-FR-040 | Each customer MUST be assigned a stable internal customer identifier that is unique, never changes, and is never reused. |
| USER-FR-041 | Each customer MUST be associated with the stable Google account identifier obtained from validated Google identity information. |
| USER-FR-042 | An authenticated customer MUST be able to retrieve their own identity information. |
| USER-FR-043 | Identity MUST remain consistent across repeated sign-ins (see USER-FR-009). |
| USER-FR-044 | A customer's identity MUST NOT be usable to access another customer's resources. |
| USER-FR-045 | Duplicate-identity and conflict scenarios MUST be handled per USER-FR-010 and USER-FR-011. |
| USER-FR-046 | Changes to Google profile information, such as name or email address, MUST NOT automatically create a new customer identity. |
| USER-FR-047 | An email address alone MUST NOT be used to merge customer accounts. |
| USER-FR-048 | The service MUST store only customer information needed for the approved business functionality (USER-DEC-010): the internal customer identifier, the Google account identifier, and necessary profile information such as name and email where required. It MUST NOT store unnecessary Google profile data. Field definitions belong to the data-model step. |
| USER-FR-049 | Customer identity SHOULD avoid unnecessary coupling to provider-specific profile information so that additional providers can be considered in the future (USER-DEC-009). No other provider is implemented in the first release. |

### 4.5 Authorization
Authentication establishes *who* the customer is. Authorization determines *what* they may do.

| ID | Requirement |
|---|---|
| USER-FR-050 | Every operation MUST be classified as **public**, **guest-accessible**, or **authenticated-only**. The detailed list is established in the API-contract step. |
| USER-FR-051 | Authenticated-only operations (including checkout and access to protected customer information) MUST reject guests. |
| USER-FR-052 | Customer-specific information MUST be accessible only to the customer it belongs to. |
| USER-FR-053 | Authorization decisions MUST use the authenticated identity established by the server from the validated access token, not a customer identifier supplied by the client. |
| USER-FR-054 | A customer MUST NOT be able to access another customer's private data by altering an identifier in a request. |

No administrator role or role-based administration is introduced (USER-DEC-012) unless separately approved.

## 5. Security Requirements

| ID | Requirement |
|---|---|
| USER-SEC-001 | Google identity assertions MUST be verified server-side for authenticity, issuer, intended audience, and expiry before being trusted. |
| USER-SEC-002 | Access tokens MUST be verified for signature integrity, expiry, and other approved claims before being trusted. |
| USER-SEC-003 | Secrets and credentials (signing keys, Google client secrets) MUST be supplied through configuration, MUST NOT be committed to the repository, and MUST NOT be exposed to clients. |
| USER-SEC-004 | Access to secrets and to customer identity data MUST follow least privilege. |
| USER-SEC-005 | All authentication-related inputs MUST be validated for presence, type, format, and size before use. |
| USER-SEC-006 | Authentication error responses MUST be safe: they MUST NOT reveal credentials, tokens, secrets, internal state, or unnecessary personal or account information. |
| USER-SEC-007 | The service MUST protect against identity impersonation, including forged, expired, or substituted tokens and identity claims. |
| USER-SEC-008 | Logs MUST NOT contain tokens, credentials, or secrets, and SHOULD minimize personal data. |
| USER-SEC-009 | The service MUST include controls to limit repeated failed or abusive authentication attempts, including sign-in and refresh-token requests (USER-DEC-013). Thresholds and enforcement policies are set in the security and implementation design. Controls SHOULD be configurable and MUST NOT expose sensitive account information in error responses. |
| USER-SEC-010 | In production, credentials and tokens MUST be transmitted only over encrypted transport. |
| USER-SEC-011 | Refresh tokens MUST be stored securely and MUST be revocable. The token store MUST NOT contain refresh tokens in a form that allows an attacker who obtains the store to use them (no reusable refresh tokens exposed on disclosure). The service MUST enforce the lifecycle controls in USER-FR-080 to USER-FR-083. Storage design and unresolved lifecycle details are determined in the security and design review. |
| USER-SEC-012 | The security design MUST document that logout does not immediately invalidate issued access tokens (USER-FR-074). |
| USER-SEC-013 | Guest cart identifiers MUST be unguessable and protected against unauthorized access (see USER-FR-036). Mechanism is defined in the Cart specification and reviewed in the security design. |

## 6. Error Handling Requirements

Outcomes only; response formats and status codes are defined in the API contract.

| ID | Scenario | Expected outcome |
|---|---|---|
| USER-FR-060 | Google authentication fails | No session is created; the customer remains a guest; a safe, generic failure is reported. |
| USER-FR-061 | Google credentials invalid or expired | Treated as failed authentication; no session; no detail beyond a generic reason. |
| USER-FR-062 | Access token missing | Protected operation is refused as unauthenticated. |
| USER-FR-063 | Access token malformed | Protected operation is refused as unauthenticated. |
| USER-FR-064 | Access token invalid or expired | Protected operation is refused as unauthenticated; expiry MAY be distinguishable so clients can renew using the refresh token. |
| USER-FR-065 | Customer identity cannot be resolved | Access is refused; no fallback to another or guest identity; the event is logged without sensitive data. |
| USER-FR-066 | Identity conflict | Sign-in is refused; no merge; recorded for review. |
| USER-FR-067 | Access to a protected operation without authorization | Access is refused; no data about the requested resource is disclosed. |
| USER-FR-068 | All of the above | Handled consistently, with no exposure of credentials, tokens, or internal details. |
| USER-FR-069 | Google identity has an unverified email | Sign-in is refused per USER-FR-006; no customer created; no session; generic failure. |
| USER-FR-077 | Refresh token invalid, expired, or revoked | Renewal is refused; no access token issued; customer must sign in again. |
| USER-FR-078 | Too many failed or abusive attempts | Further attempts are limited per USER-SEC-009, without revealing sensitive account information. |
| USER-FR-079 | Guest attempts checkout | Refused as unauthenticated; guest cart is not lost. |

## 7. Dependencies

| Dependency | Notes |
|---|---|
| Google authentication services | Required for sign-up and first sign-in, and for sign-in after the refresh session ends. If unavailable, new sign-ins fail; guests, and customers holding a valid access token or valid refresh token, are unaffected because renewal does not involve Google. |
| Application token mechanism | Access-token signing key compromise or misconfiguration affects all authenticated access. Refresh-token validation and revocation require server-side state. |
| PostgreSQL | Needed for persistent customer identity and (subject to design) refresh-token state. If unavailable, sign-in, renewal, and identity resolution fail. Data model is a later step. |
| Cart domain | Defines guest-cart lifecycle, persistence, ownership, and merge. This domain depends on it for USER-FR-033 to USER-FR-037. |
| Checkout domain | Defines how a guest is prompted to authenticate before checkout. |
| Product/catalog domain | Provides public product browsing. |
| Configuration mechanism | Supplies token lifetimes, secrets, and abuse-protection settings. |

External-service failures MUST result in a safe authentication failure, never in a granted session.

## 8. Acceptance Criteria

Criteria whose detailed behavior belongs to another domain depend on that domain's approved specification and are noted accordingly.

| ID | Criterion | Traces to |
|---|---|---|
| USER-AC-001 | Given a valid Google identity with a verified email, the customer is authenticated and receives an access token and a refresh token. | FR-001–004, 020; DEC-006, 014 |
| USER-AC-002 | Given an invalid, expired, or rejected Google credential, no customer is created and no session is issued. | FR-007, 061 |
| USER-AC-003 | Repeated sign-ins with the same Google account resolve to the same internal customer identifier, with no duplicate customers. | FR-009, 010, 043 |
| USER-AC-004 | A valid, unexpired access token identifies the correct customer. | FR-021, 023 |
| USER-AC-005 | A missing, malformed, invalid, or expired access token cannot access protected functionality, and the failure is refused as unauthenticated. | FR-022, 024, 025, 062–064 |
| USER-AC-006 | A guest cannot access authenticated-only functionality. | FR-051 |
| USER-AC-007 | Changing a customer identifier in a request does not grant access to another customer's resources; authorization uses the server-established identity. | FR-053, 054, 044 |
| USER-AC-008 | Authentication failures expose no credentials, tokens, secrets, or unnecessary personal information in responses or logs. | FR-028, SEC-006, SEC-008 |
| USER-AC-009 | Public product browsing works without authentication. *(Depends on the product/catalog specification for details.)* | FR-032; DEC-001 |
| USER-AC-010 | Authentication-related failures are handled consistently across all scenarios in §6. | FR-068 |
| USER-AC-011 | A Google identity with an unverified email is rejected; no customer is created and no session is issued. | FR-006, 069; DEC-014 |
| USER-AC-012 | An access token stops being accepted after the configured lifetime (15 minutes by default). | FR-026, 072; DEC-007 |
| USER-AC-013 | An identity conflict results in a refused sign-in with no automatic merge, and the conflict is recorded. | FR-011, 066, 047 |
| USER-AC-014 | Repeated failed or abusive authentication attempts are limited, and the limiting responses reveal no sensitive account information. Thresholds depend on the security design. | SEC-009; FR-078; DEC-013 |
| USER-AC-015 | A valid, unexpired, unrevoked refresh token yields a new access token without Google sign-in. | FR-070; DEC-006 |
| USER-AC-016 | An invalid, unknown, malformed, expired (past configured lifetime, 7 days by default), or revoked refresh token yields no access token. | FR-071, 076, 077 |
| USER-AC-017 | After current-session logout, the revoked refresh token or session cannot obtain a new access token. | FR-027, 073; DEC-008 |
| USER-AC-018 | After logout, an access token issued earlier is not necessarily invalidated and remains usable until it expires naturally; protected APIs continue to validate it normally. This behavior is documented in the requirements and security design. | FR-074, 084, 085; SEC-012; DEC-008 |
| USER-AC-026 | Under the adopted rotation policy, a successfully consumed refresh token cannot be used again to obtain an access token. *(Final rotation behavior depends on the security design, USER-Q-001.)* | FR-080, 081; SEC-011 |
| USER-AC-027 | Reuse of an invalidated refresh token is rejected, recorded as a security event without the token being logged, and discloses nothing about the token's prior state. Any further session-level response follows the approved security design. | FR-082; SEC-008, 011 |
| USER-AC-028 | Concurrent refresh requests with the same refresh token result in at most one successful consumption and no undefined session state. *(Handling depends on the security design, USER-Q-001.)* | FR-083 |
| USER-AC-029 | A disclosed refresh-token store does not yield usable refresh tokens. *(Verified against the security design and data model.)* | SEC-011 |
| USER-AC-019 | Token lifetimes are read from configuration; changing the configured values changes enforced lifetimes without code changes. | FR-072; DEC-007 |
| USER-AC-020 | A guest can create and manage their own guest cart, and cannot access another guest's cart or an authenticated customer's cart. *(Cart owns guest-cart creation, persistence, and ownership. Dependent on the approved Cart specification and cross-domain integration verification.)* | FR-033, 036, 037; DEC-002, 004 |
| USER-AC-021 | A guest cannot complete checkout, and is directed to authenticate per the Checkout specification. *(Depends on the approved Checkout specification.)* | FR-035, 079; DEC-003 |
| USER-AC-022 | After a guest signs in, the authenticated customer identity is established by this domain, and the guest cart is transferred or merged into the customer's cart without any other customer's cart being accessed or modified. *(This domain verifies only identity establishment. Merge behavior belongs to Cart. Dependent on the approved Cart specification and cross-domain integration verification of the full guest-to-authenticated-cart flow.)* | FR-034; DEC-005 |
| USER-AC-023 | A change to the customer's Google name or email on a later sign-in does not create a new customer, and no accounts are merged by email alone. | FR-046, 047 |
| USER-AC-024 | Only the approved customer information is stored; unnecessary Google profile data is not. *(Verified against the data-model specification.)* | FR-048; DEC-010 |
| USER-AC-025 | The User & Authentication implementation does not include administrative authentication or roles, authentication providers other than Google, a dedicated account-deletion workflow, or a dedicated account-recovery workflow, unless separately approved. | DEC-009, 011, 012 |

## 9. Approved Decisions

All approved by the project owner. Status: **Approved**.

| ID | Decision | Reflected in |
|---|---|---|
| USER-DEC-001 | Guests can browse public products without signing in. | FR-032, AC-009 |
| USER-DEC-002 | Guests can create and manage a cart; guest cart identifiers are protected; lifecycle is defined by Cart. | FR-033, 036; SEC-013; AC-020 |
| USER-DEC-003 | Guest checkout is **not** allowed; customers must authenticate before completing checkout. | FR-035, 051, 079; AC-021 |
| USER-DEC-004 | Guest carts SHOULD persist across browser restarts for a limited period (defined in Cart). | FR-037; AC-020 |
| USER-DEC-005 | Guest cart is transferred or merged on sign-in; details in Cart. | FR-034; AC-022 |
| USER-DEC-006 | Use short-lived JWT access tokens plus refresh tokens; refresh tokens support secure storage, validation, expiration, revocation and are not assumed to be JWTs. | FR-020, 070, 075, 076; SEC-011; AC-015–016 |
| USER-DEC-007 | Initial lifetimes: access 15 minutes, refresh 7 days; configurable, enforced. | FR-026, 071, 072; AC-012, 016, 019 |
| USER-DEC-008 | Logout revokes the applicable refresh token or session; issued access tokens may remain valid until natural expiry; this must be documented. | FR-027, 073, 074, 084, 085; SEC-012; AC-017–018 |
| USER-DEC-009 | Google only for the first release; avoid unnecessary coupling to provider-specific data. | FR-001, 049; AC-025 |
| USER-DEC-010 | Store only needed customer information: internal ID, Google account ID, necessary name/email. | FR-048; AC-024 |
| USER-DEC-011 | Dedicated account deletion/recovery deferred; privacy and order-retention obligations must still be reviewed before production launch. | §2.2, §10; AC-025 |
| USER-DEC-012 | Administrative identities and roles out of scope for the first release. | §2.2, FR-050 note; AC-025 |
| USER-DEC-013 | Rate limiting and abuse protection are required; thresholds set in security/implementation design. | SEC-009; FR-078; AC-014 |
| USER-DEC-014 | Verified Google email required; Google account ID is the identity basis, email is not the sole key. | FR-005, 006, 069; AC-001, 011 |

## 10. Remaining Questions and Pre-Launch Obligations

None of the 14 business decisions remains open. The following items are intentionally **not decided here** and are assigned to later steps. They do not block review of this document.

| ID | Item | Owner step |
|---|---|---|
| USER-Q-001 | Refresh-token format and storage approach; confirmation of rotation as the initial policy; exact response to reuse of an invalidated token (e.g., whether the whole refresh session is revoked); concurrency-handling mechanism and tolerance for benign client retries (USER-FR-080 to 083). | Security and design review |
| USER-Q-002 | Concrete rate-limiting thresholds, scope (per client, per account), and enforcement policy. | Security and implementation design |
| USER-Q-003 | Whether a customer may hold multiple concurrent refresh sessions (e.g., several devices), and whether "log out of all devices" is needed. | Security and design review (needs product input if required) |
| USER-Q-004 | How tokens are transmitted between client and service. | API-contract step |
| USER-Q-005 | Product-browsing APIs and the final public / guest-accessible / authenticated-only operation lists. | API-contract step and product/catalog domain |
| USER-Q-006 | Guest cart identifier mechanism, retention period, merge details. | Cart specification |
| USER-Q-007 | How a guest is prompted to authenticate before checkout. | Checkout specification |
| USER-Q-008 | Database structure and field definitions for customer identity and refresh-token state. | Data-model step |
| USER-PRE-001 | **Before production launch**, review applicable privacy and data-protection obligations and order-retention requirements. Deferring account deletion and recovery (USER-DEC-011) is not permission to ignore these. | Pre-production review |

## 11. Assumptions and Constraints

### 11.1 Confirmed constraints
- Node.js, Express.js, JavaScript (no TypeScript).
- PostgreSQL.
- REST APIs.
- Google OAuth 2.0 as the only customer authentication provider in the first release.
- JWT access tokens plus refresh tokens for application authentication.
- Configuration via environment variables.
- No additional authentication providers or technologies without approval.

### 11.2 Remaining assumptions
- Any behavior not stated above is not approved. Behavior assigned to another domain (§10) MUST be taken from that domain's approved specification, not assumed here.
- Acceptance criteria marked as depending on other domains cannot be fully verified until those specifications are approved.

## 12. Security-Design Decisions Incorporated (Step 5, 2026-10-24)

The project owner approved the security-design decisions below. This section records them and their effect. **No existing requirement text or ID was changed.** One requirement (USER-FR-086) and one acceptance criterion (USER-AC-030) were added for the approved 90-day absolute session lifetime (§12.4). Details and rationale are in [security-design.md](security-design.md) §3.

### 12.1 Approved decisions and effect on this document
| Decision | Approved choice | Requirements it clarifies | Effect |
|---|---|---|---|
| USER-SD-DEC-004 | Strict single-use refresh-token rotation; no grace period | USER-FR-080–083 | USER-FR-080's "SHOULD" is not edited; the approved policy makes strict rotation the implemented behavior. The owner may choose to formalize it later. |
| USER-SD-DEC-005 | Opaque, cryptographically random refresh tokens; not JWTs | USER-FR-075 | Consistent |
| USER-SD-DEC-006 | Store a one-way digest of each refresh token; begin unkeyed; reassess keyed if database write access is in the threat model | USER-SEC-011; USER-AC-029 | Consistent |
| USER-SD-DEC-007 | Consume the token and create its successor in one transaction; atomic or equivalently safe; ambiguous outcomes are never confirmed rollbacks; losing request gets the generic refresh error | USER-FR-076, 083 | Consistent |
| USER-SD-DEC-008 | Consumed-token reuse rejects, records the event, and revokes the entire session | USER-FR-082; USER-AC-026, 027 | Applies USER-AC-026; no requirement text change (requirements defer session-level response to the security design) |
| USER-SD-DEC-009 | Occasional sign-outs from lost responses and concurrent refreshes accepted; no grace period; no automatic retry | USER-FR-081 | Consistent |
| USER-SD-DEC-003 | Access-token claims: issuer, audience, subject, issued-at, expiry; small, bounded, configurable clock skew that never bypasses expiry | USER-FR-023, 026, 053 | Consistent |
| USER-SD-DEC-011 | Multiple active sessions per customer; logout ends only the current session; no logout-all endpoint in the first release | USER-FR-027, 073 | Resolves USER-Q-003 |
| USER-SD-DEC-012 | Logout revokes the applicable refresh session; issued access tokens stay valid up to the 15-minute lifetime | USER-FR-073, 074, 084 | Consistent; residual window documented |
| USER-SD-DEC-010 | Absolute session lifetime of **90 days**; a session must not exceed it even when actively refreshed; refreshing never extends it | USER-FR-071, 072 | Adds USER-FR-086 and USER-AC-030 (§12.4) |
| USER-SD-DEC-013 | **Approved (conditional):** refresh-token `Secure` `HttpOnly` cookie, access token in memory; **Profile X and Profile Y are alternatives; no production profile is selected** until feasibility is verified | USER-SEC-010; USER-Q-004 | Conditional; depends on deployment feasibility |
| USER-SD-DEC-014 | **Approved (conditional):** cookie attributes per profile (security-design §10.3) | USER-SEC-006, 010 | Conditional on the profile |
| USER-SD-DEC-015 | Approved: sole CORS allowed origin `https://varsha-homemade.onrender.com`; no wildcard; credentials only if required | USER-SEC-003 | Revisit if custom domains are configured |
| USER-SD-DEC-016 | Approved: `X-CSRF-Protection` header, `403 REQUEST_REJECTED`, Origin validation, missing `Origin` rejected on refresh and logout, session CSRF token | USER-SEC-007 | Design-level; no requirement text changed |
| USER-SD-DEC-018 | Approved (approach): server-side environment secrets; startup validation; key identifiers and rotation; exposed Google client secret rotated before implementation | USER-SEC-003, 004 | Named owners still open |
| USER-SD-DEC-001 | **Approved (conditional, in principle):** ID-token verification with a server-issued, single-use nonce, five-minute lifetime, page-held binder (decision 027) | USER-SEC-001; USER-FR-002, 007 | Documented in security-design §4.4; a replay-protection requirement is **proposed, not added** |
| USER-SD-DEC-002 | **Approved (conditional):** HS256 if Express is the only application JWT issuer and verifier | USER-FR-026, 072; USER-SEC-002 | Consistent |
| USER-SD-DEC-025 | Approved: Supabase hosts PostgreSQL only; Express authenticates; no Supabase Auth; no Data API dependence; authentication tables must not be exposed through Supabase's HTTP APIs | USER-SEC-011, 004 | Design-level; no requirement text changed |
| USER-SD-DEC-026 | **Approved (conditional):** standard PostgreSQL driver; test connectivity first; dedicated least-privilege role separate from the database owner | USER-SEC-004 | Conditional on the connectivity test |
| USER-SD-DEC-027 | **Approved (conditional, in principle):** page-held binder, with its limitations validated | USER-SEC-001 | Documented in security-design §4.4 |
| USER-SD-DEC-028 | Approved: session-scoped CSRF token in both profiles | USER-SEC-007 | Design-level |
| USER-SD-DEC-029 | Approved: Profile X may require users to sign in again where third-party cookies are blocked; the design handles it | USER-FR-070, 077 | Consistent |
| USER-SD-DEC-022 | Immediate access-token revocation deferred | USER-FR-085 | Consistent |
| USER-SD-DEC-015 (principle) | Explicit CORS allow-list; no wildcard origin with credentialed requests | USER-SEC-003 | Origins still open |
| USER-SD-DEC-019 (principle) | Persist mandatory security events; redacted logs; no tokens, digests, credentials or secrets in logs | USER-FR-011, 028, 065, 066, 082; USER-SEC-008 | Event definitions, retention, operations still open |
| USER-SD-DEC-021 (process) | Approve each dependency individually before installation | §11.1 constraint | Consistent |
| USER-SD-DEC-023 (single tab) | One refresh at a time per browser tab; no automatic retry | USER-FR-083 | Cross-tab coordination still open |
| USER-SD-DEC-024 (policy) | A request stays rejected if a required event write fails; failures detectable and monitored | USER-FR-082 | Monitoring mechanism still open |

### 12.2 Status of the open design questions in §10
| ID | Status after Step 5 |
|---|---|
| USER-Q-001 | **Resolved.** Strategy, format, storage approach, reuse response, concurrency and the 90-day absolute session lifetime are approved. A keyed digest is a conditional follow-up (USER-SD-DEC-006), not an open question. |
| USER-Q-002 | **Open.** Rate-limit thresholds, storage, client-address identification and trusted-proxy assumptions (USER-SD-DEC-017, Open; targets proposed in security-design §11.1). |
| USER-Q-003 | **Resolved.** Multiple sessions allowed; no logout-all in the first release. |
| USER-Q-004 | **Resolved in part.** The refresh-token `Secure` `HttpOnly` cookie and in-memory access token are approved conditionally (USER-SD-DEC-013), with Profile X and Profile Y as alternatives; **no production profile is selected** until feasibility is verified. |
| USER-Q-005 to USER-Q-008, USER-PRE-001 | Unchanged. |

### 12.3 Open items that may later require a requirement
- A maximum absolute session lifetime is now approved and added as USER-FR-086 (§12.4).
- Google flow (USER-SD-DEC-001) and refresh-token transport (USER-SD-DEC-013) may affect USER-FR-001 to 011 and USER-FR-020 if they change the sign-in credential or add endpoints; any such change would be proposed for approval, not applied silently.
- **Replay protection** and **CSRF and CORS controls** are now requirements USER-SEC-014 and USER-SEC-015 (§12.7, approved under delegation).
- See §12.7 for the requirements added in Step 5d.
- **Requirements USER-SEC-014 to 016 and USER-FR-087** were added in §12.7 (Step 5d, approved under delegation).

### 12.4 Requirement added by an approved decision (USER-SD-DEC-010)
| ID | Requirement |
|---|---|
| USER-FR-086 | The **absolute session lifetime** MUST NOT exceed **90 days** (approved by the project owner, Step 5). A session's **absolute session expiry** is fixed at session creation and MUST NOT be extended by refreshing. A **refresh-token expiry** (7 days by default; USER-FR-071, 072) MUST NOT be later than the session's absolute session expiry. A refresh attempted at or after the absolute session expiry MUST be rejected under USER-FR-076, and the customer MUST sign in with Google again. Already-issued access tokens are not affected beyond their own expiry (USER-FR-074). |

| ID | Acceptance criterion | Traces to |
|---|---|---|
| USER-AC-030 | A session that is actively refreshed cannot obtain a successful refresh 90 days after its creation; a successor token's expiry never exceeds the session's absolute expiry; the absolute expiry does not change on refresh. | USER-FR-086; USER-SD-DEC-010 |

### 12.5 Terminology
| Term | Meaning |
|---|---|
| Session (refresh session) | The record created at one sign-in; the unit of revocation; the family of that sign-in's refresh tokens |
| Absolute session expiry | Session creation + 90 days; fixed; never extended by refreshing |
| Refresh-token expiry | Per-token expiry, 7 days by default, never later than the absolute session expiry |
| Access-token expiry | 15 minutes by default; stateless; independent of the session |
| Inactivity window | The refresh-token lifetime; there is no separate inactivity timer |

### 12.6 Identity decisions recorded in Step 5c
The owner approved case-insensitive email uniqueness (with no automatic merging), a unique (provider, subject) identity pair, a nullable name, required validated email, profile updates at each sign-in, and UUID identifiers. **No requirement text was changed.** They define the conflict cases of USER-FR-011 and USER-FR-066 (an email collision refuses sign-in and records the event) and are consistent with USER-FR-005, 046, 047 and USER-AC-013 and 023. Details are in data-model.md §15. One case remains open (an existing subject whose email now collides).

### 12.7 Requirements added in Step 5d (approved under owner delegation, 2026-10-25)
**No existing requirement text was changed.** The owner may override these.
| ID | Requirement |
|---|---|
| USER-SEC-014 | The service MUST accept a Google identity credential only if it carries a nonce that the service issued, that has not expired, and that has not been used before. The service MUST consume the nonce atomically on the first presentation of a validly signed credential, whether or not sign-in then succeeds, and MUST reject any later presentation without creating a session. |
| USER-SEC-015 | The service MUST reject any state-changing request authenticated by the refresh-token cookie unless its `Origin` exactly matches an allowed origin and it carries a valid session-scoped CSRF token in the `X-CSRF-Protection` header. These checks MUST complete before any refresh token is consumed or revoked. A rejected request MUST receive HTTP 403 with error code `REQUEST_REJECTED` and MUST NOT reveal which check failed. |
| USER-SEC-016 | The tables holding customer identities, sessions, refresh-token digests, sign-in nonces and security events MUST NOT be readable or writable through any HTTP API exposed by the database hosting service. The application's runtime database role MUST hold only the privileges required by the approved operations and MUST be separate from the database owner role. |
| USER-FR-087 | The service MUST allow an unauthenticated client to obtain a single-use sign-in nonce, valid for five minutes by configuration and bound to a client-supplied binder hash, and MUST rate-limit its issuance. |

| ID | Acceptance criterion | Traces to |
|---|---|---|
| USER-AC-031 | A nonce presented a second time is rejected and creates no session. | USER-SEC-014 |
| USER-AC-032 | An expired nonce is rejected. | USER-SEC-014; USER-FR-087 |
| USER-AC-033 | Two simultaneous requests carrying the same nonce produce at most one success. | USER-SEC-014 |
| USER-AC-034 | A refresh or logout failing the Origin, header or token check is rejected with 403 `REQUEST_REJECTED` and the refresh token is neither consumed nor revoked. | USER-SEC-015 |
| USER-AC-035 | A refresh or logout request with a missing `Origin` is rejected. | USER-SEC-015 |
| USER-AC-036 | Only a digest of the CSRF token is stored. | USER-SEC-015 |
| USER-AC-037 | Each protected table is unreachable through the database host's HTTP APIs, checked as each API role. | USER-SEC-016 |
| USER-AC-038 | The runtime role cannot alter schema and is separate from the owner role. | USER-SEC-016 |
| USER-AC-039 | Nonce issuance is rate-limited. | USER-FR-087; USER-SEC-009 |

**Verification.** Acceptance criteria are verified through a written acceptance-verification checklist run against a test environment (no unit tests). Creating that checklist as a documentation file is a proposed next step.
