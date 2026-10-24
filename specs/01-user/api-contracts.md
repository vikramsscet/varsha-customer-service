# 01 — User & Authentication: API Contracts

**Status:** Approved for implementation planning (project owner). Decisions carry their own statuses: §9 (original), §10 (Step 5) and §11 (Step 5b). Elements not marked approved remain proposed.

Elements are **approved only where §10 or §11 records an approval**; everything else in this document is **proposed** and remains open for review. Items needing a decision are collected in §9 (`API-DEC-xxx`), and §10 and §11 record later approvals. Where a later section and an earlier one differ, the later section governs.

**Source of truth:** [requirements.md](requirements.md). This document does not change the requirements. Where a requirement cannot be represented cleanly in the API, it is flagged in §8 rather than altered.

**Not defined here:** database schema, token format/storage, application code, and APIs for Cart, Checkout, Order, Payment, Delivery, Feedback, or product browsing.

Keyword usage follows the requirements: MUST / SHOULD / MAY.

---

## 1. Proposed Endpoint Inventory

| # | Method and path | Purpose | Access category (proposed) |
|---|---|---|---|
| 1 | `POST /api/v1/auth/google` | Google sign-up / sign-in; issue application credentials | Public (credential-based; the caller is a guest until it succeeds) |
| 2 | `POST /api/v1/auth/refresh` | Obtain a new access token using a refresh token | Public (credential-based; presents a refresh token, not an access token) |
| 3 | `POST /api/v1/auth/logout` | Revoke the current refresh token or session | Public (credential-based; presents a refresh token) |
| 4 | `GET /api/v1/users/me` | Return the authenticated customer's permitted profile | Authenticated-only |

No additional endpoints are proposed. Two were considered and **not** proposed:
- *Logout from all devices* — depends on USER-Q-003, which is unresolved.
- *A Google OAuth redirect/callback endpoint* — only needed if the authorization-code flow is chosen over the ID-token flow (API-DEC-001).

**Classification note.** Requirements define three categories: public, guest-accessible, authenticated-only. Endpoints 1–3 are not tied to a guest's own resources and do not require an access token, so they are listed as *public*; their own credential (Google credential or refresh token) is what is validated. Whether this labelling is acceptable is API-DEC-009.

## 2. Conventions (Proposed)

### 2.1 General
- Base path `/api/v1`. Transport: HTTPS only in production (USER-SEC-010).
- Request and response bodies: `application/json; charset=utf-8`.
- Token-bearing responses SHOULD include `Cache-Control: no-store`.
- Field names: `camelCase`.
- Timestamps, if any: ISO 8601 UTC.
- Example identifiers and token values are illustrative placeholders. The customer identifier format is decided in the data-model step.

### 2.2 Common request headers
| Header | Used by | Notes |
|---|---|---|
| `Content-Type: application/json` | endpoints 1 (and 2, 3 if a body is used) | Required when a body is sent. |
| `Accept: application/json` | all | SHOULD be sent. |
| `Authorization: Bearer <accessToken>` | endpoint 4 | Required. Exactly one `Bearer` credential. |
| `Cookie` (refresh token) | endpoints 2, 3 | Only under the proposed cookie transport (§5). |
| `X-Request-Id` | all | OPTIONAL client-supplied correlation id; the server MAY generate one. Never carries secrets. |

### 2.3 Access token response shape
Used by endpoints 1 and 2:
```json
{
  "accessToken": "<access-token>",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```
`expiresIn` is in seconds and is derived from configuration (USER-FR-072); `900` is the approved default of 15 minutes (USER-FR-026), not a hard-coded value.

## 3. Proposed Shared Error Format

All error responses use this envelope:
```json
{
  "error": {
    "code": "AUTH_ACCESS_TOKEN_INVALID",
    "message": "Authentication is required or the credentials are not valid.",
    "requestId": "req_example_123"
  }
}
```
| Field | Rules |
|---|---|
| `error.code` | Stable, machine-readable, upper snake case. From §4. |
| `error.message` | Generic, human-readable, safe to display. MUST NOT include credentials, tokens, secrets, internal state, stack traces, or unnecessary personal or account information (USER-SEC-006). |
| `error.requestId` | Correlation identifier for support and logs. |
| `error.details` | OPTIONAL, validation errors only: list of `{ "field": "...", "issue": "..." }`. MUST NOT echo submitted credentials or token values. |

Validation example (details allowed):
```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "The request is not valid.",
    "details": [ { "field": "credential", "issue": "required" } ],
    "requestId": "req_example_123"
  }
}
```
Rate-limit responses SHOULD include a `Retry-After` header. The same error format is used regardless of endpoint (USER-FR-068).

## 4. Proposed HTTP Status and Error-Code Mapping

| Scenario | Proposed status | Proposed error code | Requirement |
|---|---|---|---|
| Body missing, not JSON, wrong type, field missing/too long | 400 | `VALIDATION_FAILED` | USER-SEC-005 |
| Google credential invalid, expired, rejected, wrong audience/issuer | 401 | `AUTH_GOOGLE_FAILED` | USER-FR-007, 061 |
| Google identity with unverified email | 401 | `AUTH_GOOGLE_FAILED` (same generic code; see API-DEC-004) | USER-FR-006, 069 |
| Customer cancelled Google flow | *No API call is made.* The client remains a guest. | — | USER-FR-008 |
| Identity conflict | 409 | `AUTH_IDENTITY_CONFLICT` | USER-FR-011, 066 |
| Access token missing | 401 | `AUTH_ACCESS_TOKEN_MISSING` | USER-FR-062 |
| Authorization header malformed | 401 | `AUTH_ACCESS_TOKEN_MISSING` (treated as missing, per USER-FR-025) | USER-FR-025, 063 |
| Access token invalid, bad signature, or expired | 401 | `AUTH_ACCESS_TOKEN_INVALID` (expiry MAY use `AUTH_ACCESS_TOKEN_EXPIRED`; API-DEC-005) | USER-FR-022, 064 |
| Customer identity cannot be resolved from a valid token | 401 | `AUTH_ACCESS_TOKEN_INVALID` | USER-FR-065 |
| Refresh token missing, malformed, invalid, unknown, expired, revoked, consumed | 401 | `AUTH_REFRESH_TOKEN_INVALID` (one generic code; reason not disclosed) | USER-FR-076, 077, 082 |
| Authenticated but not permitted to the resource | 403 | `AUTH_FORBIDDEN` | USER-FR-067 |
| Too many failed or abusive attempts | 429 | `RATE_LIMITED` | USER-SEC-009, USER-FR-078 |
| Google service or other dependency unavailable | 503 | `SERVICE_UNAVAILABLE` | Requirements §7 (safe failure, never a granted session) |
| Unexpected server error | 500 | `INTERNAL_ERROR` | USER-FR-068 |

