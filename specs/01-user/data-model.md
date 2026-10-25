# 01 — User & Authentication: Data Model

**Status:** Approved for implementation planning

Approval of this data model does **not** approve any unresolved decision in Section 11, and does **not** finalize the physical database schema. All Section 11 decisions remain open.

This is a **logical** specification. It defines what data is needed, its constraints, and its lifecycle. It contains no SQL, migrations, schema files, or code. Physical design (exact column types, DDL, index definitions) follows approval of this document.

**Labels used for every significant design choice**
- **[Required]** — mandated by an approved specification (requirements or API contracts).
- **[Proposed]** — a recommendation for review; not approved.
- **[Unresolved]** — a decision not yet made; recorded, not decided.

Keywords follow the requirements: MUST / SHOULD / MAY.

---

## 1. Status and Scope

**In scope:** the logical PostgreSQL data needed by the User & Authentication domain: customer identity, refresh-token/session records, sign-in nonce records, and security-event recording.

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
| [api-contracts.md](api-contracts.md) | The five endpoints the model must support (including the nonce endpoint) and the proposed customer response (`id`, `name`, `email`). |
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
| **Sign-in Nonce** | One short-lived, single-use server-issued nonce for a Google sign-in attempt (§14.1). | **[Approved in principle]** (USER-SD-DEC-001; USER-DM-DEC-014); need **[Required]** (USER-SEC-014, USER-FR-087) |

**[Proposed]** Customer, Refresh Session, and Refresh Token are kept as separate logical entities. A session is the unit of revocation and can hold one or more token records. A **token chain is not mandatory**: it exists only if rotation is approved (USER-FR-080, **[Unresolved]** under USER-Q-001). If rotation is not approved, a session may hold a single token record, or the two entities may be merged at physical design, without affecting the Customer entity or the API. The separation does not prejudice the approved refresh-token security requirements (section 5.3).

**[Proposed]** The logical Refresh Token entity describes the behavior and state needed to manage refresh credentials; it does not mandate a separate physical table. The final physical representation must support all approved refresh-token security requirements. Fields such as consumption state and successor relationships apply only where required by the approved strategy. Any simplification or merging of entities must preserve revocation, expiry, concurrency safety, invalidated-token event recording, and token-store disclosure protection.

No entities beyond these five are proposed. Considered and excluded: a login-history table, a profile table, a role table, and a provider-link table (none is justified by the approved requirements; USER-DEC-009 defers account linking).

## 4. Customer Entity

### 4.1 Fields

| Field (logical) | Meaning | Type (logical) | Nullability | Label |
|---|---|---|---|---|
| Customer identifier | Internal, stable identity. Immutable, unique, never reused. | Opaque unique identifier | Required | **[Required]** existence/properties (USER-FR-040). Format **[Unresolved]**; a randomly generated UUID is **[Proposed]** (non-guessable, safe to expose in tokens and responses). |
| Identity provider | The provider that authenticated the customer; `google` in the first release | Text | Required | **[Approved]** (Step 5c; USER-DEC-009 limits the first release to Google) |
| Provider subject | Stable Google subject identifier from validated Google identity information; the external identity key | Text, case-sensitive, immutable | Required | **[Required]** (USER-FR-005, 041). Maximum length to be verified against Google's documented limit at physical design. |
| Name | Customer's display name from Google | Text | Nullable | **[Approved]** (Step 5c): NULL when Google provides no usable name (empty after trimming); never invented; returned as `"name": null` (§15.3) |
| Email | Customer's email from Google, stored only after the Google ID token and its verified-email claim are validated | Text, stored exactly as Google supplies it | Required | **[Approved]** (Step 5c): required; unique case-insensitively as a data-integrity rule, never an identity key (§15.4) |
| Created timestamp | When the customer was first created. | Timestamp with time zone (UTC) | Required | **[Proposed]** |
| Updated timestamp | When name or email last changed. | Timestamp with time zone (UTC) | Required | **[Proposed]** (needed because name/email can change; see 4.3) |

Fields intentionally **not** included: email-verified flag (sign-in already requires verified email, USER-FR-006, so the value is constant and storing it is unnecessary data), profile picture, locale, phone, address, last-login time, and any other Google profile data. USER-FR-048 forbids unnecessary profile data. **[Required]**

Shipping addresses and phone numbers needed for delivery belong to Checkout/Delivery specifications, not here.

### 4.2 Uniqueness
| Constraint | Label |
|---|---|
| Customer identifier is unique. | **[Required]** (USER-FR-040) |
| The pair (identity provider, provider subject) is unique across customers (one customer per Google account) | **[Required]** (USER-FR-009, 010); pair form **[Approved]** (Step 5c) |
| Email is **not** a unique identity key. | **[Required]** (USER-FR-005, 047) |
| Email is unique **case-insensitively** as a data-integrity rule, **not** as an identity key; no automatic account merging | **[Approved]** (Step 5c; USER-DM-DEC-002; implications in §15.4) |

### 4.3 Email and name changes
- **[Required]** A change in Google name or email MUST NOT create a new customer (USER-FR-046). Identity is the Google account identifier (the approved meaning is the pair provider and subject; see §15).
- **[Proposed]** On a later successful sign-in, the stored name and email are updated from the *validated* Google identity information, and the updated timestamp is set. This keeps the profile current without changing identity. Updates are applied only from validated, email-verified identity data (USER-FR-002, 006).
- **[Unresolved]** Whether the customer's email should be updated automatically, or kept as-is until explicit confirmation, has privacy and Order-history implications (an Order may need the email at purchase time). Order domain should snapshot what it needs rather than rely on this field. USER-DM-DEC-003.

### 4.4 Why email must not be the sole identity key
**[Required rationale]** (USER-FR-005, 046, 047)
- A customer's email can change, so keying on it would either create a new customer or lose the existing one.
- An email address can be reassigned to a different person over time, so email alone cannot prove the same individual; merging by email could grant one person another's identity.
- The stable Google account identifier does not change when email or name changes.

### 4.5 Nullability implications for the approved API

The API returns `customer.id`, `customer.name`, and `customer.email`. The API contract (api-contracts.md, §6.4) leaves field nullability to this data model.

