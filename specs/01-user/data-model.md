# 01 — User & Authentication: Data Model

**Status:** Draft — pending project owner review

This is a **logical** specification. It defines what data is needed, its constraints, and its lifecycle. It contains no SQL, migrations, schema files, or code. Physical design (exact column types, DDL, index definitions) follows approval of this document.

**Labels used for every significant design choice**
- **[Required]** — mandated by an approved specification (requirements or API contracts).
- **[Proposed]** — a recommendation for review; not approved.
- **[Unresolved]** — a decision not yet made; recorded, not decided.

Keywords follow the requirements: MUST / SHOULD / MAY.

---

## 1. Status and Scope

**In scope:** the logical PostgreSQL data needed by the User & Authentication domain: customer identity, refresh-token/session records, and security-event recording.

**Out of scope:**
- Guest-cart persistence, guest-cart identifiers, and merge (Cart domain; USER-Q-006). No guest-cart tables are defined here.
- Cart, Checkout, Order, Payment, Delivery, and Feedback schemas.
- Access tokens: they are stateless JWTs validated on every request (USER-FR-021, 074) and need no record in this model. **[Required — consistent with USER-DEC-006/008]**
- Rate-limiting state (USER-Q-002; storage not decided, no new dependency approved).
- Administrative identities, other authentication providers, account deletion/recovery workflows (USER-DEC-009, 011, 012).

## 2. Source Specifications and Traceability

| Source | Used for |
|---|---|
| [requirements.md](requirements.md) | Identity, token, privacy, and security requirements (USER-FR, USER-SEC, USER-AC, USER-DEC, USER-Q). |
| [api-contracts.md](api-contracts.md) | The four endpoints the model must support and the proposed customer response (`id`, `name`, `email`). |
| [../README.md](../README.md) | SDD rules. This data model follows rules 3 and 12 (reviewed models; distinguish confirmed from proposed). |

`specs/shared/` and `specs/decisions/` contain no conventions or decision records yet, so none are applied. Naming and type conventions below are therefore proposals.

Section 10 holds the full traceability matrix.

## 3. Entity Inventory

| Entity | Responsibility | Status |
|---|---|---|
| **Customer** | Durable customer identity bound to a Google account. | **[Required]** (USER-FR-040, 041, 048) |
| **Refresh Session** | One authenticated login lifetime for a customer; the unit of revocation. | **[Proposed]** structure; the need for revocable refresh state is **[Required]** (USER-FR-073, 075, SEC-011) |
| **Refresh Token** | One issued refresh credential within a session (stored only in non-reusable form). | **[Proposed]** structure; need is **[Required]** |
| **Security Event** | Record of security-relevant occurrences for review. | **[Proposed]** structure; the need to record is **[Required]** (USER-FR-011, 065, 066, 082) |

**[Proposed]** Session and Token are separate entities so that, if rotation is adopted (USER-FR-080, unresolved), a session can hold a chain of tokens and be revoked as a unit. If rotation is not adopted, the two collapse into one record without affecting the Customer entity or the API. **[Unresolved]** pending USER-Q-001.

No other entities are proposed. Considered and excluded: a login-history table, a profile table, a role table, and a provider-link table (none is justified by the approved requirements; USER-DEC-009 defers account linking).

## 4. Customer Entity

### 4.1 Fields

| Field (logical) | Meaning | Type (logical) | Nullability | Label |
|---|---|---|---|---|
| Customer identifier | Internal, stable identity. Immutable, unique, never reused. | Opaque unique identifier | Required | **[Required]** existence/properties (USER-FR-040). Format **[Unresolved]**; a randomly generated UUID is **[Proposed]** (non-guessable, safe to expose in tokens and responses). |
| Google account identifier | Stable Google subject identifier from validated Google identity information. | Text, case-sensitive, immutable | Required | **[Required]** (USER-FR-005, 041). Maximum length to be verified against Google's documented limit at physical design. |
| Name | Customer's display name from Google. | Text | Nullable | **[Proposed]** (see 4.4) |
| Email | Customer's email from Google. | Text | Required | **[Proposed]** required (see 4.4) |
| Created timestamp | When the customer was first created. | Timestamp with time zone (UTC) | Required | **[Proposed]** |
| Updated timestamp | When name or email last changed. | Timestamp with time zone (UTC) | Required | **[Proposed]** (needed because name/email can change; see 4.3) |