Notes:
- **401 vs 403.** `401` means authentication failed or is missing (the caller's identity is not established). `403` means the caller is authenticated but not permitted. These are not interchangeable and are not merged.
- 401 responses for protected operations SHOULD include a `WWW-Authenticate: Bearer` header.
- Missing/malformed/invalid refresh tokens deliberately share one code and a generic message so the response does not reveal why (USER-FR-076, 082).
- **Expired-token code.** If a distinct `AUTH_ACCESS_TOKEN_EXPIRED` is approved (API-DEC-005, open), it is only a different error code for a request that has already been rejected by normal token validation. It does not bypass, relax, or replace validation, and an expired token is never accepted.
- `AUTH_FORBIDDEN` is included for completeness (USER-FR-067). `GET /users/me` returns only the caller's own data, so it does not itself produce 403.

## 5. Token Transmission Proposal (USER-Q-004 — **not approved**)

### 5.1 Proposal under evaluation
| Credential | Proposed transmission |
|---|---|
| Google credential | Request body over HTTPS (`POST /auth/google`). |
| Access token | Response body; sent on protected requests as `Authorization: Bearer <token>`. |
| Refresh token | `Secure`, `HttpOnly` cookie over HTTPS; not returned in the response body. |

This is a proposal only. It is not treated as approved.

### 5.2 Trade-offs
| Choice | Benefits | Costs / risks |
|---|---|---|
| Google credential in body | Not logged in URLs or history; simple. | Needs HTTPS; the body MUST NOT be logged (USER-SEC-008). |
| Access token in response body + Bearer header | Simple for REST clients; not sent automatically by the browser, so not exposed to CSRF. | The frontend must hold the token (typically in memory); a page script (XSS) could read it. Short 15-minute life limits exposure. |
| Refresh token in `HttpOnly` cookie | Not readable by page scripts, which reduces theft through XSS. | The browser sends it automatically, so refresh and logout become CSRF-relevant; requires cookie, CORS, and CSRF decisions; non-browser clients (e.g., mobile) need a different approach. |
| Refresh token in response body (alternative) | Works uniformly for any client; no CSRF on refresh. | Must be stored by the client in script-accessible storage, which raises theft risk; the larger exposure is why the cookie option is being evaluated. |

### 5.3 Considerations if the cookie option is adopted (decisions needed, not finalized)
- **Cookie attributes:** superseded by the two alternative profiles in §11.3 (security-design §10.3). The earlier narrow-`Path` proposal conflicts with the approved `Path=/` baseline and the `__Host-` prefix, and is no longer proposed.
- **CSRF:** refresh and logout would be cookie-authenticated state-changing requests. Candidate mitigations (to be chosen, not selected here): `SameSite` restrictions, `Origin`/`Referer` validation, a required custom request header, and a CSRF token. Choice depends on whether frontend and API are same-site or cross-site.
- **Cross-origin deployment** (frontend and API on different origins): would require an explicit CORS allow-list (no wildcard when credentials are used), `Access-Control-Allow-Credentials`, frontend requests sent with credentials, and likely `SameSite=None; Secure`, which weakens the built-in CSRF protection and makes the explicit CSRF mitigation mandatory. Third-party-cookie restrictions in browsers may also affect cross-site cookies.
- **Same-site deployment** (for example, a frontend and API on subdomains of one registrable domain): `SameSite=Lax` or `Strict` may be viable, with simpler CSRF protection.
- **Logout and rotation** both set or clear the cookie; the cookie must be cleared on logout and replaced on rotation (if confirmed).
- **Non-browser clients** are not described anywhere in the requirements; cookie transport may not suit them.

### 5.4 Uncertainty recorded
The repository does **not** establish the frontend architecture (framework, single-page or server-rendered), whether the frontend is on the same site as this API, the deployment origins, or whether non-browser clients exist. Therefore cookie attributes, the CORS policy, and the CSRF approach are **not defined**. They need the decisions API-DEC-002 and API-DEC-003.

### 5.5 How the endpoint contracts below depend on this
Endpoints 2 and 3 are written for the **proposed cookie transport**. If the body alternative is chosen instead, the refresh token would be a request/response body field and the CSRF discussion would not apply to those endpoints. The contracts will be revised accordingly.

---

## 6. Endpoint Contracts

### 6.1 `POST /api/v1/auth/google`

**Purpose.** Sign up or sign in a customer with Google. Validate the Google credential on the server, reject unverified emails, find or create the customer using the stable Google account identifier, and issue application credentials.

| Item | Contract (proposed) |
|---|---|
| Method / path | `POST /api/v1/auth/google` |
| Access category | Public (credential-based) |
| Required credentials | A Google credential in the request body. No access token. |
| Request headers | `Content-Type: application/json`, `Accept: application/json` |

**Request body**
```json
{
  "credential": "<google-identity-credential>"
}
```
- `credential`: the Google identity credential produced by Google sign-in on the client. The **ID-token flow is the recommended proposal** and is used here for illustration.
- **The request schema cannot be finalized until the frontend authentication flow is confirmed** (API-DEC-001, open). The frontend flow is not established in the repository. If the confirmed flow requires something other than an ID token (for example an authorization code), the request field(s) will change. An authorization-code callback endpoint is **not** added unless the chosen flow requires it.

**Success — `200 OK`** (proposed; `201 Created` for first sign-up is an alternative, see API-DEC-006)

Headers (cookie transport proposal): `Set-Cookie: <refresh-token cookie, attributes undecided>`; `Cache-Control: no-store`.
```json
{
  "accessToken": "<access-token>",
  "tokenType": "Bearer",
  "expiresIn": 900,
  "customer": {
    "id": "<internal-customer-id>",
    "name": "Example Customer",
    "email": "customer@example.com"
  }
}
```
The refresh token is not in the body under the cookie proposal. The `customer` fields follow data minimization (USER-FR-048); the Google account identifier is not returned.

**Error scenarios**
| Scenario | Status | Code |
|---|---|---|
| Invalid body / missing or oversized `credential` | 400 | `VALIDATION_FAILED` |
| Invalid, expired, or rejected Google credential | 401 | `AUTH_GOOGLE_FAILED` |
| Unverified Google email | 401 | `AUTH_GOOGLE_FAILED` |
| Identity conflict | 409 | `AUTH_IDENTITY_CONFLICT` |
| Rate limited | 429 | `RATE_LIMITED` |
| Google or database unavailable | 503 | `SERVICE_UNAVAILABLE` |
| Unexpected error | 500 | `INTERNAL_ERROR` |

**Example safe error**
```json
{
  "error": {
    "code": "AUTH_GOOGLE_FAILED",
    "message": "Sign-in with Google could not be completed.",
    "requestId": "req_example_123"
  }
}
```
No credential, token, Google error detail, or account information is returned.

**Validation rules**
- Body MUST be a JSON object with a non-empty string `credential` within a maximum length (value set in design). Unknown fields SHOULD be rejected or ignored (API-DEC-010).
- Server-side verification MUST confirm authenticity, issuer, intended audience, and expiry (USER-SEC-001), and MUST check that Google reports the email as verified (USER-FR-006).
- The identity key is the stable Google account identifier; email is not the sole key (USER-FR-005). A changed name or email MUST NOT create a new customer (USER-FR-046), and accounts MUST NOT be merged by email alone (USER-FR-047).
- On failure, no customer is created and no credentials are issued (USER-FR-007, 008).

**Guest-cart integration.** This endpoint, as drafted, takes no guest-cart input and returns no cart data. Cart owns guest-cart identification, merge triggering, and merge behavior, and Integration (domain 08) must verify the complete guest sign-in and cart-merge journey. No cart endpoint and no guest-cart identifier is defined here. API-DEC-012 stays open until the Cart specification defines the mechanism.

**Behavior required of the contract**
- Repeated sign-ins by the same Google account return the same internal customer identifier (USER-FR-009, 010).
- On success, an access token and a refresh token are issued (USER-FR-020).

**Security.** USER-SEC-001, 003, 005, 006, 007, 008, 009, 010; no logging of the credential or issued tokens.

**Traceability.** USER-FR-001–011, 020, 028, 040, 041, 046–048, 060, 061, 066, 069, 078; USER-SEC-001, 005, 006, 008, 009, 010; USER-AC-001, 002, 003, 008, 010, 011, 013, 014, 023, 024; USER-DEC-006, 009, 010, 013, 014.

---

### 6.2 `POST /api/v1/auth/refresh`

**Purpose.** Obtain a new access token using a valid refresh token, without repeating Google sign-in (USER-FR-070).

| Item | Contract (proposed) |
|---|---|
| Method / path | `POST /api/v1/auth/refresh` |
| Access category | Public (credential-based; no access token required) |
| Required credentials | A valid refresh token (cookie under the proposed transport) |
| Request headers | `Accept: application/json`; `Cookie` carrying the refresh token; plus any CSRF header if the cookie option is adopted (API-DEC-003) |
| Request body | None under the cookie proposal. |

**Success — `200 OK`**

Headers: `Cache-Control: no-store`; and, because rotation is approved (USER-SD-DEC-004), the replacement refresh token is returned (as `Set-Cookie` under the proposed cookie transport, which is still open under API-DEC-002).
```json
{
  "accessToken": "<access-token>",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

**Error scenarios**
| Scenario | Status | Code |
|---|---|---|
| Refresh token missing, malformed, unknown, invalid, expired, revoked, or already consumed | 401 | `AUTH_REFRESH_TOKEN_INVALID` |
| CSRF check fails (only if cookie option adopted) | 403 or 401 (API-DEC-003) | to be defined |
| Rate limited | 429 | `RATE_LIMITED` |
| Dependency unavailable | 503 | `SERVICE_UNAVAILABLE` |
| Unexpected error | 500 | `INTERNAL_ERROR` |

**Example safe error**
```json
{
  "error": {
    "code": "AUTH_REFRESH_TOKEN_INVALID",
    "message": "The session is no longer valid. Please sign in again.",
    "requestId": "req_example_123"
  }
}
```
The response MUST NOT reveal whether the token was expired, revoked, consumed, or unknown (USER-FR-076, 082).

**Validation rules**
- A refresh token MUST be present, well-formed, and within size limits (USER-SEC-005).
- It MUST be unexpired against the configured lifetime (default 7 days; USER-FR-071, 072) and not revoked.
- No client-supplied customer identifier is accepted or trusted (USER-FR-053).

**Required behaviors — with details still pending (USER-Q-001)**
| Required behavior (from requirements) | Status |
|---|---|
| Rotation is the recommended initial policy: a successful refresh consumes the presented token and issues a replacement (USER-FR-080). | **Approved** (USER-SD-DEC-004, Step 5): strict single-use rotation. The conditional wording above now applies. |
| A consumed refresh token MUST NOT be reusable (USER-FR-081). | Required. |
| Presenting an invalidated token MUST be rejected, recorded as a security event without logging the token, and disclose nothing about its prior state (USER-FR-082). | Required. The response is `401 AUTH_REFRESH_TOKEN_INVALID`. |
| Whether reuse of a consumed token also revokes the whole refresh session. | **Approved** (USER-SD-DEC-008): yes, the entire affected session. |
| Concurrent requests with the same token: at most one succeeds, with no undefined session state or more than one usable chain (USER-FR-083). | Required. **Approved** (USER-SD-DEC-007): atomic conditional update or equivalently safe transaction strategy, with consume and successor creation in one transaction; an ambiguous outcome is never treated as a confirmed rollback; the losing request returns the generic `401 AUTH_REFRESH_TOKEN_INVALID` without revealing token state. This resolves API-DEC-007. |
| Tolerance for benign client retries (for example, a lost response). | **Approved** (USER-SD-DEC-009): none. No grace period and no automatic retry of a failed refresh. |
| Lost-response recovery (a refresh succeeded on the server but the client never received the replacement token). | **Approved** (USER-SD-DEC-009): occasional sign-out is accepted. The consumed token is not reusable (USER-FR-081); the client must sign in again. |
| Session-level revocation on reuse of a consumed token. | **Approved** (USER-SD-DEC-008): the entire affected session is revoked. |
| Refresh-token format and storage; no usable refresh tokens exposed if the token store is disclosed (USER-SEC-011). | **Approved** (USER-SD-DEC-005, 006): opaque random token, not a JWT; stored as a one-way digest, unkeyed to begin; the protection requirement stands. |

**No grace period.** This contract does not define a grace period and does not permit reuse of a consumed refresh token. Any such behavior requires explicit security-design approval and a requirements update, because it would affect USER-FR-081.

**Security.** USER-SEC-005, 006, 008, 009, 010, 011; USER-FR-082, 083.

**Traceability.** USER-FR-020, 028, 070–072, 075–077, 080–083; USER-SEC-005, 006, 008, 009, 010, 011; USER-AC-015, 016, 019, 026, 027, 028, 029; USER-DEC-006, 007, 013.

---

### 6.3 `POST /api/v1/auth/logout`

**Purpose.** Log out the current session by revoking the applicable refresh token or refresh session (USER-FR-027, 073).

| Item | Contract (proposed) |
|---|---|
| Method / path | `POST /api/v1/auth/logout` |
| Access category | Public (credential-based; identifies the session by its refresh token) |
| Required credentials | The refresh token for the current session (cookie under the proposed transport). An access token is **not** required, since it may already have expired. |
| Request headers | `Accept: application/json`; `Cookie` carrying the refresh token; plus any CSRF header if the cookie option is adopted |
| Request body | None under the cookie proposal. |

**Success — `204 No Content`** (proposed)

Headers (cookie transport): a `Set-Cookie` that clears the refresh-token cookie. No body.

**Client-side logout vs. server-side revocation.** Logout has two distinct parts:
- **Client-side logout:** the client discards its local authentication state (the in-memory access token and any local session data, and, under the cookie proposal, relies on the cookie being cleared by the response). The client MUST do this **regardless of whether the server-side call succeeds or fails**, so a failed or unreachable server never leaves the user appearing signed in on that client.
- **Server-side revocation:** the service revokes the refresh token or session identified by the refresh token presented. This is the only server-side effect.

**A missing refresh token does not identify a session.** If the request carries no refresh token, the service has no session to revoke, and **no server-side revocation takes place**. The response is still `204` (proposed) so that client-side logout can complete; `204` in that case means "nothing to revoke", not "a session was revoked".

**Idempotency (proposed, API-DEC-008; open).** Logout with an invalid, expired, unrecognized, or already revoked refresh token also returns `204`, so repeated or stale logouts do not fail and do not reveal token state (USER-SEC-006). An alternative is a `401` for these cases.

**Dependency failure.** If a refresh token or session was recognized but the revocation **could not be completed** because a dependency failed, the service MUST NOT return success; it returns `503` (below). The client still clears its local state, and may retry the server-side logout.

**Error scenarios**
| Scenario | Status | Code |
|---|---|---|
| CSRF check fails (only if cookie option adopted) | 403 or 401 (API-DEC-003) | to be defined |
| Rate limited | 429 | `RATE_LIMITED` |
| Dependency unavailable (revocation could not be recorded) | 503 | `SERVICE_UNAVAILABLE` |
| Unexpected error | 500 | `INTERNAL_ERROR` |

If revocation cannot be recorded, the service MUST NOT report success (see "Dependency failure" above). The client clears local state either way.

**Example safe error**
```json
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "The request could not be completed. Please try again.",
    "requestId": "req_example_123"
  }
}
```

**Validation rules**
- The refresh token, when present, MUST be well-formed and within size limits.
- Only the refresh token or session of the current session is affected. Logout from other sessions or all devices is not defined (USER-Q-003).

**Behavior statement (required by the requirements)**
- After a successful server-side revocation, the revoked refresh token MUST NOT be usable to obtain a new access token (USER-FR-073; USER-AC-017).
- **Logout does not necessarily invalidate access tokens that were already issued.** Such tokens MAY remain valid until they expire naturally, at most the configured access-token lifetime (15 minutes by default) (USER-FR-074). Protected APIs continue to validate access tokens on every request (USER-FR-084). This contract does **not** claim that logout immediately invalidates previously issued access tokens. If immediate invalidation is required, a separate mechanism must be proposed and approved (USER-FR-085).

**Security.** USER-SEC-006, 008, 009, 010, 011, 012.

**Traceability.** USER-FR-027, 028, 073, 074, 084, 085; USER-SEC-006, 008, 009, 010, 011, 012; USER-AC-017, 018; USER-DEC-008.

---

### 6.4 `GET /api/v1/users/me`

**Purpose.** Return the authenticated customer's permitted profile information (USER-FR-042).

| Item | Contract (proposed) |
|---|---|
| Method / path | `GET /api/v1/users/me` |
| Access category | Authenticated-only |
| Required credentials | A valid, unexpired access token |
| Request headers | `Authorization: Bearer <accessToken>`, `Accept: application/json` |
| Request body | None. No customer identifier is accepted in the path, query, header, or body. |

**Success — `200 OK`**

Headers: `Cache-Control: no-store`.
```json
{
  "customer": {
    "id": "<internal-customer-id>",
    "name": "Example Customer",
    "email": "customer@example.com"
  }
}
```
Fields are limited to those needed by approved functionality (USER-DEC-010, USER-FR-048). The Google account identifier is not returned. Whether any further profile fields are needed is not established by the requirements (see API-DEC-011). **Whether each field is required or nullable is deferred to the data-model specification**; the example shows the proposed minimal response only.

**Error scenarios**
| Scenario | Status | Code |
|---|---|---|
| `Authorization` header missing or malformed | 401 | `AUTH_ACCESS_TOKEN_MISSING` |
| Token invalid, bad signature, or expired | 401 | `AUTH_ACCESS_TOKEN_INVALID` (or `AUTH_ACCESS_TOKEN_EXPIRED`, API-DEC-005) |
| Customer identity cannot be resolved | 401 | `AUTH_ACCESS_TOKEN_INVALID` |
| Rate limited | 429 | `RATE_LIMITED` |
| Dependency unavailable | 503 | `SERVICE_UNAVAILABLE` |
| Unexpected error | 500 | `INTERNAL_ERROR` |

**Example safe error**
```json
{
  "error": {
    "code": "AUTH_ACCESS_TOKEN_INVALID",
    "message": "Authentication is required or the credentials are not valid.",
    "requestId": "req_example_123"
  }
}
```

**Validation rules**
- Exactly one `Authorization: Bearer` credential. Anything else is treated as missing (USER-FR-025).
- The token MUST be verified for signature, expiry, and approved claims (USER-SEC-002).
- The customer identity MUST come from the validated token. A client-supplied customer identifier is never used to establish identity. Attempts to send one do not change which customer is returned (USER-FR-053, 054; USER-AC-007).
- A guest (no valid access token) is refused (USER-FR-051; USER-AC-006).

**Security.** USER-SEC-002, 005, 006, 007, 008, 010.

**Traceability.** USER-FR-021–025, 042, 044, 048, 051–054, 062–065, 068; USER-SEC-002, 005, 006, 007, 008; USER-AC-004, 005, 006, 007, 008, 012, 024; USER-DEC-001 (guest/public distinction), 010.

---

## 7. Security Considerations

- **No secrets in errors or logs.** Credentials, access tokens, refresh tokens, Google credentials, and signing keys MUST NOT appear in responses or logs, including request-body logging for `/auth/google` (USER-FR-028; USER-SEC-006, 008).
- **Generic failures.** Distinct internal causes are not revealed through status or message beyond the proposed codes (USER-SEC-006).
- **Server-side verification** of Google credentials and access tokens on every relevant request (USER-SEC-001, 002).
- **Identity from the server only.** The only source of customer identity for authorization is the validated access token (USER-FR-053).
- **Abuse protection.** Endpoints 1–3 (and 4) are subject to rate limiting; thresholds and scope are USER-Q-002. Rate-limit responses are generic and reveal no account information (USER-SEC-009).
- **Transport.** HTTPS only in production (USER-SEC-010).
- **Refresh-token lifecycle.** Required behaviors are in §6.2; unresolved design details are in USER-Q-001.
- **Access-token lifetime limitation.** Logout does not necessarily invalidate access tokens already issued (§6.3; USER-SEC-012). The security design must document this.
- **Token-transmission risk trade-offs** are in §5.

## 8. Requirements Traceability and Flagged Gaps

### 8.1 Coverage
| Requirement area | Covered by |
|---|---|
| Google authentication (USER-FR-001–011, 060, 061, 066, 069) | Endpoint 1 |
| Token issuance and validation (USER-FR-020–025, 062–064) | Endpoints 1, 4 |
| Refresh and rotation (USER-FR-070–072, 075–077, 080–083) | Endpoint 2 |
| Logout and access-token behavior (USER-FR-027, 073, 074, 084, 085) | Endpoint 3 |
| Identity retrieval and authorization (USER-FR-042, 044, 051–054) | Endpoint 4 |
| Security (USER-SEC-001–012) | §7 and each endpoint |
| Acceptance criteria USER-AC-001–008, 010–019, 023, 024, 026–029 | Endpoints as traced above |

### 8.2 Flagged: requirements that this API cannot cleanly represent or that need further definition
These are flagged, and the requirements are **not** changed.
1. **Guest-cart merge on sign-in (USER-FR-034, USER-AC-022).** `POST /auth/google` as drafted does not accept or reference a guest cart, and the Cart mechanism is undefined (USER-Q-006). The API contract does not decide how the merge is triggered (for example, within sign-in, or by a separate Cart call after sign-in). Cart owns guest-cart identification, merge triggering, and merge behavior, and Integration (domain 08) must verify the complete guest sign-in and cart-merge journey. API-DEC-012 remains open until the Cart specification defines the mechanism, and it needs a decision before the sign-in contract is finalized.
2. **Identity conflict (USER-FR-011, 066).** The requirements do not define what a conflict is in concrete terms. Because email is not a unique key, it is unclear whether two distinct Google accounts with the same email is a conflict, a permitted case, or neither. The `409` response is provided but the trigger cases need definition (API-DEC-013).
3. **Generic failure for unverified email (USER-FR-069).** The requirement says a generic failure. A distinct, helpful message would conflict with it, so the proposal uses the generic `AUTH_GOOGLE_FAILED`. See API-DEC-004.
4. **Distinguishing guests on every request (USER-FR-030).** This is a cross-cutting behavior for all domains' operations, not an endpoint of this domain. It is not represented as an endpoint here and is left to the shared authorization approach in design.
5. **Logout without an access token.** The requirement does not say whether logout needs a valid access token. The proposal avoids requiring it (§6.3), and this needs confirmation (API-DEC-009, 008). Related: with no refresh token presented there is no session to revoke, so `204` in that case does not mean a revocation occurred (§6.3).
6. **Status of the requirements document.** The header of requirements.md still reads "Ready for final approval — not yet approved for implementation", while this task states that it has been approved for implementation planning. This document was drafted on that instruction, and requirements.md was not modified. The header should be updated by the project owner when appropriate.

### 8.3 Open questions and assumptions
| ID | Item |
|---|---|
| Open question | Frontend architecture, deployment origins, and client types are not established (see §5.4). |
| Open question | USER-Q-001 to USER-Q-003 remain with the security and design review. |
| Assumption (unapproved) | The ID-token flow is assumed only for illustration (API-DEC-001). |
| Assumption (unapproved) | Customer identifier format is opaque and set in the data-model step. |
| Assumption (unapproved) | The contract uses a JSON error envelope and the proposed code names. |
| Assumption (unapproved) | One refresh token per session. Multi-session behavior (USER-Q-003) is not decided. |

## 9. Decisions Requiring Project Owner Approval

All Status: **Open**. Proposals are recommendations only.

| ID | Decision | Proposal in this draft |
|---|---|---|
| API-DEC-001 | Google credential type and frontend authentication flow: ID token in the body, or an authorization-code (redirect/callback) flow. The request schema cannot be finalized until this is confirmed. | ID token in a JSON body field `credential` (recommended). No callback endpoint unless the chosen flow requires one. |
| API-DEC-002 | Refresh-token transport (resolves USER-Q-004): `HttpOnly` cookie, response body, or other; and the supported client types. | `Secure`, `HttpOnly` cookie. Needs deployment and client information. |
| API-DEC-003 | If cookies are used: cookie attributes (`SameSite`, `Path`, name), CORS policy, CSRF mitigation, and the status code for a failed CSRF check. | Not finalized; depends on API-DEC-002 and deployment origins. |
| API-DEC-004 | Whether an unverified email gets a distinct error code or the same generic `AUTH_GOOGLE_FAILED`. | Same generic code, consistent with USER-FR-069. |
| API-DEC-005 | Whether an expired access token is distinguishable (`AUTH_ACCESS_TOKEN_EXPIRED`) so clients can refresh. | Distinguishable (permitted by USER-FR-064). |
| API-DEC-006 | Success status for first sign-up: `200` always, or `201` for newly created customers. | `200` always, so the response does not reveal whether the account is new. |
| API-DEC-007 | Response for the losing request in a concurrent refresh, and tolerance for benign retries. | `401 AUTH_REFRESH_TOKEN_INVALID`; tolerance left to USER-Q-001. |
| API-DEC-008 | Logout with a missing, invalid, unrecognized, or already revoked refresh token: `204` (idempotent) or `401`. A dependency failure while revoking a recognized session is always non-success (`503`). | `204` for the former; `503` for dependency failure. |
| API-DEC-009 | The access-category label for endpoints 1–3 (public, credential-based), and that logout does not require an access token. | As drafted. |
| API-DEC-010 | Handling of unknown request fields (reject or ignore) and maximum field sizes. | Decide in design. |
| API-DEC-011 | Profile fields returned by `GET /users/me`. | `id`, `name`, `email`. |
| API-DEC-012 | How guest-cart merge is triggered relative to sign-in (see §8.2 item 1). Cart owns identification, triggering, and merge behavior; Integration verifies the full journey. | **Open** until the Cart specification defines the mechanism. No cart endpoint or guest-cart identifier is proposed here. |
| API-DEC-013 | Definition of the identity-conflict scenarios that produce `409`. | Not defined; needs input. |
| API-DEC-014 | Approval of the shared error envelope (§3) and the status/error-code table (§4). | As drafted. |
| API-DEC-015 | Approval of the four-endpoint inventory, and that logout-from-all-devices is excluded pending USER-Q-003. | As drafted. |

Related items still held by other steps: USER-Q-001 (refresh-token format, storage, rotation confirmation, reuse response, concurrency, retry tolerance), USER-Q-002 (rate-limit thresholds), USER-Q-003 (concurrent sessions, logout-all), USER-Q-005 (operation lists), USER-Q-006 (guest cart mechanism), USER-Q-008 (data model).

## 10. Security-Design Decision Updates (Step 5, 2026-10-24)

Decisions approved by the project owner that affect these contracts. Full detail is in [security-design.md](security-design.md) §3. **The endpoint inventory is unchanged, and no endpoint was added.** A formatting defect in §6.2 (a table row left after the "No grace period" paragraph) was repaired by moving that row back into its table; its meaning is unchanged apart from the approvals recorded above.

### 10.1 API decisions
| ID | Status after Step 5 | Note |
|---|---|---|
| API-DEC-001 | **Approved (conditional)** | Google ID-token flow with a server-issued, single-use nonce, five-minute lifetime (USER-SD-DEC-001, in principle). Documented in §11.1 and §11.2 for final review. |
| API-DEC-002 | **Approved (conditional)** | Refresh-token transport (USER-SD-DEC-013): `Secure`, `HttpOnly` refresh cookie and in-memory access token. **Profile X (current Render origins) and Profile Y (custom domains or proxy) are alternatives; no production profile is selected** until feasibility is verified (§11.3). |
| API-DEC-003 | **Approved** | CORS allow-list principle and CSRF baseline: `X-CSRF-Protection`, `403 REQUEST_REJECTED`, Origin validation, missing `Origin` rejected on refresh and logout, session CSRF token (USER-SD-DEC-015, 016, 028). Cookie attributes per profile (§11.3) remain conditional. |
| API-DEC-004 | Open | Unchanged |
| API-DEC-005 | Open | Unchanged |
| API-DEC-006 | Open | Unchanged |
| API-DEC-007 | **Approved** | The losing request in a concurrent refresh returns the generic `401 AUTH_REFRESH_TOKEN_INVALID` (USER-SD-DEC-007). Benign retries are not tolerated (USER-SD-DEC-009). |
| API-DEC-008 | Open | `204` versus `401` for logout with a stale or unknown token is unchanged. Approved: logout revokes the applicable refresh session; a revocation that cannot be persisted is never reported as success (USER-SD-DEC-012). |
| API-DEC-009 to 015 | Open | Unchanged, except API-DEC-015: the inventory is now documented with **five** endpoints (§11.5), pending final approval. API-DEC-012 (guest-cart merge) stays with the Cart specification. |

### 10.2 Contract behavior now approved
- `POST /auth/refresh` is a strict single-use rotation: each success consumes the presented token and returns a replacement (USER-SD-DEC-004).
- Presenting a consumed refresh token returns the generic refresh error, records a security event, and revokes the whole session, after which no token in that session refreshes (USER-SD-DEC-008).
- No grace period, and the client must not automatically retry a failed refresh; clients make one refresh request at a time per tab (USER-SD-DEC-009, 023).
- `POST /auth/logout` ends only the current session; there is no logout-all endpoint in the first release (USER-SD-DEC-011, 012).
- Already-issued access tokens may stay valid until their expiry, up to 15 minutes by default (USER-SD-DEC-012, 022).
- Access-token claims are issuer, audience, subject, issued-at and expiry (USER-SD-DEC-003).
- A refresh attempted at or after the session's absolute expiry (90 days after sign-in, USER-SD-DEC-010) returns the generic refresh error and the customer signs in again. Refreshing never extends the session, and a successor token's expiry never exceeds the session's absolute expiry. This adds no field and no endpoint.

### 10.3 Still open and affecting the contracts
Sign-in request schema (API-DEC-001), refresh-token transport and cookie behavior (API-DEC-002, 003; note that §5's cookie proposal was written before the confirmed origins were found to be cross-site), logout status for stale tokens (API-DEC-008), the status code for a failed CSRF check, and whether a nonce endpoint or callback endpoint is added (neither is approved).

### 10.4 Contract changes the server-issued nonce would require (NOT approved)
The recommendation for decision USER-SD-DEC-001 is the Google ID-token flow with a server-issued, single-use nonce. It is **not approved**, and **nothing in §1 to §9 has been changed**. If you approve it, the contract would change as follows, for review before any implementation.

| Item | Proposed change |
|---|---|
| New endpoint (provisional) | `POST /api/v1/auth/google/nonce`. Access category: public. No request body. |
| Success | `200 OK` with `Cache-Control: no-store` and a body such as `{ "nonce": "<server-issued-nonce>", "expiresIn": <seconds, from configuration> }`. The value is random, single-use and short-lived. |
| Errors | `429 RATE_LIMITED`, `503 SERVICE_UNAVAILABLE`, `500 INTERNAL_ERROR`, in the shared error format (§3). |
| `POST /api/v1/auth/google` | No new required field is proposed. The credential's nonce claim must match a live, unconsumed server-issued nonce, which is consumed on first presentation whether sign-in succeeds or fails. An absent, unknown, expired or consumed nonce returns the generic `401 AUTH_GOOGLE_FAILED`. Whether an optional client-binding handle is added is open. |
| Endpoint inventory (§1) | Grows from four to five endpoints; this affects API-DEC-015. |
| Cross-cutting | The new endpoint is rate-limited and covered by the CORS allow-list. Nonce state is not in the approved data model (data-model §13.6). |

### 10.5 Contract changes from the approved CSRF, CORS and cookie decisions
Recorded for contract review; superseded in detail by §11, which writes these controls into the contract.
- **Refresh and logout:** require a custom request header (name not chosen; add it to §2.2 when approved) and a valid `Origin` that matches the configured frontend origin. A request failing either check is rejected **before** any token is consumed, with a generic response that reveals no token state. The status code and error code are open (API-DEC-003).
- **CORS:** `Access-Control-Allow-Origin` names the exact allowed origin `https://varsha-homemade.onrender.com` (not a wildcard); credentials are allowed only when the cookie strategy is in effect; allowed methods `GET`, `POST`, `OPTIONS`; allowed headers `Authorization`, `Content-Type` and the custom header; responses vary on `Origin`.
- **Cookie:** the baseline replaces the open attribute list in §5.3, conditional on API-DEC-002. `Path=/` means the cookie is sent on every request to the API host, which accepts it only on refresh and logout.
- **No other endpoint is added.**