- `id`: Always present for a stored customer. The customer identifier is required.
- `name`: **[Proposed]** The logical field is nullable because the approved requirements do not guarantee that Google always supplies a name. The API example shows a name string, but the contract does not specify how a missing name is represented. How `GET /api/v1/users/me` represents a missing name remains an open decision (USER-DM-DEC-010): `name: null`, omission of the field, or another explicitly agreed representation. This document does not change the API contract; if the selected behavior differs from what the contract already permits, the contract must be amended through its own review.
- `email`: **[Proposed]** The Customer record requires an email because sign-in requires a verified email (USER-FR-006). However, making email non-null in the data model remains subject to confirmation of the identity-validation and profile requirements (USER-DM-DEC-011). Until this decision is approved, the model does not establish an unconditional guarantee that email is always present. If email is not required, the API contract must explicitly define how a missing email is represented.

The proposed email constraint and the API's response guarantees must be consistent with the final approved decisions. No unresolved nullability decision is settled by this section.

### 4.6 Fields returnable by `GET /api/v1/users/me`
| Field | Returnable | Label |
|---|---|---|
| Customer identifier (`id`) | Yes | **[Required]** (API contract proposed response; USER-FR-042) |
| Name (`name`) | Yes | Per API contract; nullability **[Unresolved]** |
| Email (`email`) | Yes | Per API contract |
| Provider subject | **No** | **[Proposed]**: API contract states it is not returned; no approved need |
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
| CSRF-token digest | One-way digest of the session-scoped CSRF token (§14.2); the raw value is returned once at sign-in and never stored | Required | **[Approved]** (USER-SD-DEC-028) |
| Absolute session expiry | Session creation + 90 days; fixed at creation; never extended by refreshing | Required | **[Required]** (USER-SD-DEC-010 approved). Whether it is stored at creation or computed from configuration is **[Unresolved]** (USER-DM-DEC-005 remainder) |

### 5.2 Refresh Token

| Field (logical) | Meaning | Nullability | Label |
|---|---|---|---|
| Token record identifier | Unique, opaque identifier of the token record; safe to appear in events and logs. | Required | **[Proposed]** |
| Session | The session this token belongs to. | Required | **[Proposed]** |
| Token verifier | A one-way digest of the refresh token; the raw token is never stored. The digest algorithm and whether it is keyed remain open (USER-SD-DEC-006; USER-DM-DEC-004). | Required | **[Required]** property (USER-SEC-011); the stored-digest form is **[Approved]** (USER-SD-DEC-005, 006) |
| Issued timestamp | When the token was issued. | Required | **[Proposed]** |
| Expiry timestamp | When the token stops being valid. Derived from the configured lifetime at issue time (default 7 days; USER-FR-071, 072); the configured value MUST NOT be hard-coded. | Required | **[Required]** behavior; storing the computed expiry is **[Proposed]** (so later configuration changes do not silently alter tokens already issued; **[Unresolved]** whether that is desired, USER-DM-DEC-005) |
| Consumed timestamp | When the token was successfully consumed; null if not consumed. | Nullable | The **outcome** that a consumed token is never reusable is **[Required]** (USER-FR-081). This field is a **[Proposed]** way to record consumption. What counts as "consumed" depends on the refresh strategy, which is **[Unresolved]** (USER-Q-001). |
| Successor | The token issued in exchange for this one. | Nullable | **[Approved]** (rotation approved, USER-SD-DEC-004). |

### 5.3 Token validity (derived, not stored as a field)
**[Required]** A presented refresh token MUST be accepted only if all of the following are true at the time of the request:
1. It matches a stored token record.
2. It is not expired against its expiry (USER-FR-071, 072).
3. Its session is not revoked (USER-FR-073, 077).
4. It is not consumed (USER-FR-081).

If any condition fails, the request is rejected under USER-FR-076 with no disclosure of which condition failed. **A consumed refresh token MUST NOT be reusable successfully; the model provides no path by which a consumed token can be accepted.** **[Required]**

Presentation of an invalidated (consumed, revoked, or superseded) token MUST also be recorded as a security event, without logging the token (USER-FR-082). **[Required]** A revoked session MUST NOT be usable to refresh credentials (USER-FR-073, 077). **[Required]**

These required outcomes apply to **any** refresh-token strategy, rotating or not.

### 5.4 Support for rotation and concurrency (conceptual; no strategy selected)

**[Unresolved]** Refresh-token rotation, reuse response, session-level revocation on reuse, lost-response recovery, benign-retry tolerance, and the concurrency mechanism remain open (USER-Q-001; API-DEC-007). This section specifies required outcomes without selecting a token strategy or implementation mechanism.

**Required outcomes, regardless of the eventual strategy:**

- A consumed refresh token cannot be reused successfully (USER-FR-081).
- Concurrent refresh requests must not produce more than one successful use of the same token (USER-FR-083).
- Presentation of an invalidated token must be recorded as a security event (USER-FR-082).
- A revoked session cannot be used to refresh credentials (USER-FR-073, 077).
- The token store must not yield usable refresh tokens if disclosed (USER-SEC-011; USER-AC-029).

**Security-design dependency:**

A reusable, non-rotating refresh token presents a potential conflict with the requirements that a consumed token cannot be reused and that concurrent requests cannot successfully use the same token more than once. If the token remains reusable, these requirements may not be satisfied. If it is permanently consumed after one successful use, the client needs an approved way to obtain a replacement before refreshing again.

Therefore, a non-rotating design MUST NOT be approved merely because it validates the token's identity, expiry, and session status. Before implementation, the security design MUST demonstrate how the proposed strategy satisfies every approved refresh-token security requirement, including USER-FR-073, USER-FR-081, USER-FR-082, USER-FR-083, USER-SEC-011, and USER-AC-029.

If no viable non-rotating strategy satisfies those requirements, the design review MUST document the limitation and propose an alternative for explicit approval. Rotation remains unresolved until that review is complete. This document does not select rotation or any alternative strategy.

**What the model needs to represent:**

- **Durable consumption state:** If the approved strategy uses single-use tokens, the system needs a durable way to recognize a previously consumed token, reject it, and record the required security event.
- **Concurrency safety:** The transition that permits a token to be used successfully must be coordinated so that simultaneous requests cannot both succeed with the same token. The mechanism (for example, an atomic conditional update or row locking) is not selected here.
- **Successor relationship:** Because rotation is approved, the model represents the relationship between a consumed token and its successor. A constraint allowing at most one successor is conditional on that decision. No successor chain is required if an alternative strategy is approved.
- **Session revocation:** The model supports session-level revocation. Whether detecting reuse must revoke the entire session remains unresolved.
- **Retries and lost responses:** The model does not authorize a grace period or successful reuse of a consumed token. Any retry or lost-response tolerance must be explicitly reviewed and approved, with the requirements updated if necessary.

