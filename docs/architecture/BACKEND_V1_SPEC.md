<!-- Purpose: Spec for backend v1. A spec, not code: it exists so each slice can be reviewed against written acceptance criteria before it is built. -->
# Backend v1 — Specification

**Status:** DRAFT for review. No code is written against this until the open decisions in §10 that gate a slice are answered.
**Written:** 2026-10-10. **Grounded in:** a file-level inventory of the existing front-end contracts (cited below as `file:line`) and `SALVAGE_MANIFEST.md`.

## 1. Context and decisions already made

This repository is a static SPA (Caddy on Railway) with **no backend**. Shopify is the system of record for catalog and checkout. Every other "service" is a stub, a mock, or browser-local storage. On 2026-10-10 the maintainer decided:

| # | Decision | Consequence |
|---|---|---|
| D1 | Build a real backend service (this spec). | Reverses the roadmap's v1.0 anti-scope line "do not build a backend". `docs/MASTER_EXECUTION_PROGRAM.md` must be amended when slice 0 lands. |
| D2 | v1 scope: **quotes + admin queue, real admin auth, customer accounts / logged-in state, Fleet Workspace persistence.** | Four capabilities, sliced in §6. |
| D3 | Lives in this repo under `server/`, deployed as a **second Railway service** alongside the unchanged static front-end service. | Shared contracts, one CI, one PR flow. |
| D4 | Customers sign in with **Shopify Customer Accounts** (hosted login). | We never store customer passwords; the backend is a Shopify confidential client (§5.2). |
| D5 | **Launch runs in parallel.** The storefront launch list in `docs/MASTER_EXECUTION_PROGRAM.md` does not depend on this backend. | Every slice must keep guest/signed-out behavior working exactly as today. |

## 2. Principles

1. **The front end already wrote the contract.** Quotes, admin auth, customer auth and cart each sit behind a port. The backend's job is real implementations of those ports, not new product design. Where a port is wrong for a real system, this spec says so explicitly (§4, §5) rather than bending the server to fit a stub.
2. **The server is the authority.** Client-side checks (`hasPermission`, forward-only status transitions, per-project limits, email regex) are UX. Every one is re-enforced server-side. The `VITE_ADMIN_ENABLED` flag is a build-time convenience, not a security boundary.
3. **Honest outcomes, never fabricated success** (the standing rule in `quoteDeliveryAdapter.ts:8-10`). An API failure is shown as a failure, with the existing phone/mailto fallback. A quote is never reported delivered unless it was durably stored.
4. **Guest keeps working.** Signed-out users keep today's localStorage behavior. Signing in adds persistence; it never gates browsing, cart, or checkout.
5. **Boring technology.** Fewest moving parts that satisfy the contracts.
6. **One slice = one PR = one review gate.** No slice starts until the previous one is reviewed and merged.

## 3. Architecture

### 3.1 Topology

```
Browser ── same origin ──> Caddy (static SPA service)
                              ├─ /*      → SPA (existing)
                              └─ /api/*  → reverse_proxy → backend service (Railway private network)
                                                              └─ Postgres (Railway)
Browser ── Shopify-hosted ──> checkout / customer-account login   (unchanged / S3)
```