## 11. Authentication Transport, Nonce and CSRF Contract (Step 5b, 2026-10-24)
Records the contract effect of decisions USER-SD-DEC-001, 013, 014, 015, 016, 027, 028, 029. Full design and rationale are in [security-design.md](security-design.md) §4.4 and §10. **No production profile is selected.** This section extends §3, §4 and §6; where it differs from them, this section governs.

### 11.1 New endpoint: `POST /api/v1/auth/google/nonce` (approved in principle; for final review)
**Why it exists:** only the server can know that a nonce was issued and used exactly once.

| Item | Contract |
|---|---|
| Purpose | Issue a short-lived, single-use nonce for one Google sign-in attempt |
| Access category | Public. No access token and no cookie credential. |
| Request headers | `Content-Type: application/json`, `Accept: application/json` |
| Request body | `{ "binderHash": "<one-way hash of the page-held binder>" }` (hash algorithm and encoding are fixed at implementation planning; SHA-256 with URL-safe encoding is proposed) |
| Validation | `binderHash` is required, a string, within length bounds, and uses the expected character set |
| Success | `200 OK`, `Cache-Control: no-store`, body `{ "nonce": "<server-issued-nonce>", "expiresIn": 300 }`. The lifetime is five minutes and is read from configuration. |
| Errors | `400 VALIDATION_FAILED`, `429 RATE_LIMITED`, `503 SERVICE_UNAVAILABLE`, `500 INTERNAL_ERROR`, in the shared error format (§3) |
| Security | The nonce is random from a cryptographically secure source. It is never logged. The endpoint is rate-limited and caps outstanding nonces per client. |
| Cookies and CSRF | None; it does not use the refresh cookie |
| Traceability | USER-SD-DEC-001, 027; security-design §4.4 |