These are logical requirements and dependencies, not a selection of a physical schema or concurrency mechanism.

### 5.5 Security Event

Records occurrences that the requirements say must be recorded or reviewed.

| Field (logical) | Meaning | Nullability | Label |
|---|---|---|---|
| Event identifier | Unique identifier. | Required | **[Proposed]** |
| Event type | Categorical type (for example: refresh-token reuse of an invalidated token, identity conflict, unresolvable identity). The full list was **[Unresolved]** here and is superseded by security-design §13.4 (Step 5d). | Required | **[Required]** need (USER-FR-011, 065, 066, 082) |
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
| Refresh Session → Refresh Token | A session manages the refresh-token state associated with that session. The logical model may represent this state using one or more token records, depending on the approved refresh-token strategy. A separate physical token table and a token chain are not mandatory. If token records are used, each belongs to exactly one session. A chain of tokens exists only if rotation is approved. | **[Proposed]**; chain length **[Unresolved]** (USER-Q-001) |
| Refresh Token → Successor | Each token has at most one successor. | **[Approved]** (rotation approved, USER-SD-DEC-004) |
| Customer / Session / Token → Security Event | An event may reference a customer, session, or token record; each reference is optional. | **[Proposed]** |

### 6.2 Keys and integrity
| Constraint | Label |
|---|---|
| Each entity has a primary key (its identifier). | **[Proposed]**; for Customer, unique and immutable identifier is **[Required]** |
| Session references Customer by a foreign key; a session cannot exist without an existing customer. | **[Proposed]** |
| Token references Session by a foreign key; a token cannot exist without an existing session. | **[Proposed]** |
| Successor references another Token record and is itself unique (at most one successor per token), so token chains cannot fork. | **[Approved]** (rotation approved, USER-SD-DEC-004) |
| Security Event may carry identifiers of a related customer, session, or token record, but retaining an event MUST NOT require retaining the corresponding session or token record indefinitely. Whether these are enforced foreign keys, plain recorded identifiers, or removed/anonymized on retention is not decided. | Independence **[Required for the proposal to be acceptable]**: see 8.4. Mechanism, referential actions, and anonymization/removal strategy **[Unresolved]** — USER-DM-DEC-007, 012 |
| The pair (identity provider, provider subject) is unique. | **[Required]** (USER-FR-010); pair form **[Approved]** |
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
| Customer: unique on (identity provider, provider subject) | Look up the customer at sign-in; enforces no duplicates (**[Required]** outcome, USER-FR-010). |
| Customer: case-insensitive unique on email | Enforces email uniqueness and detects an email collision at sign-in. Used for conflict detection only, **never** to resolve identity or merge accounts. |
| Customer: primary key on identifier | Resolve identity from a validated access token for `GET /users/me`. |
| Token: unique on token verifier | Look up a presented token at refresh and logout; enforces one record per verifier. |
| Token: on session | Find all tokens in a session when revoking or inspecting a session. |
| Token: unique on successor | Enforce at most one successor per token. Approved with rotation (USER-SD-DEC-004). |
| Session: on customer | Find a customer's sessions (needed for any "all sessions" behavior if USER-Q-003 later requires it, and for referential integrity checks). |
| Token / Session: on expiry (and revoked) | Support removal of expired or revoked records under a retention rule, once retention is decided. Deferred until retention is defined. |
| Security Event: on occurred timestamp | Review recent events. |

The email index exists only to enforce uniqueness and detect collisions; email is not an identity or lookup key for resolving a customer (USER-FR-005).

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

The consolidated retention table, with every period marked OPEN, is in security-design §13.3.

| Topic | Status |
|---|---|
| Customer account deletion | **[Unresolved]**; deferred by USER-DEC-011. Privacy obligations must still be reviewed before production (USER-PRE-001). |
| Personal-data retention (name, email) | **[Unresolved]** (USER-PRE-001). |
| Order-related retention obligations | **[Unresolved]**; Order history may need customer data to persist, so customer data MUST NOT be deleted on the assumption it is unneeded. Depends on the Order specification. |
| Authentication-session retention (expired/revoked sessions and tokens) | **[Unresolved]** — USER-DM-DEC-007. Note: retaining consumed token records is what lets reuse be recognized as reuse; removing them earlier would reject the token as unknown but could not identify it as a reuse event. |
| Security-event retention | **[Unresolved]**; 180 days is a **proposed operational default, provisional, pending privacy and legal review** (Step 5f); USER-DM-DEC-007. |
| Security-event references, referential actions, anonymization or removal | References: plain nullable identifiers, no foreign keys (Step 5f, conditional). Anonymization or removal **[Unresolved]** — USER-DM-DEC-012. |

**Event retention is independent of token retention.** Retaining a security event MUST NOT require retaining the corresponding session or token record indefinitely, and removing a session or token record MUST NOT force removal of a security event that is still within its retention need. No retention period is approved; 180 days for security events is a provisional proposal (Step 5f). **[Required for the proposal to be acceptable]** Security events continue to exclude raw tokens, token verifiers, credentials, secrets, and unnecessary personal data. **[Required]** (USER-FR-028, USER-SEC-008)

**Retention and recognition limitation:** The requirement to record presentation of an invalidated token must be implemented within the capabilities of the approved token strategy and retention policy. If the records or other information needed to recognize a previously invalidated token are removed, the system may be unable to distinguish that token from an unknown token. The security design and retention decision must explicitly address this limitation and ensure that the approved security requirements are not silently weakened. No retention period or recognition window is selected by this document.

## 9. Lifecycle Behavior

| Event | Data effect | Label |
|---|---|---|
| First successful Google sign-in (new Google account) | A Customer is created; a Session and a first Token are created. | **[Required]** outcome (USER-FR-004, 020); structure **[Proposed]** |
| Repeat sign-in (same Google account) | The same Customer is resolved; no new Customer; name/email updated per 4.3; a new Session and Token are created. | **[Required]** (USER-FR-009, 010); name/email update **[Proposed]**; new session per sign-in **[Proposed]** |
| Sign-in rejected (invalid credential, unverified email, conflict) | No Customer created, no Session or Token issued. A security event is recorded for conflicts. | **[Required]** (USER-FR-006, 007, 011) |
| Successful refresh | Strict single-use rotation (approved, USER-SD-DEC-004): the presented Token is marked consumed and a successor is issued in the same transaction, as a single atomic outcome. Any strategy must satisfy every approved refresh-token security requirement (consumed-token rejection, concurrency safety, invalidated-token events, revoked-session rejection). | **[Required]**; rotation **[Approved]** |
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

