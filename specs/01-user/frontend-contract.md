# 01 — User & Authentication: Frontend Authentication Contract

**Status:** Draft — pending project owner review

This document states the **browser behavior** the React frontend must follow to work with the authentication API. It is a behavior contract, not a React design, and it contains no application code, configuration or credentials.

**Labels:** **[Approved]** · **[Approved (conditional)]** · **[Recommended]** (not approved) · **[Open]**. Decision IDs are in [security-design.md](security-design.md) §3.

**Sources:** [requirements.md](requirements.md), [api-contracts.md](api-contracts.md) §11, [data-model.md](data-model.md) §14, [security-design.md](security-design.md) §4.4, §6.4, §10.

---

## 1. Scope and assumptions
- **In scope:** Google sign-in, the nonce and binder, the access token, the refresh-token cookie, the CSRF token, refresh concurrency, logout, CORS and credentials, handling expired access tokens and failed refreshes.
- **Out of scope:** component design, routing, styling, Cart, Checkout and other domains, native clients.
- **Clients:** browser only (confirmed). Frontend origin `https://varsha-homemade.onrender.com`; API origin `https://varsha-customer-service.onrender.com`.
- **Profiles:** the two cookie profiles (X: current cross-site Render origins; Y: custom domains or same-origin proxy) are **alternatives**, and **no production profile is selected**. This contract applies to both unless a section says otherwise.

## 2. Initiating Google sign-in  [Approved (conditional): decisions 001, 027]
1. Before each sign-in attempt the page generates a **binder**: a high-entropy random secret kept **only in memory**. It computes the binder's hash.
2. The page calls `POST /api/v1/auth/google/nonce` with the binder hash and receives a nonce and its lifetime (five minutes).
3. The page starts Google sign-in passing that nonce. The nonce is single-use.
4. If the nonce has expired (the customer waited too long), the page requests a **new nonce** and starts again. This is a customer-driven restart, not an automatic retry loop.
5. The page never stores the binder, the nonce or the Google credential beyond the attempt, and never logs them.

## 3. Submitting the Google credential  [Approved (conditional)]
- The page posts the Google credential and the raw binder to `POST /api/v1/auth/google` over HTTPS.
- Any failure is shown as a generic sign-in failure. The page does not interpret the reason, and does not retry automatically.
- `409 AUTH_IDENTITY_CONFLICT`: shown as a non-retryable message that this Google account cannot be used for sign-in and that the customer should contact support, with no automatic retry. The support path is a **manual, owner-handled procedure that must be defined before production** (Step 5d).
- On success the page receives the access token, its lifetime, the **CSRF token** (once), and the customer's profile. The refresh token arrives as a cookie the page cannot read.

## 4. Access token  [Approved: memory only]
- Held **only in page memory** (for example a module variable). Never in `localStorage`, `sessionStorage`, cookies readable by script, URLs or logs.
- Sent as `Authorization: Bearer <token>` on protected requests, and **without** cookie credentials.
- Lost on page reload. It is restored by the page-load refresh (§6).

## 5. CSRF token  [Approved: decision 028]
- Received once at sign-in. Stored in **browser storage shared by the tabs of the frontend origin**, so reloads, new tabs and restored tabs still have it. It is not an authentication credential by itself.
- Sent in the `X-CSRF-Protection` header on **every refresh and logout** request.
- Cleared on logout and whenever the session ends. If it is missing while the cookie exists, a refresh is rejected (`403 REQUEST_REJECTED`) and the customer signs in again.

## 6. Refresh and page load  [Approved: decisions 004, 009, 023]
- On page load the page attempts **one** refresh to restore the session. If it fails, it shows the sign-in prompt.
- A refresh request sends credentials (the cookie) **only when the API origin differs from the page origin**, plus the `X-CSRF-Protection` header and a JSON content type. `Origin` is set by the browser.
- The refresh response gives a new access token. The replacement refresh token arrives as a cookie.
- **No automatic retry of a failed refresh request.** A failed refresh ends the session in that client and shows the sign-in prompt.

## 7. One refresh at a time per tab  [Approved: decision 023]
Within one tab, at most one refresh is in flight. Callers that need a token while one is in flight wait for and share its result.

## 8. Coordination across tabs  [Recommended; Open]
- **Why it matters.** The refresh cookie is shared by every tab, and each refresh replaces it. Strict rotation treats a second request carrying an **already-used** token as reuse, which revokes the **whole session** (decision 008). Two tabs refreshing at the same moment, for example when a browser restores several tabs together, can therefore sign the customer out.
- **Recommended behavior.** Tabs of the frontend origin agree that only one refreshes at a time (for example by a browser-provided cross-tab lock). A waiting tab then refreshes afterwards using the cookie the previous refresh set, or reuses a fresh access token if one is already available.
- **What this does and does not achieve.** It **reduces** reuse incidents caused by overlapping refreshes in the same browser. It **does not guarantee** that every race or lost response is eliminated. It cannot prevent: a refresh whose response was lost (the server rotated the token but the browser never received the new cookie); an intermediary resending a request; an ambiguous database outcome on the server; a crash mid-request; a browser without the lock feature; or a blocked cookie.
- **Decision needed:** require this coordination, or accept the risk.