### 11.2 `POST /api/v1/auth/google`: changes
- **Request:** `{ "credential": "<google-id-token>", "binder": "<raw page-held binder>" }`. `binder` is required and validated.
- **Processing order:** verify the Google token (signature, issuer, audience, expiry, verified email) → atomically consume the nonce carried in the token → check that the binder hashes to the stored hash → find or create the customer by Google `sub` → create the session.
- **Failure:** an absent, unknown, expired or consumed nonce, or a wrong binder, returns the generic `401 AUTH_GOOGLE_FAILED`, creates no session, and records a security event without the credential.
- **Success response** gains one field: `csrfToken`, the session-scoped CSRF token. It is returned **once** and never again.

```json
{
  "accessToken": "<access-token>",
  "tokenType": "Bearer",
  "expiresIn": 900,
  "csrfToken": "<session-csrf-token>",
  "customer": { "id": "<internal-customer-id>", "name": "Example Customer", "email": "customer@example.com" }
}
```
The refresh token is set as a cookie (§11.3). Customer field nullability remains open (USER-DM-DEC-001, 010, 011).

### 11.3 Cookie and CORS behavior by alternative profile
Alternatives, not simultaneously active. Neither is production-ready until its prerequisites are verified.
| | Profile X: current Render origins (cross-site) | Profile Y: custom domains (Y1) or same-origin proxy (Y2) |
|---|---|---|
| Refresh cookie | `Secure; HttpOnly; SameSite=None`, host-only | `Secure; HttpOnly; SameSite=Lax`, host-only; `Path=/` allows `__Host-` |
| `Path` | `/` or a restricted path | `/` (Y1); a restricted path advisable for Y2 |
| CORS | Credentialed; exact origin `https://varsha-homemade.onrender.com`; `Vary: Origin`; allowed headers `Authorization`, `Content-Type`, `X-CSRF-Protection` | Y1: credentialed for the new frontend origin. Y2: none |
| Browser risk | Browsers that block third-party cookies may prevent silent refresh; a CSRF token does not fix that (security-design §10.6) | Cookie not third-party |