Status as of Step 5b: decisions resolved by the security design are recorded in §13 and §14. Every decision not listed there remains **Open**. Proposals are recommendations only.

| ID | Decision | Proposal | Depends on |
|---|---|---|---|
| USER-DM-DEC-001 | Is customer name required or nullable? | Nullable; confirm API implication (field may be null/absent). | API contract §6.4 |
| USER-DM-DEC-002 | Does email carry a data-integrity uniqueness constraint? What is an "identity conflict" (USER-FR-011)? | Not decided. If email is unique, a second Google account with the same email would conflict. If not, it is permitted. Both affect API-DEC-013 (`409`). | API-DEC-013; product input |
| USER-DM-DEC-003 | Are name/email updated on later sign-in, and what do Orders snapshot? | Update from validated Google data; Orders snapshot their own copy. | Order specification |
| USER-DM-DEC-004 | Refresh-token format, derivation/hashing/encryption, key management. | Not decided; must meet USER-SEC-011 and USER-AC-029. | USER-Q-001 |
| USER-DM-DEC-005 | Absolute session lifetime versus per-token lifetime (with rotation, a session could otherwise extend indefinitely); whether expiry is stored or computed from configuration. | Not decided; the model can hold either. | USER-Q-001; USER-DEC-007 |
| USER-DM-DEC-006 | Are security events stored in the database, a log, or both; full event-type list. | Database record proposed; needs confirmation. | Security design |
| USER-DM-DEC-007 | Retention of expired/revoked sessions, consumed token records, and security events (event retention independent of token retention). | No periods proposed. | USER-PRE-001; USER-Q-001 |
| USER-DM-DEC-008 | Identifier format for Customer and records. | Randomly generated, non-guessable UUID. | Physical design |
| USER-DM-DEC-009 | Referential behavior on customer removal. | None defined; customers not removed in first release. | USER-DEC-011; Order retention |
| USER-DM-DEC-010 | How `GET /api/v1/users/me` represents a missing name: `name: null`, omission of the field, or another agreed contract. | Not selected. Any choice must be reflected in the API contract through its own review. | API contract §6.4 |
| USER-DM-DEC-011 | Whether email is required (NOT NULL) in the Customer record. | Required, since sign-in needs a verified email; confirm against validation and profile requirements. | USER-FR-006; profile requirements |
| USER-DM-DEC-012 | Security-event references (enforced links vs. recorded identifiers), referential actions, and anonymization or removal strategy. | None selected; event retention must not force indefinite token-record retention. | USER-PRE-001; USER-DM-DEC-007 |
| USER-DM-DEC-013 | Whether rotation is approved, and, if not, which non-rotating strategy satisfies USER-FR-081–083. Session-level revocation on reuse and absolute session lifetime (DEC-005) remain with it. | None selected. | USER-Q-001 |

*(Superseded Step 5c to 5f: USER-DM-DEC-002 is Approved, §15.1; the conflict scenarios are in §15.4 and requirements §12.9.)* Original: USER-DM-DEC-002 stays open: email uniqueness and the exact identity-conflict scenarios are unresolved until API-DEC-013 is settled. No automatic account merging by email is introduced (USER-FR-047).

**Carried-over unresolved items (not decided here):** rotation confirmation, reuse response, session-level revocation, concurrency mechanism and retry tolerance, lost-response recovery (USER-Q-001); multiple concurrent sessions and logout-from-all-devices (USER-Q-003); rate-limit state (USER-Q-002); guest-cart mechanism (USER-Q-006); pre-launch privacy review (USER-PRE-001).

**Dependencies:** security design (USER-Q-001/002/003); Order specification (data retention, customer data needed at purchase); Cart specification (guest-to-customer merge uses the customer identifier but is not modeled here).

## 12. Acceptance Criteria for the Data Model

IDs use `USER-DM-AC-nnn`. Each is verifiable against the physical design.

**Type column:** *Required* = behavior mandated by an approved specification. *Proposed structure* = depends on a proposed logical/physical choice, verifiable only once that choice is approved. *Conditional* = applies only if the stated decision is approved. No criterion assumes an unresolved decision has been made.