## 9. Expired access tokens and failed refresh  [Approved: no automatic retry; flow Recommended]
| Situation | Behavior |
|---|---|
| A protected request returns `401` (access token expired or invalid) | Perform **one** refresh (single-flight). If it succeeds, **replay the original request once**. If the replay fails, or the refresh fails, end the session. |
| Refresh returns `401 AUTH_REFRESH_TOKEN_INVALID` | Show the sign-in prompt. **Do not retry.** |
| Refresh or logout returns `403 REQUEST_REJECTED` | Treat as non-retryable. If the CSRF token or `Origin` is wrong the customer signs in again. Never loop. |
| `429 RATE_LIMITED` | Honor `Retry-After`; no tight loop |
| `503 SERVICE_UNAVAILABLE` | Show a generic message; no automatic retry of refresh |
- Each original request is replayed **at most once**. There is no retry loop anywhere.
- The API distinguishes an expired token with `AUTH_ACCESS_TOKEN_EXPIRED` (API-DEC-005, approved under delegation). The page still treats any `401` on a protected request the same way: one refresh, one replay.

## 10. Logout  [Approved: decisions 011, 012]
- The page calls `POST /api/v1/auth/logout` with credentials, `X-CSRF-Protection`, and a JSON content type.
- It **clears the in-memory access token and the CSRF token whether or not the server call succeeds**, and tells other tabs to clear theirs.
- It does not retry logout. Logout ends **only the current session**; there is no logout-all.
- Already-issued access tokens may remain valid until expiry (up to 15 minutes); the page simply stops using them.

## 11. CORS, credentials and CSRF  [Approved: decisions 015, 016]
- Calls to the API from the frontend origin are cross-origin under Profile X and Y1, so the API's CORS allow-list must contain the frontend origin (`https://varsha-homemade.onrender.com` today; **re-approved if custom domains are configured**).
- Credentials are sent only on refresh and logout (cookie-authenticated). Bearer calls omit them.
- The custom header makes the browser send a preflight request, which the allow-list answers.
- The page never reads the refresh cookie and never puts tokens in URLs.
- **Script-injection hygiene [Recommended]:** a strict content-security policy and no raw HTML injection, because the Origin check and custom header do not stop script injection.

## 12. Profile X: third-party-cookie fallback  [Approved: decision 029]
Under Profile X the refresh cookie is a third-party cookie. A browser that blocks or partitions it may refuse to store or send it, so silent refresh fails. **A CSRF token does not solve that.**
- The page treats a failed refresh as the end of the session and shows the sign-in prompt. This can happen every 15 minutes (the access-token lifetime) and on every reload in such browsers.
- The page may remember that a refresh failed right after a sign-in and then **skip the silent refresh on later loads**, to avoid pointless requests and to show a clear message. **[Open]** for review.
- Each such sign-in creates a session whose cookie is never stored. This is accounted for in rate limits and retention (security-design §10.6, §11.1).
- Profile Y removes the cause.

## 13. Prohibited behavior
- No access, refresh or CSRF token in URLs, logs or analytics.
- No secret of any kind in React code or browser-delivered configuration: no Google client secret, no signing secret, no Supabase key.
- No automatic retry of a failed refresh; no unbounded retries of any authentication request.
- No use of the Supabase client library or Data API from the browser.

## 14. Acceptance criteria
| ID | Criterion | Traces to |
|---|---|---|
| FE-AC-001 | The access token exists only in memory; it is never in persistent storage, URLs or logs. | §4; USER-SEC-008 |
| FE-AC-002 | A nonce is fetched before each sign-in; an expired nonce is replaced by a new one only when the customer restarts the attempt. | §2 |
| FE-AC-003 | Refresh and logout always send `X-CSRF-Protection`; credentials are sent only on those calls. | §5, §11 |
| FE-AC-004 | At most one refresh is in flight per tab; a failed refresh is never retried automatically. | §7, §6 |
| FE-AC-005 | An expired access token causes one refresh and at most one replay of the original request, never a loop. | §9 |
| FE-AC-006 | Logout clears local state whether or not the server call succeeds, and does not retry. | §10 |
| FE-AC-007 | Under Profile X, a refresh that fails because the cookie is absent leads to the sign-in prompt with no retry loop. | §12 |
| FE-AC-008 | No secret appears in the frontend code or its configuration. | §13 |
| FE-AC-009 | If cross-tab coordination is approved, two tabs never send overlapping refresh requests in the same browser. | §8 (conditional) |

## 14.1 Traceability to requirements (added Step 5f)
This contract implements `USER-SEC-014` to `016` (nonce, binder, CSRF), `USER-FR-087` (nonce endpoint) and acceptance criteria `USER-AC-031` to `039` (requirements §12.7). It adds no requirement.

## 15. Open items and questions
1. Require cross-tab coordination (§8), or accept the risk?
2. May the page skip silent refresh after a known cookie failure (§12)?
3. Final review of the sign-in sequence (§2, §3) once the nonce endpoint is approved.
4. Which profile is used in production (not selected).
5. Whether a development origin is needed for the CORS allow-list (none is specified).
