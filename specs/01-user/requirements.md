# 01 — User & Authentication: Requirements

**Status:** Draft — awaiting review and approval
**Scope of this document:** business and functional requirements only. No API contracts, data model, or implementation.

Keyword usage: **MUST** = mandatory; **SHOULD** = recommended, deviation needs justification; **MAY** = optional. Requirements that depend on an open decision are marked **[Pending Decision: USER-DEC-xxx]** and are not finalized.

---

## 1. Purpose

The User & Authentication domain is responsible for customer identity, authentication, authenticated sessions, and account-related behavior for Varsha Homemade. It establishes *who* a caller is (a guest or a specific authenticated customer) so that other domains (Cart, Checkout, Order, Payment, Delivery, Feedback) can decide what that caller may do.

## 2. Scope

### 2.1 In scope
- Customer identity.
- Google sign-up and sign-in.
- Application authentication using JWT.
- Guest and authenticated access models.
- Authentication status and session handling.
- Account-related behavior.
- Authentication failure handling.
- Secure handling of authentication credentials and tokens.
- Authentication-related authorization requirements.

### 2.2 Out of scope
- Cart operations.
- Checkout processing.
- Order management.
- Payment processing.
- Delivery tracking.
- Customer feedback.

Other domains depend on this domain, but their business functionality is specified separately.

## 3. Customer Types

### 3.1 Guest customer
- A visitor who has not authenticated.
- MAY be allowed to browse products and use a guest cart, subject to approved decisions (USER-DEC-001, USER-DEC-002).
- MUST NOT automatically receive the permissions of an authenticated customer.
- MUST NOT be treated as having a verified customer identity.

### 3.2 Authenticated customer
- A customer who has successfully authenticated using the approved mechanism (Google).
- Has a customer identity recognized by the application.
- MUST be able to access only the functionality permitted to authenticated customers.

**Guest checkout is not assumed to be supported.** It is recorded as USER-DEC-003.

## 4. Functional Requirements

### 4.1 Google Authentication
| ID | Requirement |
|---|---|
| USER-FR-001 | A customer MUST be able to sign up or sign in using their Google account. |
| USER-FR-002 | The service MUST validate Google authentication responses or identity tokens on the server side; client-supplied identity claims MUST NOT be trusted without validation. |
| USER-FR-003 | On successful validation, the service MUST identify the customer associated with the Google account. |
| USER-FR-004 | If no customer exists for a validated Google account, the service MUST create one (sign-up); otherwise it MUST sign in the existing customer. |
| USER-FR-005 | The service MUST treat the trusted Google account identifier as the basis of identity. An email address alone MUST NOT be sufficient to establish a verified identity. |
| USER-FR-006 | The service MUST take into account the email verification status supplied by Google. Treatment of unverified emails MUST be defined before approval (see USER-DEC-014). |
| USER-FR-007 | Invalid, expired, or rejected Google credentials MUST be refused, and no authenticated application session MUST be created. |
| USER-FR-008 | A cancelled or failed Google authentication MUST result in no authenticated session and MUST leave the customer as a guest. |
| USER-FR-009 | Repeated sign-ins by the same Google account MUST resolve to the same internal customer identity. |
| USER-FR-010 | The service MUST NOT create duplicate customer identities for the same Google account. |
| USER-FR-011 | Identity conflicts (e.g., a validated identity that matches an existing customer inconsistently) MUST be handled safely: the sign-in MUST be refused rather than merged or guessed, and the conflict MUST be recorded for review. |