| ID | Criterion | Type | Traces to |
|---|---|---|---|
| USER-DM-AC-001 | A customer has an immutable, unique identifier that is never reused. | Required | USER-FR-040 |
| USER-DM-AC-002 | At most one customer exists per Google account identifier, and a uniqueness rule enforces it. | Required | USER-FR-009, 010 |
| USER-DM-AC-003 | Email is not the sole identity key and is never used to resolve a customer's identity or to merge accounts; it is used only to enforce uniqueness and detect conflicts. *(Amended Step 5d under owner delegation. Original wording: "Email is not the sole identity key and is never used to look up or merge identities.")* | Required | USER-FR-005, 047 |
| USER-DM-AC-004 | Changing name or email does not create a new customer. | Required | USER-FR-046 |
| USER-DM-AC-005 | Only approved customer data is stored; no other Google profile data is stored. | Required (field list is Proposed structure) | USER-FR-048; USER-DEC-010 |
| USER-DM-AC-006 | Customer data can supply `id`, `name`, and `email` for `GET /users/me`, and the Google account identifier is not exposed. The representation of a missing name must follow USER-DM-DEC-010. The API's email response guarantee must be consistent with the final resolution of USER-DM-DEC-011. | Required for the fields and identifier protection; nullability behavior depends on the relevant approved decisions. | USER-FR-042; api-contracts §6.4 |
| USER-DM-AC-007 | A session cannot exist without a customer, and a token record cannot exist without a session. | Proposed structure | Section 6 |
| USER-DM-AC-008 | A customer is not prohibited from having more than one session unless an approved decision restricts this. | Required (no restriction approved) | USER-Q-003 |
| USER-DM-AC-009 | No raw refresh token is stored, and disclosure of the token store does not yield usable refresh tokens. The approved token representation and storage approach must demonstrate this property. | Required | USER-SEC-011; USER-AC-029 |
| USER-DM-AC-010 | Refresh-token expiry is derived from configuration with a 7-day default, and no lifetime is hard-coded. | Required | USER-FR-071, 072 |
| USER-DM-AC-011 | A token that is expired, whose session is revoked, or that is consumed is never accepted. | Required | USER-FR-073, 076, 081 |
| USER-DM-AC-012 | A consumed refresh token cannot be reused successfully, and no grace period exists unless separately approved. This holds under any approved refresh strategy. | Required | USER-FR-081; api-contracts §6.2 |
| USER-DM-AC-013 | Concurrent refresh requests do not produce more than one successful use of the same token, under any approved refresh strategy. The mechanism follows the approved security design. | Required | USER-FR-083; USER-AC-028 |
| USER-DM-AC-014 | Presenting an invalidated (consumed, revoked, or superseded) refresh token is rejected and recorded as a security event. | Required | USER-FR-082; USER-AC-027 |
| USER-DM-AC-015 | A revoked session cannot be used to refresh credentials. | Required | USER-FR-073, 077; USER-AC-017 |
| USER-DM-AC-016 | Logout of a recognized session records revocation; a failed revocation is never reported as success. | Required | USER-FR-073; api-contracts §6.3 |
| USER-DM-AC-017 | An identity conflict is recorded as a security event. | Required (the scenarios that constitute a conflict are Unresolved, USER-DM-DEC-002) | USER-FR-011, 066 |
| USER-DM-AC-018 | Security events contain no raw tokens, token verifiers, credentials, secrets, or unnecessary personal data. | Required | USER-FR-028; USER-SEC-008 |
| USER-DM-AC-019 | Retaining a security event does not require retaining the corresponding session or token record indefinitely. | Required for this proposal; mechanism Unresolved (USER-DM-DEC-012) | USER-DEC-011; USER-PRE-001 |
| USER-DM-AC-020 | No access-token records exist, and logout does not claim to invalidate issued access tokens. | Required | USER-FR-074, 084 |
| USER-DM-AC-021 | The model contains no guest-cart, Cart, Checkout, Order, Payment, Delivery, or Feedback structures. | Required (scope) | Section 1 |
| USER-DM-AC-022 | No data is deleted by default and no retention period exists until decided. | Required | USER-DEC-011; USER-PRE-001 |
| USER-DM-AC-023 | Before implementation, the approved refresh-token strategy must demonstrably satisfy every applicable approved security requirement, including session revocation, consumed-token rejection, invalidated-token event recording, concurrency safety, and protection against token-store disclosure. A non-rotating strategy must not be approved solely on the basis of token validation; its handling of single-use semantics and concurrent requests must be explicitly demonstrated. If no viable strategy satisfies these requirements, the design review must identify the limitation and obtain approval for an alternative before implementation. | Required | USER-FR-073, USER-FR-081–083; USER-SEC-011; USER-AC-029; USER-Q-001 |
| USER-DM-AC-024 | **Rotation (approved, USER-SD-DEC-004):** a successful refresh results in the presented token being consumed and exactly one successor being issued as one atomic outcome. | Required (approved) | USER-FR-080, 081, 083 |
| USER-DM-AC-025 | **Rotation (approved, USER-SD-DEC-004):** a token has at most one successor, so a token chain cannot fork. | Required (approved) | USER-FR-083 |
| USER-DM-AC-026 | **Session revocation on reuse (approved, USER-SD-DEC-008):** presenting a consumed token revokes its session. | Required (approved) | USER-FR-082; USER-Q-001 |
| USER-DM-AC-027 | **Absolute session lifetime (approved, USER-SD-DEC-010, 90 days):** a session cannot yield tokens beyond it. | Required (approved) | USER-DM-DEC-005 |

## 13. Security-Design Decision Updates (Step 5, 2026-10-24)

The project owner approved security-design decisions that resolve some items above. Full detail is in [security-design.md](security-design.md) §3. This section records the effect on the **logical** model. **No physical schema, DDL, or migration has been created or decided.** Existing text above is unchanged.

### 13.1 Effect on data-model decisions
| ID | Status after Step 5 |
|---|---|
| USER-DM-DEC-004 | **Resolved in part.** Refresh tokens are opaque random values stored as a one-way digest, unkeyed to begin (USER-SD-DEC-005, 006). Still open: the digest algorithm and the point at which a keyed digest is adopted, which would need a key identifier stored with the digest and its own approval. |
| USER-DM-DEC-005 | **Resolved in part.** The absolute session lifetime is **90 days** (USER-SD-DEC-010, approved). The absolute session expiry is therefore a required attribute of every session. Still open: whether it is stored at creation or computed from configuration (proposal: store at creation). |
| USER-DM-DEC-006 | **Resolved in part.** Mandatory security events are persisted and operational logs are redacted (USER-SD-DEC-019). Still open: the full event list and operational details. |
| USER-DM-DEC-007 | **Open.** Retention (USER-SD-DEC-020). |
| USER-DM-DEC-012 | **Approved (conditional), Step 5f:** plain nullable identifiers, no enforced foreign keys; anonymization or removal waits for the retention review. (Original: Open.) |
| USER-DM-DEC-013 | **Resolved in part.** Rotation is approved (USER-SD-DEC-004) and reuse revokes the whole session (USER-SD-DEC-008). The non-rotating alternative is no longer needed. Absolute session lifetime remains with DEC-005. |
| USER-DM-DEC-001, 002, 003, 008, 009, 010, 011 | **Open**, unchanged. |

### 13.2 Logical-model elements now approved
- **Successor relationship, unique-successor constraint, and its index** are no longer conditional on rotation. Each token has at most one successor.
- **Consumed state** is recorded durably so a consumed token is never reusable; consumption and successor creation happen in the **same transaction**, and an ambiguous outcome is never treated as a confirmed rollback (USER-SD-DEC-007). The exact transaction boundaries are decided at physical design.
- **Token verifier** is a stored one-way digest of the token; the raw token is never stored. This settles the earlier open question about whether a dedicated stored field exists. Its uniqueness and index remain **Proposed** for physical design.
- **Session revocation on reuse** is approved: the Refresh Session carries the revocation, and "reuse detected" is an approved revocation reason. The full set of reasons stays open.
- **Security Event** is approved as a persistent record for the mandatory events; *(Superseded: the event types are in security-design §13.4.)* Original: event types beyond the mandatory three remain open.
- **Multiple sessions** per customer are allowed; no logout-all (USER-SD-DEC-011).

### 13.3 Acceptance criteria status
USER-DM-AC-024, 025, 026 and 027 now apply (rotation, session revocation on reuse, and the approved 90-day absolute session lifetime). USER-DM-AC-023 still requires the physical design to demonstrate that the approved strategy satisfies the approved refresh-token security requirements.