Fields intentionally **not** included: email-verified flag (sign-in already requires verified email, USER-FR-006, so the value is constant and storing it is unnecessary data), profile picture, locale, phone, address, last-login time, and any other Google profile data. USER-FR-048 forbids unnecessary profile data. **[Required]**

Shipping addresses and phone numbers needed for delivery belong to Checkout/Delivery specifications, not here.

### 4.2 Uniqueness
| Constraint | Label |
|---|---|
| Customer identifier is unique. | **[Required]** (USER-FR-040) |
| Google account identifier is unique across customers (one customer per Google account). | **[Required]** (USER-FR-009, 010) |
| Email is **not** a unique identity key. | **[Required]** (USER-FR-005, 047) |
| Whether email additionally carries a uniqueness constraint (as a data-integrity rule, not an identity key). | **[Unresolved]** — USER-DM-DEC-002 |

### 4.3 Email and name changes
- **[Required]** A change in Google name or email MUST NOT create a new customer (USER-FR-046). Identity is the Google account identifier.
- **[Proposed]** On a later successful sign-in, the stored name and email are updated from the *validated* Google identity information, and the updated timestamp is set. This keeps the profile current without changing identity. Updates are applied only from validated, email-verified identity data (USER-FR-002, 006).
- **[Unresolved]** Whether the customer's email should be updated automatically, or kept as-is until explicit confirmation, has privacy and Order-history implications (an Order may need the email at purchase time). Order domain should snapshot what it needs rather than rely on this field. USER-DM-DEC-003.

### 4.4 Why email must not be the sole identity key
**[Required rationale]** (USER-FR-005, 046, 047)
- A customer's email can change, so keying on it would either create a new customer or lose the existing one.
- An email address can be reassigned to a different person over time, so email alone cannot prove the same individual; merging by email could grant one person another's identity.
- The stable Google account identifier does not change when email or name changes.

### 4.5 Nullability implications for the approved API
The API returns `customer.id`, `customer.name`, `customer.email` and defers requiredness to this data model (api-contracts §6.4).
- `id` and `email`: always present for a stored customer (**[Proposed]** email NOT NULL, because sign-in requires a verified email).
- `name`: **[Proposed]** nullable, because Google may not supply a name. The API example shows a name string; if name is nullable, the response field could be `null` or absent. This is an implication to confirm, **not** a change to the API contract. USER-DM-DEC-001.

### 4.6 Fields returnable by `GET /api/v1/users/me`
| Field | Returnable | Label |
|---|---|---|
| Customer identifier (`id`) | Yes | **[Required]** (API contract proposed response; USER-FR-042) |
| Name (`name`) | Yes | Per API contract; nullability **[Unresolved]** |
| Email (`email`) | Yes | Per API contract |
| Google account identifier | **No** | **[Proposed]** — API contract states it is not returned; no approved need |
| Created / updated timestamps | Not returned | **[Proposed]**; the API contract does not include them |

## 5. Refresh Session and Refresh Token Entities

### 5.1 Refresh Session

| Field (logical) | Meaning | Nullability | Label |
|---|---|---|---|
| Session identifier | Unique, opaque identifier of the session. Used in security events; MUST NOT be a credential. | Required | **[Proposed]** |
| Customer | The customer who owns the session. | Required | **[Required]** relationship (customer-to-session association; USER-FR-020) |
| Created timestamp | When the session started (sign-in). | Required | **[Proposed]** |
| Revoked timestamp | When the session was revoked; null if not revoked. | Nullable | **[Required]** capability (USER-FR-073); field **[Proposed]** |
| Revocation reason | Categorical reason (for example: logout, reuse-detected). | Nullable | **[Proposed]**; the set of reasons **[Unresolved]** |
| Absolute session expiry | Optional upper bound on the session's total life. | Nullable | **[Unresolved]** — USER-DM-DEC-005 |