### 4.2 JWT Authentication
| ID | Requirement |
|---|---|
| USER-FR-020 | After successful authentication, the service MUST issue an application authentication token (JWT) that identifies the customer. |
| USER-FR-021 | Protected functionality MUST validate the JWT on every request. |
| USER-FR-022 | Invalid, expired, or improperly signed tokens MUST be rejected. |
| USER-FR-023 | A valid token MUST resolve to exactly one customer identity. |
| USER-FR-024 | A request to protected functionality without a token MUST be rejected. |
| USER-FR-025 | A malformed authorization header MUST be rejected and MUST be treated the same as missing credentials for access purposes. |
| USER-FR-026 | Token expiration MUST be enforced. Renewal behavior is **[Pending Decision: USER-DEC-006, USER-DEC-007]**. |
| USER-FR-027 | Logout and token invalidation behavior is **[Pending Decision: USER-DEC-008]**. |
| USER-FR-028 | Credentials and tokens MUST NOT appear in logs or error responses. |

Refresh tokens, token rotation, and server-side revocation are **not assumed**. JWT lifetimes and signing algorithm are **not specified** here as none are approved in the repository.

### 4.3 Guest Access
Guest behavior is undecided. The following needs are identified; none is approved.

| ID | Requirement |
|---|---|
| USER-FR-030 | The service MUST distinguish guest callers from authenticated customers on every request. |
| USER-FR-031 | Operations MUST be classified as public, guest-accessible, or authenticated-only (see §4.5). Guests MUST NOT access an operation unless it is explicitly approved as guest-accessible. |
| USER-FR-032 | Whether guests can browse products without authentication is **[Pending Decision: USER-DEC-001]**. |
| USER-FR-033 | Whether guests can create and manage a cart, how it is tied to a browser/device, and whether it persists is **[Pending Decision: USER-DEC-002, USER-DEC-004]**; detailed behavior belongs to the Cart domain. |
| USER-FR-034 | Whether a guest cart is transferred or merged after Google sign-in is **[Pending Decision: USER-DEC-005]**. |
| USER-FR-035 | Whether guest checkout is supported is **[Pending Decision: USER-DEC-003]**. |
| USER-FR-036 | If guest identifiers or cart identifiers exist, they MUST be protected so one guest cannot access another guest's data. **[Applies only if guests are given identifiers; USER-DEC-002]** |

### 4.4 Customer Identity
| ID | Requirement |
|---|---|
| USER-FR-040 | Each customer MUST be assigned a stable, unique internal customer identifier that never changes and is never reused. |
| USER-FR-041 | Each customer MUST be associated with a trusted Google account identifier. |
| USER-FR-042 | An authenticated customer MUST be able to retrieve their own identity information. |
| USER-FR-043 | Identity MUST remain consistent across repeated sign-ins. |
| USER-FR-044 | A customer's identity MUST NOT be usable to access another customer's resources. |
| USER-FR-045 | Duplicate-identity and conflict scenarios MUST be handled per USER-FR-010 and USER-FR-011. |

Unresolved: profile fields (USER-DEC-010), account linking (USER-DEC-009), account deletion and recovery (USER-DEC-011). The data model is not designed in this step.

### 4.5 Authorization
Authentication establishes *who* the customer is. Authorization determines *what* they may do.

| ID | Requirement |
|---|---|
| USER-FR-050 | Every operation in the service MUST be classified as one of: **public** (no identity needed), **guest-accessible**, or **authenticated-only**. |
| USER-FR-051 | Authenticated-only operations MUST reject guests. |
| USER-FR-052 | Customer-specific information MUST be accessible only to the customer it belongs to. |
| USER-FR-053 | The customer identity used for authorization MUST be derived from the validated token, not from customer identifiers supplied in the request. |
| USER-FR-054 | A customer MUST NOT be able to access another customer's private data by altering an identifier in a request. |

No administrator role or role-based access control is defined. Whether admin identities are in scope is USER-DEC-012.

## 5. Security Requirements