### 13.4 Not changed
- No nonce state, rate-limit state, or key-identifier field was added; the related decisions (USER-SD-DEC-001, 017, and any keyed digest) remain open.
- The absolute session expiry is **required** (not nullable) on every session. Whether it is stored at creation or computed from configuration remains open.
- A single instance is the current plan (F5), but durable state, including sessions and tokens, belongs in PostgreSQL and never in process memory.
- Retention periods are not chosen.

### 13.5 Absolute session lifetime and terminology (USER-SD-DEC-010)
Logical rules the physical design must preserve:
- Each session has an **absolute session expiry** = creation time + 90 days, fixed at creation. Refreshing never changes it.
- Each refresh token has its own **refresh-token expiry** (7 days by default; USER-FR-071, 072). A successor's expiry is the earlier of (now + refresh lifetime) and the session's absolute expiry, so a token never outlives its session.
- A refresh at or after the absolute session expiry is rejected with the generic refresh error and the session ends.
- Access tokens (15 minutes by default) are stateless and have no record; they can outlive the session by up to their own remaining lifetime.
- The 90-day value is read from configuration like the other lifetimes (proposal), and a value above 90 days requires separate approval.

| Term | Meaning |
|---|---|
| Session (refresh session) | The record created at one sign-in; the unit of revocation; the family of that sign-in's refresh tokens |
| Absolute session expiry | Session creation + 90 days; fixed; never extended by refreshing |
| Refresh-token expiry | Per-token expiry, 7 days by default, never later than the absolute session expiry |
| Access-token expiry | 15 minutes by default; stateless; independent of the session |
| Inactivity window | The refresh-token lifetime; there is no separate inactivity timer |

The revised row for the Refresh Session's *Absolute session expiry* in §5.1 (previously "Nullable, Unresolved") is superseded by this section. Physical types and constraints are not decided here.

### 13.6 Sign-in Nonce entity and retention requirements (approved in principle in Step 5b; see §14.1)
**Sign-in Nonce — approved in principle (USER-SD-DEC-001; USER-DM-DEC-014; see §14.1).** If the server-issued, single-use nonce is approved, the logical model needs a short-lived record:

| Field (logical) | Meaning | Label |
|---|---|---|
| Nonce value (or a one-way digest of it) | The random server-issued value the Google credential must carry | Proposed |
| Created timestamp, expiry timestamp | Short lifetime, set by configuration; no value proposed | Proposed |
| Consumed marker | Set exactly once, atomically, on first presentation, whether sign-in then succeeds or fails | Proposed |

The nonce is unique, is not tied to a customer (the caller is unauthenticated), and is removed under a short retention rule. An optional binding to the requesting browser is open. **No physical schema is decided.** If the nonce is not adopted, nothing is added.

**Retention requirements for review (USER-SD-DEC-020, open; detail in security-design §13.3).** No period is approved (180 days for security events is a provisional proposal, Step 5f).
- *Sessions:* active sessions are needed to authorize refresh; revoked and expired session records are needed to reject their tokens and attribute events.
- *Consumed refresh-token records:* needed at least until the token's own expiry to recognize a presented token as previously consumed. If a consumed record is deleted earlier, the token looks unknown, is still rejected, but cannot be recognized as reuse, so no reuse event is recorded and the session is not revoked on that basis.
- *Security events:* kept independently of token records (USER-DM-AC-019).

**Cookie, CORS and CSRF decisions** (USER-SD-DEC-013 to 016) have no effect on the logical data model. The absolute session expiry, the stored one-way token digest, and the successor relationship are unchanged.

## 14. Security-Design Decision Updates (Step 5b, 2026-10-24)
Logical-model effect of decisions USER-SD-DEC-001, 025, 026, 027, 028, 029. **No physical schema, DDL or migration is created or decided.** Existing text above is unchanged except the three rows and headings noted in §5.1, §5.2, §11 and §13.6.

### 14.1 Sign-in Nonce entity (approved in principle; USER-DM-DEC-014)
**Why it is needed:** "single use" requires the server to remember what it issued. In-process memory is lost on restart or redeploy and is not shared across instances.

| Field (logical) | Meaning | Nullability |
|---|---|---|
| Nonce digest | One-way digest of the issued nonce; the nonce is found by its digest | Required, unique |
| Binder hash | The hash of the page-held binder supplied when the nonce was requested | Required |
| Created timestamp, expiry timestamp | Expiry is creation plus five minutes (from configuration) | Required |
| Consumed marker | Set once, atomically, on first presentation of a validly signed Google token carrying the nonce | Nullable (null until consumed) |

Relationships: none to a customer or session. Lifecycle: expired and consumed rows are removed under a short retention rule, **opportunistically** at issuance (no scheduler is decided; part of decision 020). Constraints: the digest is unique; consumption is a single conditional update that succeeds for exactly one request.

### 14.2 Session CSRF-token digest (USER-DM-DEC-015, approved)
The Refresh Session carries a one-way digest of its session-scoped CSRF token (§5.1). It is created at sign-in, constant for the session, never logged, and discarded with the session. **Why:** the token must be checked against the session on refresh and logout without storing the raw value.

### 14.3 Security events
**Event types follow security-design §13.4.** Mandatory and durable: refresh-token reuse and session revocation, identity conflict, unresolvable identity, **nonce replay and invalid nonce** (`AUTH_NONCE_REPLAY`, `AUTH_NONCE_INVALID`), and the audit events for successful sign-in, refresh-token rotation, session revocation and logout, each atomic with its state change. Failed sign-in and rate-limit rejection are **operational logs and metrics only**, so they cannot cause unbounded database writes. Events never contain the credential, the nonce, the binder, tokens or digests. The nonce-replay event was previously labelled "proposed"; that label is superseded (USER-DM-DEC-006).

### 14.4 Database access controls (logical requirement)
- The authentication tables must not be reachable through Supabase's HTTP APIs, and nothing is granted on them to the roles those APIs use.
- **Row-level security is not a replacement for database permissions.** The Express application login holds only the privileges it needs, with no schema-change, superuser or row-security-bypass rights, and the database owner role is separate from the application login.
- Specific schema, role and privilege design belongs to the physical design; no role names or credentials are recorded.

### 14.5 Profile X and orphaned sessions
Where third-party cookies are blocked, each re-sign-in creates a session whose refresh cookie is never stored. Such sessions are unused until they expire. Retention and cleanup of expired sessions (decision 020) and sign-in rate limits account for this.

### 14.6 New data-model decisions and acceptance criteria
| ID | Decision | Status |
|---|---|---|
| USER-DM-DEC-014 | Sign-in Nonce entity (§14.1) | Approved in principle |
| USER-DM-DEC-015 | CSRF-token digest on the session (§14.2) | Approved |