### 5.2 Refresh Token

| Field (logical) | Meaning | Nullability | Label |
|---|---|---|---|
| Token record identifier | Unique, opaque identifier of the token record; safe to appear in events and logs. | Required | **[Proposed]** |
| Session | The session this token belongs to. | Required | **[Proposed]** |
| Token verifier | The stored, **non-reusable representation** of the refresh token used to recognize a presented token. The raw token is never stored. Form is undecided (see 8.2). | Required | **[Required]** property (USER-SEC-011); form **[Unresolved]** |
| Issued timestamp | When the token was issued. | Required | **[Proposed]** |
| Expiry timestamp | When the token stops being valid. Derived from the configured lifetime at issue time (default 7 days; USER-FR-071, 072); the configured value MUST NOT be hard-coded. | Required | **[Required]** behavior; storing the computed expiry is **[Proposed]** (so later configuration changes do not silently alter tokens already issued; **[Unresolved]** whether that is desired, USER-DM-DEC-005) |
| Consumed timestamp | When the token was successfully used to obtain a new access token; null if not consumed. | Nullable | **[Required]** capability (USER-FR-081: a consumed token is not reusable); field **[Proposed]**; only used if rotation is adopted |
| Successor | The token issued in exchange for this one. | Nullable | **[Proposed]**; only if rotation is adopted |

### 5.3 Token validity (derived, not stored as a field)
**[Required]** A presented refresh token MUST be accepted only if all of the following are true at the time of the request:
1. It matches a stored token record.
2. It is not expired against its expiry (USER-FR-071, 072).
3. Its session is not revoked (USER-FR-073, 077).
4. It is not consumed (USER-FR-081).

If any condition fails, the request is rejected under USER-FR-076 with no disclosure of which condition failed. **A consumed refresh token MUST NOT be reusable successfully; the model provides no path by which a consumed token can be accepted.** **[Required]**

### 5.4 Support for rotation and concurrency (conceptual; no strategy selected)
**[Unresolved]** Rotation confirmation, reuse response, session-level revocation on reuse, lost-response recovery, benign-retry tolerance, and the concurrency mechanism all remain open (USER-Q-001; API-DEC-007). This section records only what the model must be able to represent.

What the model could support:
- **Consumption recorded durably.** The consumed timestamp lets the system recognize a presented, previously consumed token and reject it (USER-FR-081), and treat it as a recorded security event (USER-FR-082).
- **Single winner for concurrent requests (USER-FR-083, [Required] outcome).** The model needs a way for the transition "unconsumed → consumed" to occur at most once per token, atomically, so that two simultaneous requests cannot both succeed. Constraints on the data (for example, allowing at most one successor per token, section 6) prevent two successors from existing even if application logic fails. The *mechanism* (for example, a database-level atomic conditional update versus row locking) is a concurrency strategy and is **not selected here**.
- **Session as the unit of revocation.** Revoking a session invalidates every token in it, which would be available if session-level revocation on reuse is later approved. Whether it is used is **not decided**.
- **No grace period.** The model does not include a grace window and does not permit consumed-token reuse. Any retry or lost-response tolerance requires explicit security approval and a requirements update (api-contracts §6.2). **[Required]**

### 5.5 Security Event

Records occurrences that the requirements say must be recorded or reviewed.

| Field (logical) | Meaning | Nullability | Label |
|---|---|---|---|
| Event identifier | Unique identifier. | Required | **[Proposed]** |
| Event type | Categorical type (for example: refresh-token reuse of an invalidated token, identity conflict, unresolvable identity). The full list is **[Unresolved]**. | Required | **[Required]** need (USER-FR-011, 065, 066, 082) |
| Occurred timestamp | When it occurred. | Required | **[Proposed]** |
| Customer | Related customer, if known. | Nullable | **[Proposed]** |
| Session / token record | Related session or token record identifier, if known. | Nullable | **[Proposed]** |
| Request identifier | The API `requestId` for correlation. | Nullable | **[Proposed]** (aligns with api-contracts §3) |