| ID | Requirement |
|---|---|
| USER-SEC-001 | Google identity assertions MUST be verified server-side for authenticity, issuer, intended audience, and expiry before being trusted. |
| USER-SEC-002 | JWTs MUST be verified for signature integrity, expiry, and other approved claims before being trusted. |
| USER-SEC-003 | Secrets and credentials (signing keys, Google client secrets) MUST be supplied through configuration, MUST NOT be committed to the repository, and MUST NOT be exposed to clients. |
| USER-SEC-004 | Access to secrets and to customer identity data MUST follow least privilege. |
| USER-SEC-005 | All authentication-related inputs MUST be validated for presence, type, format, and size before use. |
| USER-SEC-006 | Authentication error responses MUST be safe: they MUST NOT reveal credentials, tokens, secrets, internal state, or whether a specific account exists beyond what is necessary. |
| USER-SEC-007 | The service MUST protect against identity impersonation, including forged, replayed-after-expiry, or substituted tokens and identity claims. |
| USER-SEC-008 | Logs MUST NOT contain tokens, credentials, or secrets, and SHOULD minimize personal data. |
| USER-SEC-009 | The service SHOULD limit repeated failed or abusive authentication attempts. Specific controls are **[Pending Decision: USER-DEC-013]**. |
| USER-SEC-010 | In production, credentials and tokens MUST be transmitted only over encrypted transport. |

## 6. Error Handling Requirements

Outcomes only; response formats and status codes are defined later in the API contract.

| ID | Scenario | Expected outcome |
|---|---|---|
| USER-FR-060 | Google authentication fails | No session is created; the customer remains a guest; a safe, generic failure is reported. |
| USER-FR-061 | Google credentials invalid or expired | Treated as failed authentication; no session; no detail about why beyond a generic reason. |
| USER-FR-062 | JWT missing | Protected operation is refused as unauthenticated. |
| USER-FR-063 | JWT malformed | Protected operation is refused as unauthenticated. |
| USER-FR-064 | JWT invalid or expired | Protected operation is refused as unauthenticated; expiry MAY be distinguishable so clients can re-authenticate. |
| USER-FR-065 | Customer identity cannot be resolved | Access is refused; no fallback to another or guest identity; the event is logged without sensitive data. |
| USER-FR-066 | Identity conflict | Sign-in is refused; no merge; recorded for review. |
| USER-FR-067 | Access to a protected operation without authorization | Access is refused; no data about the requested resource is disclosed. |
| USER-FR-068 | All of the above | Handled consistently, with no exposure of credentials, tokens, or internal details. |

## 7. Dependencies

| Dependency | Notes |
|---|---|
| Google authentication services | Required for sign-up and sign-in. If unavailable, new sign-ins fail; guests and customers holding valid application tokens are unaffected (unless Google is needed for renewal, USER-DEC-006). |
| Application token mechanism (JWT) | Required to maintain authenticated sessions. Key compromise or misconfiguration affects all authenticated access. |
| PostgreSQL | Needed for persistent customer identity, subject to data-model design. If unavailable, sign-in and identity resolution fail. |
| Cart and other domains | Depend on this domain to distinguish guests from authenticated customers and to obtain the customer identity. |

External-service failures MUST result in a safe authentication failure, never in a granted session.

## 8. Acceptance Criteria

| ID | Criterion | Status |
|---|---|---|
| USER-AC-001 | Given a valid Google identity, the customer is authenticated and receives an application token. | Ready |
| USER-AC-002 | Given an invalid, expired, or rejected Google credential, no authenticated session is created. | Ready |
| USER-AC-003 | Repeated sign-ins with the same Google account resolve to the same internal customer identifier, with no duplicates. | Ready |
| USER-AC-004 | A valid, unexpired JWT identifies the correct customer. | Ready |
| USER-AC-005 | A missing, malformed, invalid, or expired JWT cannot access protected functionality. | Ready |
| USER-AC-006 | A guest cannot access authenticated-only functionality. | Ready |
| USER-AC-007 | Changing a customer identifier in a request does not grant access to another customer's resources. | Ready |
| USER-AC-008 | Authentication failures expose no credentials, tokens, or sensitive internal details in responses or logs. | Ready |
| USER-AC-009 | Guests can use only the capabilities explicitly approved for guests. | Pending Decision: USER-DEC-001, 002, 003 |
| USER-AC-010 | Authentication-related failures are handled consistently across all scenarios in §6. | Ready |
| USER-AC-011 | A Google account whose email is unverified is handled as defined in the approved decision. | Pending Decision: USER-DEC-014 |
| USER-AC-012 | Logout and token expiry behave as approved. | Pending Decision: USER-DEC-006, 007, 008 |
| USER-AC-013 | An identity conflict results in a refused sign-in with no merge. | Ready |
| USER-AC-014 | Repeated failed authentication attempts are limited as approved. | Pending Decision: USER-DEC-013 |