| ID | Criterion | Type |
|---|---|---|
| USER-DM-AC-028 | A nonce can be consumed by exactly one request, and only while unexpired. | Required (approved in principle) |
| USER-DM-AC-029 | The nonce, binder hash and CSRF-token digest are never logged or placed in security events, and no raw nonce or CSRF token is stored. | Required |
| USER-DM-AC-030 | The session stores a CSRF-token digest, not the raw token. | Required (approved) |
| USER-DM-AC-031 | The application database role cannot alter schema and does not bypass row-level security; the owner role is separate. | Required (approved) |
| USER-DM-AC-032 | Expired nonces are removed without requiring a scheduler, unless one is later approved. | Proposed |

### 14.7 Still open for the physical design
The connection method and role setup (USER-SD-DEC-026), the nonce cleanup details, whether absolute session expiry is stored or computed, the identifier format, the digest algorithm and its keying, constraint and index definitions, and retention.

## 15. Identity, Profile and Event Decisions (Step 5c, 2026-10-25)
Owner decisions recorded here use the repository IDs. **No physical schema, DDL or migration is created or decided.** Where an earlier row or acceptance criterion above conflicts, this section governs for the decisions listed, and the acceptance criteria wording is **not** altered silently (§15.9).

### 15.1 Decisions
| ID | Decision | Status |
|---|---|---|
| USER-DM-DEC-008 | Internal identifiers are **UUID v4 generated by PostgreSQL** | **Approved** |
| Identity pair | The pair (identity provider, provider subject) is unique; the Google subject is the stable external identity; email is never the primary identity key and accounts are never merged by email | **Approved** |
| USER-DM-DEC-001, 010 | Customer name is nullable; stored as NULL when Google provides no usable name; returned as `"name": null`; no placeholder | **Approved** |
| USER-DM-DEC-011 | Email is required and validated (Google ID token and verified-email claim) before storage | **Approved** |
| USER-DM-DEC-002 | Email is unique **case-insensitively**, with no automatic account merging | **Approved** (implications in §15.4; the existing-subject case was decided in Step 5d, §15.4) |
| USER-DM-DEC-003 | The current profile (name, email) is updated at each successful sign-in from validated Google claims; historical orders keep a snapshot | **Approved** |
| USER-DM-DEC-005 (remainder) | The absolute session expiry is stored at session creation | **Approved** |
| USER-DM-DEC-012 | Security events use stable identifiers without foreign keys | **Approved (conditional)**; Step 5f: owner prefers plain nullable identifiers; remains provisional until the retention review (USER-SD-DEC-020, open) |
| USER-DM-DEC-006 | Mandatory events are persisted and critical state changes are atomic with their required events | Principle **Approved**; the enumeration is **Proposed** (security-design §13.4) |

### 15.2 Identity
A customer is found by (identity provider, provider subject). Email is never used to find or merge a customer. A first sign-in with an unknown pair creates a customer, unless the email is already held by another customer (§15.4).

### 15.3 Name
Nullable. If Google supplies no name, or one that is empty after trimming, the stored value is NULL. The API returns the key with the value `null`. Nothing is derived from the email or invented.

### 15.4 Email: implications of case-insensitive uniqueness (documented before any use)
- **Validation and storage:** only after the Google ID token and its verified-email claim are validated. Stored exactly as Google supplies it. Comparison is case-insensitive; no dot or plus-tag stripping.
- **Existing accounts:** none exist yet (nothing is implemented). If duplicates were ever present before the constraint is applied, they must be resolved by the owner manually; they are never merged automatically.
- **Conflict at sign-in (unknown subject, email held by another customer):** sign-in is refused with `409 AUTH_IDENTITY_CONFLICT`; no customer is created, no session is issued, nothing is merged, and the mandatory identity-conflict event is recorded (security-design §13.4). The response does not reveal the other customer.
- **Concurrent first sign-ins:** the same subject resolves to one customer (the unique pair absorbs the race); two different subjects with the same email result in one success and one conflict.
- **Future Google identity linking:** a customer holding two Google accounts would need a deliberate, authenticated linking flow that proves both identities in one session. It is never automatic by matching email. With the provider and subject stored on the customer row, supporting several identities per customer would need a separate identity record later; that is a future migration, not a first-release feature (USER-DEC-009).
- **Consequence (denial, not takeover):** a person who legitimately receives an address that another customer already holds (for example a reassigned address) cannot sign in and has no recovery path in the first release (account recovery is deferred, USER-DEC-011). A support or recovery procedure is **not defined** and is an open question.
- **Decided (Step 5d, delegated): the locked-out address owner.** The first release has **no self-service recovery**. The `409` message tells the person to contact support. A **manual, owner-handled support procedure** (out-of-band verification, then manual resolution by the project owner, never an automatic merge) **must be defined and documented before production**. Contact details and the procedure itself are not invented here and are a production gate (security-design `USER-SD-AC-045`).
- **Safety of email-based operations:** operations use the customer identifier, never an email lookup. A future feature that searches by email must treat a match as a hint and never as proof of identity.
- **Decided (Step 5d, delegated):** when a customer with an existing subject signs in and Google now reports an email that another customer already holds, **sign-in succeeds** (identity is the subject), the **stored email is kept and not updated**, and a conflict event is recorded. Reason: the customer must not be locked out of their own account because of another customer's address, and no merge ever happens.

### 15.5 Profile updates and order history
| Data | Source | Change |
|---|---|---|
| **Current profile** (name, email) | The customer record | Updated at each successful sign-in from validated, verified-email Google claims, subject to §15.4 |
| **Historical order data** | A snapshot taken by the Order domain at purchase | Never rewritten by later profile changes |
Because name may be NULL, Checkout and Order must collect and snapshot fulfilment name and contact details themselves. This is a cross-domain dependency.