**[Required]** An event MUST NOT contain a raw refresh token, access token, token verifier, Google credential, secret, or unnecessary personal data (USER-FR-028, USER-SEC-008).

**[Unresolved]** Whether security events are stored as database records, written to an application log, or both is not specified by the approved documents. The entity above is proposed on the basis that the requirement says events are "recorded" and "recorded for review" (USER-FR-011, 082). USER-DM-DEC-006.

## 6. Relationships and Constraints

### 6.1 Relationships
| Relationship | Cardinality | Label |
|---|---|---|
| Customer → Refresh Session | One customer has **zero or many** sessions; each session belongs to exactly one customer. | **[Proposed]**. The model does **not** limit a customer to one active session. Whether concurrent sessions are allowed is **[Unresolved]** (USER-Q-003). |
| Refresh Session → Refresh Token | One session has one or many token records over time (one if rotation is not adopted; a chain if it is). Each token belongs to exactly one session. | **[Proposed]**; chain length **[Unresolved]** (USER-Q-001) |
| Refresh Token → Successor | Each token has at most one successor. | **[Proposed]**; only if rotation is adopted |
| Customer / Session / Token → Security Event | An event may reference a customer, session, or token record; each reference is optional. | **[Proposed]** |

### 6.2 Keys and integrity
| Constraint | Label |
|---|---|
| Each entity has a primary key (its identifier). | **[Proposed]**; for Customer, unique and immutable identifier is **[Required]** |
| Session references Customer by a foreign key; a session cannot exist without an existing customer. | **[Proposed]** |
| Token references Session by a foreign key; a token cannot exist without an existing session. | **[Proposed]** |
| Successor references another Token record and is itself unique (at most one successor per token), so token chains cannot fork. | **[Proposed]** |
| Security Event references are optional foreign keys that do not block retention-driven removal of other data (see section 9). | **[Proposed]**; behavior on removal **[Unresolved]** |
| Customer Google account identifier is unique. | **[Required]** (USER-FR-010) |
| Token verifier is unique across token records. | **[Proposed]** (a presented token must resolve to at most one record) |
| Referential action when a customer is removed. | **[Unresolved]** — customer deletion is deferred (USER-DEC-011); no deletion behavior is defined or implied. Until decided, customers are not removed. |

### 6.3 Lifecycle constraints
- A revoked session never becomes unrevoked. **[Proposed]**
- A consumed token never becomes unconsumed. **[Required]** (USER-FR-081)
- A customer's identifier and Google account identifier never change. **[Required]** (USER-FR-040, 041)
- A token's expiry is never extended after issue. **[Proposed]**

## 7. Index Requirements and Rationale

All indexes are **[Proposed]** unless stated. Exact definitions belong to physical design.

| Index | Purpose |
|---|---|
| Customer: unique on Google account identifier | Look up the customer at sign-in; enforces no duplicates (**[Required]** outcome, USER-FR-010). |
| Customer: primary key on identifier | Resolve identity from a validated access token for `GET /users/me`. |
| Token: unique on token verifier | Look up a presented token at refresh and logout; enforces one record per verifier. |
| Token: on session | Find all tokens in a session when revoking or inspecting a session. |
| Token: unique on successor | Enforce at most one successor per token (concurrency safety; only if rotation is adopted). |
| Session: on customer | Find a customer's sessions (needed for any "all sessions" behavior if USER-Q-003 later requires it, and for referential integrity checks). |
| Token / Session: on expiry (and revoked) | Support removal of expired or revoked records under a retention rule, once retention is decided. Deferred until retention is defined. |
| Security Event: on occurred timestamp | Review recent events. |

No index on email is proposed: email is not an identity or lookup key (USER-FR-005).