**Profile X behavior the contract must allow:** if the refresh cookie is not stored or not sent, `POST /auth/refresh` returns the generic `401 AUTH_REFRESH_TOKEN_INVALID`, and the customer signs in again.

### 11.4 CSRF controls on refresh and logout
`POST /auth/refresh` and `POST /auth/logout` are cookie-authenticated and state-changing, so they require:
- `Origin` present and exactly equal to the configured frontend origin;
- `X-CSRF-Protection: <csrfToken>` present and matching the session's CSRF token (compared by digest);
- the refresh cookie.

Checks run **before any token is consumed** or revoked. A failure returns:
```json
{ "error": { "code": "REQUEST_REJECTED", "message": "The request could not be completed.", "requestId": "req_example_123" } }
```
with HTTP `403`. The response is identical for every cause (missing `Origin`, wrong `Origin`, missing header, wrong token), and is not retried by the client. A missing refresh cookie is the generic `401 AUTH_REFRESH_TOKEN_INVALID`. **A missing `Origin` is rejected** on these two endpoints.

Not cookie-authenticated, so no CSRF controls: `POST /auth/google`, `POST /auth/google/nonce`, `GET /users/me`.

Headers added to the tables in §2.2 for refresh and logout: `Origin` (set by the browser) and `X-CSRF-Protection`.