## 9. Open Questions and Decisions

Proposed defaults are recommendations only and are **not** approved requirements.

| ID | Question | Why it matters | Proposed default | Status |
|---|---|---|---|---|
| USER-DEC-001 | Can guests browse products without signing in? | Defines which operations are public. | Yes. | Open |
| USER-DEC-002 | Can guests add products to a cart? | Determines whether guest identifiers are needed and how Cart works. | Yes. | Open |
| USER-DEC-003 | Is guest checkout allowed? | Affects Checkout, Order, Payment, Delivery, and Feedback identity requirements. | No — require sign-in at checkout. | Open |
| USER-DEC-004 | Should guest carts persist across browser restarts? | Affects guest cart identification and retention. | Yes, for a limited period to be set in the Cart spec. | Open |
| USER-DEC-005 | Should a guest cart be merged into the customer's cart after Google sign-in? | Avoids losing items; needs conflict rules for duplicates. | Yes, merge on sign-in; details in Cart spec. | Open |
| USER-DEC-006 | Should both access tokens and refresh tokens be used? | Balances security and sign-in friction; affects renewal and revocation. | Short-lived access token plus refresh token. | Open |
| USER-DEC-007 | What should access-token and refresh-token lifetimes be? | Limits exposure window versus user convenience. | To be proposed with the security review once DEC-006 is decided. | Open |
| USER-DEC-008 | What logout and token-revocation behavior is required? | Determines whether tokens can be invalidated before expiry and whether server-side state is needed. | Logout invalidates the refresh token if used; access tokens expire naturally. | Open |
| USER-DEC-009 | Is account linking between multiple authentication providers required in the future? | Influences identity design (one customer, many providers). | Not now; keep identity model extensible. | Open |
| USER-DEC-010 | Which customer profile fields are mandatory? | Drives data minimization and what Google data is stored. | Only what Google supplies that is needed (name, verified email). | Open |
| USER-DEC-011 | Is account deletion or account recovery required? | Privacy obligations and link to order history retention. | Deletion/recovery out of scope for first release; revisit. | Open |
| USER-DEC-012 | Are administrative identities and permissions part of this service's scope? | Would introduce roles and a different authorization model. | No; out of scope. | Open |
| USER-DEC-013 | What rate-limiting and authentication-abuse controls are required? | Protects against brute force and abuse of sign-in. | Limit repeated failed attempts per client; specifics later. | Open |
| USER-DEC-014 | How are Google accounts with unverified email handled? | Email trust affects impersonation risk. | Refuse sign-in unless Google reports the email as verified. | Open |

## 10. Assumptions and Constraints

### 10.1 Confirmed constraints
- Node.js, Express.js, JavaScript (no TypeScript).
- PostgreSQL.
- REST APIs.
- Google authentication (Google OAuth 2.0) as the customer sign-in method.
- JWT-based application authentication.
- Configuration via environment variables.
- No additional authentication providers or technologies without approval.

### 10.2 Unapproved assumptions (need confirmation)
- Nothing here is treated as approved. Every item in §9 remains open, including guest browsing, guest carts, guest checkout, refresh tokens, token lifetimes, revocation, profile fields, account deletion/recovery, admin roles, and rate limiting.
- Acceptance criteria marked "Pending Decision" cannot be verified until the related decisions are made.
- It is assumed that Google is the only sign-in method in this release (consistent with the stated stack; account linking is USER-DEC-009).