## 8. Security and Privacy Considerations

### 8.1 Minimum customer data
**[Required]** Only the internal identifier, the Google account identifier, and necessary name/email are stored (USER-DEC-010, USER-FR-048). **[Proposed]** created/updated timestamps are the only additional customer fields.

### 8.2 Refresh-token storage security
**[Required]** If the token store is disclosed, it MUST NOT yield usable refresh tokens (USER-SEC-011; USER-AC-029). Raw refresh tokens MUST NOT be stored, and MUST NOT appear in logs or security events (USER-FR-028, USER-SEC-008).

Conceptual approaches (not selected):
| Approach | Notes |
|---|---|
| Store a one-way derivation (digest) of a high-entropy random token; compare on presentation. | A disclosed store contains no usable token. Token must be unguessable. Whether a keyed or unkeyed derivation is used is open. |
| Store the token encrypted (reversible). | A disclosed store plus the key yields usable tokens; weaker against the requirement unless key separation is strong. |
| Self-contained signed tokens with only an identifier stored. | Requirements state refresh tokens are not assumed to be JWTs (USER-FR-075). |

**[Unresolved]** The token format, the derivation/hashing scheme, any encryption, and key management are security-design decisions (USER-Q-001) and are not made here. USER-DM-DEC-004 records the rationale needed to decide: the choice must satisfy USER-SEC-011 and USER-AC-029, support lookup by presented token (section 7), and not require storing the raw token.

### 8.3 Access and logging
- **[Required]** Least-privilege access to token and customer data (USER-SEC-004).
- **[Required]** No tokens, credentials, or secrets in logs or events (USER-SEC-008).
- **[Proposed]** Events use opaque identifiers (customer, session, token record), not the verifier or personal data.

### 8.4 Retention (all unresolved)
No retention periods are set in this document. Nothing is deleted by the model.

| Topic | Status |
|---|---|
| Customer account deletion | **[Unresolved]**; deferred by USER-DEC-011. Privacy obligations must still be reviewed before production (USER-PRE-001). |
| Personal-data retention (name, email) | **[Unresolved]** (USER-PRE-001). |
| Order-related retention obligations | **[Unresolved]**; Order history may need customer data to persist, so customer data MUST NOT be deleted on the assumption it is unneeded. Depends on the Order specification. |
| Authentication-session retention (expired/revoked sessions and tokens) | **[Unresolved]** — USER-DM-DEC-007. Note: retaining consumed token records is what lets reuse be recognized as reuse; removing them earlier would reject the token as unknown but could not identify it as a reuse event. |
| Security-event retention | **[Unresolved]** — USER-DM-DEC-007. |

## 9. Lifecycle Behavior

| Event | Data effect | Label |
|---|---|---|
| First successful Google sign-in (new Google account) | A Customer is created; a Session and a first Token are created. | **[Required]** outcome (USER-FR-004, 020); structure **[Proposed]** |
| Repeat sign-in (same Google account) | The same Customer is resolved; no new Customer; name/email updated per 4.3; a new Session and Token are created. | **[Required]** (USER-FR-009, 010); name/email update **[Proposed]**; new session per sign-in **[Proposed]** |
| Sign-in rejected (invalid credential, unverified email, conflict) | No Customer created, no Session or Token issued. A security event is recorded for conflicts. | **[Required]** (USER-FR-006, 007, 011) |
| Successful refresh | If rotation is adopted: the presented Token is marked consumed and a successor is issued, atomically. If not: the Token is checked against validity only. | **[Unresolved]** (USER-Q-001) |
| Refresh with an invalid, expired, revoked, consumed, or unknown token | No access token issued; no state disclosed. Reuse of an invalidated token is recorded as a security event. Further effect (for example, revoking the whole session) is not decided. | **[Required]** (USER-FR-076, 082); extra effects **[Unresolved]** |
| Logout (token recognized) | The session (or applicable token) is marked revoked. | **[Required]** (USER-FR-073) |
| Logout (token missing, unrecognized, or already revoked) | No data change. | **[Proposed]**, consistent with API-DEC-008 (open) and api-contracts §6.3 |
| Logout when revocation cannot be persisted | No success reported; no partial revocation presented as complete. | **[Required]** (api-contracts §6.3) |
| Token expiry | The token becomes invalid by time alone; no write needed. Removal is subject to retention. | **[Required]** (USER-FR-071); removal **[Unresolved]** |