### 11.5 Error codes and endpoint inventory
Extends the §4 mapping:
| Scenario | Status | Code |
|---|---|---|
| Cookie-authenticated request fails the Origin, header or CSRF-token check | 403 | `REQUEST_REJECTED` |

**Endpoint inventory (API-DEC-015, pending final approval):** `POST /auth/google/nonce` (new), `POST /auth/google`, `POST /auth/refresh`, `POST /auth/logout`, `GET /users/me`. No logout-all endpoint.

### 11.6 Still open and affecting the contract
Sign-in request and response schema final review (API-DEC-001); the unverified-email code, expired-token code, 200 versus 201 and logout status for stale tokens (API-DEC-004 to 008); access categories, field limits, profile fields, identity-conflict triggers and the error-envelope approval (API-DEC-009 to 014); which profile is used in production.

## 12. Identity, Profile and Conflict Contract (Step 5c, 2026-10-25)
Records the contract effect of the owner decisions in data-model §15. Where §6 and this section differ, this section governs.

### 12.1 Status of the API decisions affected
| ID | Status | Note |
|---|---|---|
| API-DEC-011 | **Approved** | `GET /users/me` returns `id`, `name`, `email`. `name` is `null` (key present) when unavailable. `email` is always a string. |
| API-DEC-013 | **Approved** | An identity conflict is an email collision for an unknown subject (below). For an existing subject whose email now collides, sign-in succeeds with the stored email kept and an event recorded (Step 5d, delegated). |
| API-DEC-005, 006, 008, 009, 010, 014, 015 | **Approved (delegated, 2026-10-25)** | See §13. Numeric size limits remain open. |