**Decision: proxy `/api/*` through Caddy so the browser sees one origin.** Rationale: (a) admin and customer sessions can use first-party `HttpOnly; Secure; SameSite=Lax` cookies instead of tokens in `sessionStorage` (today's admin token handling, `adminAuthStorage.ts`); (b) no CORS surface; (c) no dependence on third-party-cookie behavior across two `*.up.railway.app` hostnames. The service's `RAILWAY_PRIVATE_DOMAIN` variable already exists for private-network addressing. `Caddyfile` currently has no `/api` route (`Caddyfile`, static + SPA fallback only), so this is a small, explicit change in slice 0.

### 3.2 Stack (recommendation — confirm in §10)

- Node 22 LTS, TypeScript, **Fastify**, **Zod** (already the repo's contract language: 39 `src/schemas/*.schema.ts`).
- **Postgres** on Railway; plain SQL migrations (`node-pg-migrate` or equivalent) and a thin typed query layer. No heavyweight ORM.
- Tests with the repo's existing `node --test` runner; DB-backed tests against a CI Postgres service container.

### 3.3 Repository layout

```
server/                 own package.json, tsconfig.json, Dockerfile, migrations/
shared/contracts/       NEW Zod schemas + inferred types for every API request/response
src/                    unchanged location; imports shared/contracts via alias
```

Constraint discovered in the inventory: `src/types` and `src/schemas` are cleanly importable from Node (no React / `import.meta` / browser globals) **but** use the `@/` alias with extensionless relative imports and `moduleResolution: Bundler`. Plain Node ESM will not resolve them. **Decision: bundle the server with esbuild (or run via `tsx` with tsconfig paths)** rather than moving files. Some contract types the server needs do **not** live in `src/types` and must be extracted into `shared/contracts/`:

- delivery `QuotePayload` + sub-types — only in `src/adapters/quoteDelivery/quoteDeliveryAdapter.ts` (which also touches `window`/`fetch`);
- `QuoteRecord` — only in `src/adapters/base44/adminQuoteAdapter.ts`;
- all fleet types' **schemas** — types exist (`src/types/fleetBuilds.ts` etc.) but there are **no zod schemas** for fleet builds/projects/templates, department standards, upfit state, or saved/compare lists.
- Name collision to resolve: `src/schemas/quote.schema.ts` and `src/types/quote.ts` already export a *different* `QuotePayload` (`{quote, source, submittedAt?, metadata?}`, `quote.schema.ts:444`). New contracts use distinct names (`QuoteSubmission`, `QuoteRecordDto`).

## 4. Capability: Quotes and the admin queue

### 4.1 Today
- Delivery: `quoteDeliveryAdapter.ts` POSTs `{...payload, referenceId, recipientEmail}` to `VITE_QUOTE_DELIVERY_ENDPOINT` if set, else opens `mailto:`. **It checks only `response.ok` and never reads the body** (`:173-182`), and derives `referenceId` client-side as `QR-` + last 6 chars of `submissionId` (`:85-87`). `submissionId` is `sub-${Date.now()}-${random6}` (`quoteRequestService.js:21-23`).
- Admin queue: `src/adapters/base44/adminQuoteAdapter.ts` — `fetchAllQuotes()` always returns `[]`; `updateQuoteStatus()` echoes its input. The queue is empty by construction (P0 #303/#307).

### 4.2 Mismatches the server must resolve
| Issue | Resolution |
|---|---|
| Admin UI reads **flat** `contactName`, `agency`, `email`, `phone`, `vehicleCount`, `notes`, `submittedAt`; payload carries **nested** `contact.{...}` and `timestamp`; `id`/`status` don't exist in the payload. | Server stores the full submission and serves a flat `QuoteRecordDto` matching what `AdminQuotesPage.jsx` reads, plus a `details` object (`lines`, `quantity`, `vehicleSummary`, `matchingSkus`, `skuStatus`, `productId`, `configuratorId`). Map: `contact.name→contactName`, `timestamp→submittedAt`, generated `id`, initial `status:'new'`. |
| Cart quotes populate `lines[]`, which `QuoteDetailModal` does not render — they would show an empty modal. | Slice adds a `lines` section to `QuoteDetailModal`. |
| Two reference IDs: submitter sees `QR-`+last6(`submissionId`); admin UI shows `QR-`+last6(`record.id`) (`adminQuoteService.js:32-34`). | Server returns `referenceId`; client adapter uses it when present; admin UI reads `record.referenceId`. Same value in both places. |
| Client discards the response and never learns a server id; idempotency is by client `submissionId` only. | Server de-duplicates on `submissionId` (unique); a replay returns the original `referenceId` with `200`. |
| Status machine `new→reviewed→quoted→closed` is forward-only, one step, enforced only client-side (`adminQuoteService.js:9,26-29,40-44`); client can pass any string. | Server validates transitions; rejects skips/backwards with `409`. |
| `adminAccessService.checkAdminAccess()` is hard-coded `{authorized:true}` and `AdminQuotesPage` relies on it in addition to the guard. | Removed; replaced by session-based checks. Data endpoints enforce `admin.quotes.view` server-side. |

### 4.3 API
- `POST /api/quotes` — **public**, body `QuoteSubmission` (= today's `QuotePayload`). Validates with Zod; re-validates contact rules (`name`, `agency`, `email` required; email regex per `quoteRequestService.js:31-45`). Rate-limited per IP. Returns `{ id, referenceId }`.
- `GET /api/admin/quotes` — `admin.quotes.view`. Newest first. Returns `QuoteRecordDto[]` (client filters/counts per status today, so no pagination in v1; revisit at ~1k rows).
- `PATCH /api/admin/quotes/:id/status` — `admin.quotes.view`; body `{status}`; validates the transition; returns updated `QuoteRecordDto`.

### 4.4 Front-end changes
1. `quoteDeliveryAdapter`: add `deliveryMethod: 'api'`; read `referenceId` from the response; keep mailto fallback and honest-failure messaging; **no silent loss** — on API failure the user sees the error with the existing phone fallback.
2. Replace `base44/adminQuoteAdapter.ts` with an HTTP adapter implementing the same two functions; delete the stub.
3. `AdminQuotesPage`/`QuoteDetailModal`: read `referenceId`; render `details.lines`.
4. Config: `VITE_QUOTE_DELIVERY_ENDPOINT` becomes `/api/quotes` (same origin).

### 4.5 Notification (open — §10 D-email)
A stored quote nobody is told about is the same lead loss as before. v1 needs an outbound email to `quotes@tfrsupply.com` on each submission. Requires a provider decision. Until chosen, the admin queue is the only notice.

### 4.6 Acceptance criteria
```
Given a valid quote payload from the PDP or the cart
When the user submits
Then the API stores it, returns a referenceId, and the same referenceId appears in the user's success message and in the admin queue

Given the same submissionId posted twice
Then exactly one record exists and both calls return the same referenceId

Given an invalid payload (missing name/agency/email, malformed email)
Then 422 and nothing stored

Given the API is unreachable
Then the user sees an honest failure with the phone fallback; no record is claimed

Given an admin without admin.quotes.view
Then GET/PATCH return 403 regardless of any client flag

Given status 'new' → 'quoted' (skip)
Then 409
```

## 5. Capabilities: admin auth, customer identity, fleet persistence

### 5.1 S2 — Real admin authentication

**Today.** `AdminAuthAdapter` = `listDemoUsers / signIn / signOut / getSession` (`adminAuthAdapter.ts:3-8`). The real service is **hard-bound to the mock adapter** (`adminAuthenticationService.ts:51`). `AdminSignInRequest` is `{demoUserId, requestId}` — **no credential field exists**, so this port cannot carry a real login unchanged. Four permissions exist exactly: `admin.shopify-sync.view`, `admin.quote-builder.view`, `admin.quotes.view`, `admin.pricing-imports.view`; four roles: `super-admin`, `ops-admin`, `sales-admin`, `viewer`. The token lives in `sessionStorage` and is passed as a function argument; there is no HTTP client in this flow.

**Design.**
- Staff accounts are **provisioned, not self-registered**: a server CLI/seed script creates users. No public signup endpoint.
- Email + password hashed with **Argon2id**; per-IP and per-account rate limiting and lockout.
- Server-side sessions in Postgres; opaque ID in an `HttpOnly; Secure; SameSite=Lax` cookie (works because of the same-origin proxy, §3.1). Idle and absolute expiry (the mock's `expiresAt: null` is not acceptable).
- Roles map to the **same four permission strings**; the server enforces them per endpoint.
- Port change: a new `HttpAdminAuthAdapter`; `AdminSignInRequest` gains `{email, password}` and loses `demoUserId`; `listDemoUsers` is removed (no real analogue); `AdminLoginPage` becomes a credential form; `adminAuthenticationService` singleton is re-bound; the separate `checkAdminAccess()` is deleted.
- `GET /api/admin/session`, `POST /api/admin/session` (login), `DELETE /api/admin/session` (logout), CSRF protection for state-changing calls.
- Open: MFA for `super-admin` (§10).

**Acceptance.** Wrong password/unknown email are indistinguishable to the caller; N failures lock; a viewer with no permissions gets 403 on every admin data endpoint; an expired session returns 401 and the guard redirects; with `VITE_ADMIN_ENABLED` unset the admin routes still 404 (unchanged).

### 5.2 S3 — Customer identity (Shopify Customer Accounts)

**Today.** `AuthContext.tsx` is a constant: `MOCK_USER {id:'mock-user', email:'demo@tfrsupply.com', role:'admin'}`, `isAuthenticated: true` always; every method is a no-op. Real consumers are few: `App.jsx` (loading/error gates), `lib/PageNotFound.jsx` (reads `user.role === 'admin'`), and an **unused** `ProtectedRoute.jsx`. The four `Login/Register/ForgotPassword/ResetPassword.tsx` pages are unrouted; their legacy `.jsx` twins import a non-existent `@/api/base44Client` and are dead. Cart, fleet, quotes and checkout do **not** read customer auth today, so the blast radius of making it real is small.

**Design (verified against Shopify's documentation on 2026-10-10).** Sources: `shopify.dev/docs/storefronts/headless/building-with-the-customer-account-api/getting-started`, `.../checkout-authentication`, and the Customer Account API reference `shopify.dev/docs/api/customer`.

1. **The backend is a confidential client.** Shopify describes confidential clients as applications "that have a back-end to perform the authorization request and a server-side session to hold the refresh token". That is this backend, in a backend-for-frontend arrangement. The SPA never handles Shopify tokens.
   - `GET /api/auth/login` — generates `state` and `nonce`, redirects to the discovered `authorization_endpoint` with `scope=openid email customer-account-api:full`, `response_type=code`, `redirect_uri`.
   - `GET /api/auth/callback` — verifies `state`; exchanges the code at the `token_endpoint` using the client id/secret; validates the returned `id_token` (signature via the discovered `jwks_uri`, and the `nonce`); stores the access and refresh tokens **server-side only**; upserts a `customers` row keyed by Shopify customer id (read via the Customer Account API); sets our own first-party session cookie.
   - Endpoints are **discovered at runtime** from `https://{shopDomain}/.well-known/openid-configuration` and `/.well-known/customer-account-api`, not hard-coded (Shopify's own guidance).
   - `GET /api/me` → `{isAuthenticated, user}` drives `AuthProvider`. Logout redirects to the `end_session_endpoint` with `id_token_hint`.
2. **`AuthProvider` becomes real.** `login()`/`logout()`/`navigateToLogin()` map to the routes above. `register`, `verifyOtp`, `resendOtp`, `requestPasswordReset`, `resetPassword` and `loginWithGoogle` are removed (Shopify owns all of it) and the four unrouted pages are deleted, not routed. `PageNotFound`'s `user.role === 'admin'` check is reconciled; customer sessions never carry admin.
3. **Logged-in checkout.** Shopify documents two ways to carry a signed-in buyer into checkout: set a `customerAccessToken` on the cart with `cartBuyerIdentityUpdate`, or append `sso=silent` to the `checkoutUrl` (which needs the buyer's active browser session on the Customer Accounts domain plus periodic `prompt=none` refresh; if that session lapses, checkout silently becomes a guest checkout). **Recommended: the cart-identity route, performed server-side** — `POST /api/checkout/identify {cartId}` calls `cartBuyerIdentityUpdate` with the stored customer token, so the token stays off the browser. The SPA keeps creating carts exactly as it does today and calls this only when signed in; guests are untouched. Shopify notes the cart must be updated again after any token refresh, and that customer-account pages such as order history may still ask the buyer to log in separately.

**Shopify-side setup (admin actions, from the getting-started guide).**
1. Settings → Customer accounts → under "Accounts in online store and checkout", choose **Customer accounts**.
2. Install the **Headless** sales channel; open its **Customer Account API settings**.
3. Choose **Confidential client**; copy the client ID and secret into Railway secrets (never `VITE_`-prefixed).
4. Register **Callback URL(s)** — Shopify requires **HTTPS** and does **not** support `localhost` or any `http` URL (use a tunnel for local development, and register the staging environment's URL separately); optional Logout URL.

**What the docs say about domains.** Discovery runs "on the storefront domain", and customer accounts "persist customer logins across online store, Hydrogen, and checkout". The pages read state **no requirement that a headless storefront share a root domain with Shopify's account domain**, and the cart-identity route above does not depend on a browser cookie shared with that domain. So a `railway.app` HTTPS URL is, on this evidence, registrable as a callback. That is the absence of a stated requirement in the pages read, not a guarantee — confirm in staging before relying on it.

**Rate limits (reference).** 100 points/second on the standard plan (200 on Advanced/Plus); 3,000 requests/minute per customer shared across all apps; over-limit returns `THROTTLED` / HTTP 429 with `Retry-After`. Not a constraint at expected volume.

**Still unverified** (verify in the staging environment; do not assume):
- (a) That the Customer Account API `access_token` is accepted as the `customerAccessToken` in the Storefront API's `cartBuyerIdentityUpdate` — the checkout-authentication page implies it, but the Storefront API reference was not read.
- (b) That Customer accounts and the Headless channel are enabled/installed on the `tfrsupply.com` store — not visible from this environment.
- (c) How pre-existing legacy customer records on that store behave at first sign-in.

**Acceptance.** Signed-out visitors browse, cart and check out exactly as today; signing in produces a real session that survives reload; sign-out ends the session server-side and calls Shopify's logout; no Shopify token or customer credential is ever sent to, stored in, or logged by the browser; a signed-in checkout arrives at Shopify identified as that customer (staging test, covers (a)); when the Shopify session or token lapses the user falls back to guest behavior with a clear signed-out state, never an error loop.

### 5.3 S4 — Fleet Workspace persistence

**Today.** All fleet state is browser-local with **no ownership, no versions, no schemas, and no storage port** — each of five Contexts calls `localStorage` directly with inline try/catch. Keys (flat `tfr_*`, unversioned): `tfr_fleet_builds` (+`activeBuildIdByProject`), `tfr_fleet_projects`, `tfr_fleet_templates`, `tfr_department_standards` (custom only; 12 shipped defaults are code, never persisted), `tfr_selected_vehicle`, `tfr_saved_products`, `tfr_compare_products`, `tfr_recently_viewed_products`, `tfr_upfit_builder`. IDs are client-generated `${prefix}-${Date.now()}-${rand}`; timestamps are **epoch-ms numbers**. Limits are client-side only: 25 builds/project, 20 projects, 50 templates/project, 50 standards. Cascade rules live in the Contexts, not the domain layer (delete-project removes its builds; duplicate-project clones builds and templates; per-project active-build map).

**What persists server-side (v1) vs stays local.**
| Persist for signed-in customers | Stays browser-local (ephemeral / device-specific) |
|---|---|
| fleet projects, fleet builds, fleet templates, custom department standards, saved products, selected vehicle | compare list, recently viewed, upfit-builder step/skip state, cart (cart persistence is launch-track blocker B8, localStorage) |

**Design.**
- Introduce a `FleetStoragePort` in the front end with two adapters: `localStorageAdapter` (today's behavior, used signed-out; first extraction is a **behavior-identical refactor** of the five Contexts) and `httpAdapter` (signed-in). This is the largest front-end change in v1 and is its own slice.
- Server tables keyed by `customer_id`; keep the client id as a unique `client_id` per customer (server `uuid` PK) so existing local data imports idempotently.
- **API shape matches today's TS types exactly** (epoch-ms numbers, same field names) to avoid front-end churn. New Zod schemas for all of them are part of the slice.
- Per-entity CRUD with `expectedVersion` optimistic concurrency — the repo already models this pattern in `adapters/quotePersistence/quoteRepository.ts` (`saveQuote/updateQuote(expectedVersion)`).
- Limits and cascades (delete-project, duplicate-project, active-build map) enforced **server-side in transactions**, reusing the pure domain functions in `src/domain/{fleetBuilds,fleetProjects,departmentStandards}` (which take `id`/`createdAt` as parameters, so the server can supply them).
- **First sign-in import:** `POST /api/fleet/import` accepts the local blob, validates, and writes idempotently on `client_id`. User is shown what will be imported and can decline; local data is not deleted until import succeeds.

**Acceptance.** Signed-out behavior is byte-for-byte unchanged (existing tests stay green); signing in on a second device shows the same projects/builds/templates; two concurrent edits of one build produce a `409`, not silent loss; the 26th build in a project is rejected server-side; deleting a project removes its builds and templates atomically.

## 6. Slices and order

| Slice | Content | Gate to start | Rough size* |
|---|---|---|---|
| **S0** Foundations | `server/` skeleton, `shared/contracts`, Railway service #2 + Postgres, `/api/health`, Caddy `/api` proxy, CI job (lint/typecheck/test/build for server + Postgres service container), env/secret handling, structured logging, migration runner | §10 D-stack confirmed | 1–2 days |
| **S1** Quotes + admin queue | §4 | S0 merged | 2–3 days |
| **S2** Admin auth | §5.1 | S1 merged | 2–3 days |
| **S3** Customer identity | §5.2 | S2 merged **and** Shopify admin setup done (§5.2) **and** the staging environment exists (callback URL + the (a)–(c) checks need it) | 3–4 days |
| **S4a** Fleet storage port | behavior-identical refactor of the 5 Contexts, no server | none (can run any time; no backend dependency) | 2–3 days |
| **S4b** Fleet server persistence + import | §5.3 server side + `httpAdapter` | S3 merged and S4a merged | 4–6 days |

\*Agent working time only, excluding review turnaround and any founder-side inputs. These are estimates, not commitments.

**Not in v1:** Shopify order/inventory/webhook sync (the ~12 stubbed foundation services stay frozen — Shopify remains the system of record), server-side dealer/contract pricing and a dealer portal, quote PDF, server-side cart.

## 7. Security and privacy requirements

- All input validated with Zod at the boundary; parameterized SQL only.
- Public `POST /api/quotes` is an open write endpoint: per-IP rate limit, body-size limit, and a spam control (honeypot at minimum; CAPTCHA is an open decision).
- Secrets only in Railway variables; never logged; never `VITE_`-prefixed. Logs exclude quote contact fields and request bodies.
- Quotes contain PII (name, agency, email, phone). The Privacy Policy page (launch blocker B9, not yet written) must describe this storage; retention period is an open decision.
- Postgres backups: confirm Railway's backup/restore posture before S1 holds real leads (not yet verified).
- Same-origin only: no `Access-Control-Allow-Origin: *`.

## 8. Testing

- Contract tests: every endpoint's request/response validated against `shared/contracts` from both sides.
- DB-backed integration tests against a CI Postgres container; no mocked database for persistence behavior.
- Front-end: existing 1,217-test suite must stay green on every slice (guest behavior unchanged); add adapter tests mirroring `tests/quote-delivery-adapter.test.mjs` (honest failure is asserted, not just success).
- Authorization tests per endpoint: unauthenticated → 401, wrong permission → 403.

## 9. Observability and operations

Structured JSON logs with request ids; `/api/health` (liveness) and `/api/ready` (DB reachable); Railway deploy health checks; an error-tracking decision (open). Staging: a second Railway environment with its own Postgres, so migrations are exercised before production.

## 10. Open decisions

| ID | Decision | Blocks | Recommendation |
|---|---|---|---|
| D-stack | Confirm Node/Fastify/Zod/Postgres/plain-SQL migrations (§3.2) | S0 | Accept as written |
| D-email | Provider for outbound quote-notification email (and later account mail) | S1 | Pick one transactional provider; until then admin queue is the only notice |
| D-domain | Production domain for the React storefront (`tfrsupply.com` is the Shopify store's primary domain; the app has only a `railway.app` URL). **Not a Shopify prerequisite per the docs read** (§5.2), but still needed for production: callback URL, cookie origin, brand | production launch (not S3) | Decide before go-live; likely a custom subdomain |
| D-shopify | ~~Read Shopify Customer Accounts docs~~ **Done 2026-10-10** (§5.2). Residual unknowns (a)–(c) are verified in staging | — | — |
| D-mfa | MFA for `super-admin` in v1 | S2 | Yes for super-admin; defer others |
| D-retention | Quote data retention period; privacy-policy wording | S1 | Set a period (e.g. 24 months) and state it |
| D-spam | CAPTCHA on public quote POST vs honeypot + rate limit only | S1 | Start with rate limit + honeypot; add CAPTCHA if abused |
| D-backups | Confirm Railway Postgres backup/restore posture | S1 | Verify before storing real leads |

## 11. Effect on the launch plan

None of §6 is on the storefront launch critical path. Launch blockers B1, B3, B4, B8, B9, B11 (`docs/MASTER_EXECUTION_PROGRAM.md` §3.1) proceed independently. When S1 lands, quote delivery stops depending on a third-party form service or the user's mail client, and the P0 data-loss risk (R7) is closed durably rather than by fallback. When S3/S4b land, the logged-in/logged-out design work becomes buildable.