### 15.6 Identifiers
UUID v4 generated by PostgreSQL (database-supported random UUID generation; availability on the project's PostgreSQL version is confirmed at physical design) for customers, sessions, refresh-token records and security events. A nonce is identified by its digest.

### 15.7 Session expiry
The absolute session expiry (creation + 90 days) is stored when the session is created and is never changed by refresh (USER-SD-DEC-010).

### 15.8 Security events
- **Stable identifiers without foreign keys**, conditional on the retention design. Trade-offs: no referential integrity (identifiers must come from server state), but events survive removal of sessions, tokens and, later, customers, and retention stays independent.
- **Atomicity:** a mandatory event is written in the same transaction as the state change it describes, so a failed event write fails the operation.
- Ordinary IP logging is disabled by default (security-design §13).

### 15.9 Tension with approved acceptance criteria (reported, not changed)
- **USER-DM-AC-003** originally said email "is never used to look up or merge identities", which conflicted with conflict detection by email. **Amended in Step 5d** (see the criterion; the original wording is preserved in it).
- **USER-DM-AC-002** refers to the "Google account identifier"; the pair form (provider, subject) is its approved meaning.

### 15.10 Acceptance criteria derived from the approved decisions
| ID | Criterion |
|---|---|
| USER-DM-AC-033 | The pair (identity provider, provider subject) is unique, and two customers can never share it. |
| USER-DM-AC-034 | Two customers can never hold the same email ignoring case, and no account is merged automatically. |
| USER-DM-AC-035 | A customer without a usable Google name has a NULL name, and the API returns `"name": null`. |
| USER-DM-AC-036 | Identifiers are random UUIDs generated by PostgreSQL. |
| USER-DM-AC-037 | A session's absolute expiry is stored at creation and never changed by refresh. |
| USER-DM-AC-038 | A mandatory security event and the state change it describes commit together or not at all. |
| USER-DM-AC-039 | A sign-in by an existing subject whose Google email collides with another customer succeeds, keeps the stored email, and records a conflict event. |

### 15.11 Still open
The retention design (which also decides the no-foreign-key choice); the content of the support procedure for a locked-out address owner (a production gate); the exact event list for failed sign-in and rate-limit rejection (decided as operational logs; see security-design §13.4).

## 16. Traceability and Consistency Check (Step 5e, 2026-10-25)
### 16.1 Traceability
| Data element | Requirements | Acceptance criteria | Decisions |
|---|---|---|---|
| Sign-in Nonce (§14.1) | USER-FR-087, USER-SEC-014, USER-SEC-009 | USER-AC-031, 032, 033, 039; USER-DM-AC-028, 029, 032 | USER-SD-DEC-001, 027; USER-DM-DEC-014 |
| Session CSRF-token digest (§14.2) | USER-SEC-015 | USER-AC-034, 036; USER-DM-AC-030 | USER-SD-DEC-028; USER-DM-DEC-015 |
| Table exposure and roles (§14.4) | USER-SEC-016 | USER-AC-037, 038; USER-DM-AC-031 | USER-SD-DEC-025, 026 |
| Identity pair, email, name (§15) | USER-FR-005, 009 to 011, 046, 047, 066 | USER-AC-013, 023; USER-DM-AC-033 to 035, 039 | USER-DM-DEC-001, 002, 003, 010, 011 |
| Events and atomicity (§14.3, §15.8) | USER-FR-011, 065, 066, 082 | USER-DM-AC-018, 019, 038 | USER-DM-DEC-006, 012; USER-SD-DEC-019, 024 |

### 16.2 Consistency check before any physical detail is finalized
| Topic | Result | Open |
|---|---|---|
| Nonce lifecycle | **Consistent** across security-design §4.4, api-contracts §11.1–11.2, this model §14.1, frontend-contract §2, USER-FR-087 and USER-SEC-014: five minutes, digest, one atomic conditional consumption, replay rejected and recorded | The digest algorithm and the binder-hash encoding are not fixed; opportunistic cleanup of expired rows is described but not designed |
| Event retention and foreign keys | **Consistent but not final:** events use plain identifiers (USER-DM-DEC-012, conditional on the retention decision); nothing in the model requires a session or token row to outlive its events | Retention periods (USER-SD-DEC-020) are OPEN and may change the foreign-key decision |
| Session relationships | **Consistent:** one customer, many sessions; one session, many token records; absolute expiry stored at creation; no customer removal in the first release; restrictive referential actions | Revocation-reason and event-type enumerations are not final |
| Email constraint | The logical constraint (case-insensitive unique, never an identity key) is consistent with the API contract (§12.3) and requirements | **UNVERIFIED** against a physical constraint: no physical design exists |

## 17. Step 5f Owner Decisions (2026-10-25)
Specification only; no schema, DDL or migration is created.
- **Event references:** an event holds customer, session and token-record references as plain nullable identifiers, not enforced foreign keys, so event lifetime is independent of those records (USER-DM-DEC-012; `USER-DM-AC-019`).
- **Session and token cleanup** is separate from event retention. A consumed token record is kept at least until the token's own expiry; no other period or timing is approved (security-design §13.3).
- **Event retention:** 180 days is a proposed operational default pending privacy and legal review; provisional.
- **Email:** `USER-FR-088` and `USER-AC-042` (requirements §12.9) record case-insensitive uniqueness. The mechanism (for example how comparison is made and how a unique violation inside a transaction is handled for the existing-subject collision) is a **physical-design question and is not decided here**. The conflict event records no email address. The reassigned-address limitation and the absence of self-service recovery stand (§15.4).
| Data element | Requirements | Acceptance criteria |
|---|---|---|
| Customer email uniqueness (§5.1, §15.4) | USER-FR-088 | USER-AC-042 |
| Security event (atomic write, no client IP) | USER-SEC-017, 018 | USER-AC-040, 041 |

## 18. Step 5g: Precedence, Collision Handling and Traceability (2026-10-25)
**Precedence.** This document is built by appending. Where an earlier section differs from a later one, the **later section governs**: §13 over §1 to §12, §14 over §13, §15 over §11 to §14, §17 and this section over all. Stale statements in earlier sections (for example the open items in §5.4, §6.1, §6.2 and §11) are historical.
**Existing-subject email collision (USER-FR-088; USER-AC-042).** Sign-in succeeds, the stored email is unchanged, and a conflict event is recorded with the existing `AUTH_IDENTITY_CONFLICT` type and a distinct reason code, **never the email address** (security-design §13.4). If the event cannot be stored, the sign-in still succeeds and the failure is detected independently (requirements §12.10, Exception 2). How the unique violation is avoided or absorbed inside the sign-in transaction is a **physical-design decision, not made here**.
| Data element | Requirements | Acceptance criteria |
|---|---|---|
| Email uniqueness, collision handling (§4.2, §15.4) | USER-FR-088, 005, 011, 047 | USER-AC-042; USER-DM-AC-034, 039 |
| Event atomicity and exceptions (§15.8) | USER-SEC-017 | USER-AC-040; USER-DM-AC-038 |
| No client IP in logs (§15.8) | USER-SEC-018 | USER-AC-041 |