### 12.2 Profile response
```json
{ "customer": { "id": "<internal-customer-id>", "name": null, "email": "customer@example.com" } }
```
The same `customer` object is returned by sign-in (§6.1, §11.2). The identifier is a UUID. The Google subject and provider are never returned.

### 12.3 Identity conflict on sign-in
When the validated subject is unknown and the case-insensitive email is held by another customer, `POST /api/v1/auth/google` returns:
```json
{ "error": { "code": "AUTH_IDENTITY_CONFLICT", "message": "The sign-in could not be completed with this account.", "requestId": "req_example_123" } }
```
with HTTP `409`. No customer is created, no session or token is issued, nothing is merged, and the response does not reveal the other account. The mandatory identity-conflict event is recorded. The frontend treats it as non-retryable.

## 13. Decisions Taken Under Owner Delegation (Step 5d, 2026-10-25)
The project owner asked for the open decisions to be decided. Each is recorded as **Approved (delegated)**: decided by the assistant under that delegation, with the reasoning below. **The owner may override any of them.** None changes an earlier approved decision.

| ID | Decision | Reason and security note |
|---|---|---|
| API-DEC-005 | An expired access token returns `401` with the distinct code `AUTH_ACCESS_TOKEN_EXPIRED`. It is still rejected by normal validation. | Lets the frontend refresh once; discloses only expiry (USER-FR-064 allows it) |
| API-DEC-006 | Sign-in returns `200` for new and returning customers. | Does not reveal whether an account is new |
| API-DEC-008 | Logout with a missing, unknown, expired or already revoked refresh token returns `204`. If a recognized session cannot be revoked because of a dependency failure the response is `503`. With no cookie, the Origin and header checks still apply and the result is `204`. | Idempotent; reveals no token state |
| API-DEC-009 | Endpoints 1 to 3, and the nonce endpoint, are labelled **public (credential-based)**; logout needs no access token. | The access token may already have expired |
| API-DEC-010 | Unknown request fields are **rejected**: `400` with `VALIDATION_FAILED` and `details` of `{ "field": "(request body)", "issue": "unknown_field" }`; the unknown key is not echoed. Malformed bodies stay `400`. For cookie-authenticated endpoints the Origin and CSRF checks run first (403), then body validation (400). **Numeric size limits are set at implementation planning from the observed Google token size and remain open.** | Strict input (USER-SEC-005); avoids reflecting input |
| API-DEC-014 | The shared error envelope is approved as drafted (`code`, `message`, `requestId`, optional `details`). **`requestId` is generated by the server;** a client-supplied value is ignored. `REQUEST_REJECTED` is added to the code table. | Prevents log injection |
| API-DEC-015 | The inventory is **five endpoints**: the nonce endpoint, sign-in, refresh, logout, `GET /users/me`. No logout-all endpoint. | Follows the approved nonce design |
`API-DEC-011` and `API-DEC-013` are resolved (§12). `API-DEC-012` (guest-cart merge) stays with the Cart specification.