## 10. Requirements Traceability Matrix

| Data element / decision | Requirement(s) | API contract |
|---|---|---|
| Customer identifier, immutable, never reused | USER-FR-040, 044; USER-AC-003 | `id` in sign-in, `/users/me` |
| Google account identifier, unique | USER-FR-005, 009, 010, 041; USER-AC-003 | §6.1 (not returned) |
| Email not identity key; no email-based merge | USER-FR-005, 046, 047; USER-AC-023 | §6.1 |
| Name/email minimal storage | USER-FR-048; USER-DEC-010; USER-AC-024 | `name`, `email` fields |
| Customer-session relationship | USER-FR-020, 053 | §6.2, §6.3 |
| Configurable refresh expiry (default 7 days) | USER-FR-071, 072; USER-DEC-007; USER-AC-016, 019 | §6.2 |
| Session revocation | USER-FR-027, 073; USER-AC-017 | §6.3 |
| Consumed token not reusable | USER-FR-081; USER-AC-026 | §6.2 |
| Reuse recorded as security event | USER-FR-082; USER-AC-027 | §6.2 |
| Concurrent refresh safety | USER-FR-083; USER-AC-028 | §6.2 |
| Token store disclosure protection | USER-SEC-011; USER-AC-029 | §6.2, §5 |
| No raw tokens in logs/events | USER-FR-028; USER-SEC-008; USER-AC-008 | §7 |
| Identity conflict recorded | USER-FR-011, 066; USER-AC-013 | `409` §4 |
| Identity unresolvable | USER-FR-065 | §4 |
| Access tokens stateless; logout does not invalidate them | USER-FR-074, 084, 085; USER-AC-018 | §6.3 |
| Retention and deletion deferred | USER-DEC-011; USER-PRE-001; USER-AC-025 | — |
| Multi-session undecided | USER-Q-003 | — |

Cart/guest-cart (USER-FR-033–037) is intentionally not mapped; it belongs to the Cart specification.

## 11. Open Decisions and Dependencies

All **Open**. Proposals are recommendations only.

| ID | Decision | Proposal | Depends on |
|---|---|---|---|
| USER-DM-DEC-001 | Is customer name required or nullable? | Nullable; confirm API implication (field may be null/absent). | API contract §6.4 |
| USER-DM-DEC-002 | Does email carry a data-integrity uniqueness constraint? What is an "identity conflict" (USER-FR-011)? | Not decided. If email is unique, a second Google account with the same email would conflict. If not, it is permitted. Both affect API-DEC-013 (`409`). | API-DEC-013; product input |
| USER-DM-DEC-003 | Are name/email updated on later sign-in, and what do Orders snapshot? | Update from validated Google data; Orders snapshot their own copy. | Order specification |
| USER-DM-DEC-004 | Refresh-token format, derivation/hashing/encryption, key management. | Not decided; must meet USER-SEC-011 and USER-AC-029. | USER-Q-001 |
| USER-DM-DEC-005 | Absolute session lifetime versus per-token lifetime (with rotation, a session could otherwise extend indefinitely); whether expiry is stored or computed from configuration. | Not decided; the model can hold either. | USER-Q-001; USER-DEC-007 |
| USER-DM-DEC-006 | Are security events stored in the database, a log, or both; full event-type list. | Database record proposed; needs confirmation. | Security design |
| USER-DM-DEC-007 | Retention of expired/revoked sessions, consumed token records, and security events. | No periods proposed. | USER-PRE-001; USER-Q-001 |
| USER-DM-DEC-008 | Identifier format for Customer and records. | Randomly generated, non-guessable UUID. | Physical design |
| USER-DM-DEC-009 | Referential behavior on customer removal. | None defined; customers not removed in first release. | USER-DEC-011; Order retention |

**Carried-over unresolved items (not decided here):** rotation confirmation, reuse response, session-level revocation, concurrency mechanism and retry tolerance, lost-response recovery (USER-Q-001); multiple concurrent sessions and logout-from-all-devices (USER-Q-003); rate-limit state (USER-Q-002); guest-cart mechanism (USER-Q-006); pre-launch privacy review (USER-PRE-001).

**Dependencies:** security design (USER-Q-001/002/003); Order specification (data retention, customer data needed at purchase); Cart specification (guest-to-customer merge uses the customer identifier but is not modeled here).

## 12. Acceptance Criteria for the Data Model

IDs use `USER-DM-AC-nnn`. Each is verifiable against the physical design.

| ID | Criterion | Traces to |
|---|---|---|
| USER-DM-AC-001 | Customer has an immutable, unique identifier that is never reused. | USER-FR-040 |
| USER-DM-AC-002 | At most one Customer exists per Google account identifier, enforced by a uniqueness constraint. | USER-FR-009, 010 |
| USER-DM-AC-003 | Email is not the sole identity key and is not used to look up or merge identities. | USER-FR-005, 047 |
| USER-DM-AC-004 | Changing name or email does not create a new Customer. | USER-FR-046 |
| USER-DM-AC-005 | Only the approved customer fields (identifier, Google account identifier, name, email, created/updated timestamps) are stored. No other Google profile data is stored. | USER-FR-048; USER-DEC-010 |
| USER-DM-AC-006 | The Customer data can supply `id`, `name`, and `email` for `GET /users/me`, and the Google account identifier is not exposed. Nullability of `name` is as decided under USER-DM-DEC-001. | USER-FR-042; api-contracts §6.4 |
| USER-DM-AC-007 | Every Session belongs to an existing Customer; every Token belongs to an existing Session. | Relationships §6.1–6.2 |
| USER-DM-AC-008 | A Customer can have more than one Session unless and until a decision (USER-Q-003) restricts this. | USER-Q-003 |
| USER-DM-AC-009 | No raw refresh token is stored; the token store, if disclosed, yields no usable refresh token. | USER-SEC-011; USER-AC-029 |
| USER-DM-AC-010 | Refresh-token expiry is derived from configuration, with a default of 7 days, and no lifetime is hard-coded. | USER-FR-071, 072 |
| USER-DM-AC-011 | A token that is expired, belongs to a revoked session, or is consumed is never accepted. | USER-FR-076, 081 |
| USER-DM-AC-012 | A consumed token cannot be successfully reused, and no grace period exists unless separately approved. | USER-FR-081; api-contracts §6.2 |
| USER-DM-AC-013 | For concurrent refresh requests, at most one results in a consumed token and a single successor; no forked token chain can exist. *(Mechanism per the approved security design.)* | USER-FR-083; USER-AC-028 |
| USER-DM-AC-014 | Logout of a recognized session records revocation; a revoked refresh token cannot obtain a new access token. | USER-FR-073; USER-AC-017 |
| USER-DM-AC-015 | A failed revocation is never reported as success. | api-contracts §6.3 |
| USER-DM-AC-016 | Security events exist for identity conflict and for reuse of an invalidated refresh token, and contain no tokens, verifiers, credentials, secrets, or unnecessary personal data. | USER-FR-011, 082; USER-SEC-008 |
| USER-DM-AC-017 | No access-token records exist; logout does not claim to invalidate issued access tokens. | USER-FR-074, 084 |
| USER-DM-AC-018 | The model contains no guest-cart, Cart, Checkout, Order, Payment, Delivery, or Feedback structures. | Scope §1 |
| USER-DM-AC-019 | No data is deleted by default, and no retention period exists until decided. | USER-DEC-011; USER-PRE-001 |
