# TFRSupply Frontend — Forensic Salvage Manifest

> **Status (2026-10-10): the stack rewrite this document was written for is NOT happening.** On 2026-10-10 the decision was made to launch the *existing* application rather than rewrite it (see `docs/MASTER_EXECUTION_PROGRAM.md` for the active launch plan). This manifest is retained as **reference documentation only** — a cited code map of the domain logic, salvageable assets, and known traps as of the 2026-08-14 tree. Treat Section 4 ("Blueprint for the New Build") as historical, not as a plan. Citations are to the Aug-14 tree and may have drifted; verify against current `main` before relying on any `file:line`.

**Purpose:** A cited, evidence-based extraction of business logic, data contracts, and edge-case knowledge from this repository ahead of a stack rewrite. Every claim below is either cited to a `file:line` in the current tree (commit range ending 2026-08-14, branch `claude/code-review-automated-agents-m3cvl8`) or explicitly marked as inference/proposal. Where a `docs/architecture/*.md` file made a claim, it was cross-checked against the actual source it describes rather than trusted at face value — two cases where the doc was stale or self-corrected are called out explicitly in Section 3.

## Scope boundary (read this first)

- **Read in depth (near-exhaustive):** `src/domain/**` (all 45 files across 12 domains), `src/adapters/quoteDelivery`, `src/adapters/pricing`, `src/schemas/pricing.schema.ts`, `src/types/` (structure, not all 50 files line-by-line), `src/services/adminAccessService.ts`, `src/services/quoteRequestService.js`, `src/services/adminQuoteService.js`, `src/adapters/base44/adminQuoteAdapter.ts`, `src/components/configurator/ConfiguratorModule.tsx` (1045 lines, read in full), all 57 `docs/architecture/*.md` files (read or grep-sampled), `docs/engineering/REPO_HEALTH.md`, `docs/engineering/backlog/*`, `docs/architecture/CONFIGURATION_COMMERCE_ARCHITECTURE.md`.
- **Sampled (representative files read, siblings inferred from consistent pattern + doc cross-check):** the ~27 `shopify*` adapter/service/hook triads (pattern verified via `grep` for `unavailable`/`fetch(` across all of `src/adapters`, 3 read in full — Storefront, Storefront Catalog, Storefront Cart), `src/domain/fleetQuote` and `src/domain/procurementPackages` (3 of 12 and 3 of 11 files read respectively; the remainder follow the documented pattern in `docs/architecture/FLEET_QUOTE_BUILDER.md`/`FLEET_PROCUREMENT_PACKAGES.md`), `src/data/products/*.json` (churn-ranked, not content-read — treated as catalog content, not logic), UI components generally (read only where a component embeds business logic, e.g. `ConfiguratorModule.tsx`).
- **Skipped:** `node_modules`, `dist/`, `base44/` entity config, `src/pages/team44` (self-documented internal demo, `CONFIGURATION_COMMERCE_ARCHITECTURE.md:54`), `src/components/showcase`/`effects`, the ~403 individual `src/data/products/*.json` records (59 commits total across the directory, confirmed by churn — this is catalog data-entry churn, not logic churn), `scripts/shopify-catalog-ingest` and `scripts/shopify-variant-gid-overlay` internals (referenced but not read line-by-line).
- **Verification method:** `npm run test` was run in full — **1217/1217 tests pass, 0 failures** (`node --test tests/*.test.mjs`, 381 suites). Every Tier 1 claim in Section 2 was checked with `grep` for `from 'react'` / `from "react"` imports across `src/domain`, `src/services`, `src/adapters` — **zero matches**, confirming the "no React coupling" claim structurally, not just by reading comments.

---

## 1. Core Domain Logic & Business Rules

### 1.1 Domain model shape

The shared TypeScript contract layer lives in `src/types/` (50 files, `src/types/index.ts` barrel) with Zod runtime validation mirrored in `src/schemas/` (40 files). Documented dependency direction (`docs/architecture/DOMAIN_MODEL.md:40-50`, verified structurally — zero React/service imports found in `src/types`):

```
common → product → vehicle → configurator → commerce → package → quote
```

Core entities: `BaseEntity`, `Money`, `Dimensions`, `ImageAsset`, `Metadata` (common primitives); `Product`, `ProductFamily`, `Category`, `Vertical` (catalog); `Configurator`, `ConfiguratorSection`, `ConfiguratorOption`, `SKUOption` (configurator); `ShopifyProduct`, `ShopifyVariant`, `VariantMapping` (commerce/Shopify boundary — kept deliberately separate from pricing, `docs/architecture/PRICING_DOMAIN.md:7-13`); `Package`/`PackageLine`; `Quote`/`QuoteLine`/`QuotePayload` (`docs/architecture/DOMAIN_MODEL.md:21-30`).

### 1.2 SKU-matching / configurator engine — the live one

**Important correction the repo made about itself, verified directly against the tree:** `docs/architecture/CONFIGURATION_COMMERCE_ARCHITECTURE.md:21` and `docs/engineering/backlog/GH-289-unify-configurator-sku-matching-engines.md` both record that an earlier "pure domain engine" (`src/domain/configuration/configuratorEngine.js`) was **never actually wired into the app** — its React context was never mounted — and has been deleted. `src/docs/configurator-architecture.md:5` is kept only as a labeled-obsolete historical document ("do not build against it"). **The only live SKU-matching engine today is inline in `src/components/configurator/ConfiguratorModule.tsx`.** This is salvage-critical: don't extract the deleted pure engine's design (the JSON schema it describes is aspirational/historical); extract the actual `ConfiguratorModule.tsx` logic instead.

Live filtering algorithm, `filterSkus()` (`src/components/configurator/ConfiguratorModule.tsx:102-122`):
- Iterates configurator `steps[]`; each step maps to an attribute key (`step.skuSegmentKey`) on `ConfiguratorSkuOption.attributes`.
- For a step with a confirmed selection (`step._verification === 'confirmed'`), a SKU is excluded unless its attribute value exactly matches the selected option's `skuSegment`.
- For a step **not** marked `'confirmed'` (i.e. unverified data), the filter only excludes a SKU if *some* SKU in the full set has that attribute — this is a soft-filter that tolerates incomplete/unverified attribute data rather than silently producing zero matches. This asymmetry (confirmed vs. unverified stepping) is a non-obvious business rule: unverified attributes degrade gracefully instead of hard-filtering.
- `wouldHaveMatches()` (`:124-127`) is the dead-end-prevention check: before rendering a filter button as clickable, the module hypothetically applies that selection and checks if any SKU would still match. Disabled buttons are computed *before* the user can click a dead end, not after (`ConfiguratorModule.tsx:279`, `:284`).

**HKB vehicle-fitment-kit auto-bundle rule** (`ConfiguratorModule.tsx:59-98`, `findMatchingHkbKit()`): given a selected vehicle (make + first word of model, lower-cased) and the currently-resolved bar length, matches against a ground-truth kit list (`docs/knowledge/configurator_data.json` → `vehicle_fitment_kits.kits`) using loose substring containment on a free-text `vehicle` field, narrowed by `compatible_lengths` (e.g. Ford F-150 has two kits gated by bar length, 44"-48" vs 51"-53"). This is a real, non-obvious business rule requiring domain knowledge (the mapping data itself, not just the code) — flagged for careful re-verification in any rewrite since the matching is string-heuristic, not ID-based.

**Generic `auto_bundle` mechanism** (`:849-863`): any SKU-filter step option can declare `auto_bundle: [accessoryItemId, ...]` in configurator JSON; when that option is selected, referenced accessory items move from "optional, user-toggled" into "auto-included, non-removable" (`AccessoriesSection`, `:451-494`). This is the generalized mechanism the HKB kit rule is layered on top of.

**Review-flag assembly rule** (`quotePayload` `useMemo`, `:916-966`): a quote is flagged for manual review when any of: no vehicle selected; any selected step is `_verification: 'needs_verification'`; a required accessory has no known SKU; or the Shopify variant resolver itself reports a review flag (propagated, not re-derived, `:928`). `checkoutReady` is **only** true when the Shopify variant resolver reports `canAddToCart` (`:961`) — the UI's "Add to Cart" affordance is deliberately gated separately from "SKU matched," which is documented explicitly (`docs/architecture/CONFIGURATOR_EXPERIENCE.md:41-46`, `ConfiguratorModule.tsx:700-705`) as "one clear cart path."

### 1.3 Pricing engine — the second "good" domain

`src/domain/pricing/pricingEngine.ts` (161 lines, read in full) is real, tested arithmetic (not just architecture scaffolding, unlike much of this repo — see Section 3). Key functions, all pure:
- `money()`/`multiplyMoney()`/`addMoney()` (`:28-38`) — currency-safe rounding to 2 decimals via `Number(amount.toFixed(2))`.
- `calculateMargin()` (`:40-48`) — gross profit = revenue − cost; gross margin % = `(grossProfit/revenue)*100`, guarded against divide-by-zero (`revenue.amount === 0 ? 0 : ...`).
- **Price source priority hierarchy** (documented `docs/architecture/PRICING_DOMAIN.md:19-26`, implemented `pricingEngine.ts:24-26` `byPriority()`): active dealer/agency contract → active promotional bundle → dealer cost + margin policy → Federal Signal MSRP → (future) Shopify storefront price as fallback.
- `resolveUnitSellingPrice()` (`:64-66`) — the actual resolution order used at runtime: `input.requestedUnitPrice` (explicit override) → `quantityBreak.unitPrice` → `contractPrice.sellingPrice` → `listPrice.price`. Confirmed unchanged by the Quote Builder integration note (`docs/architecture/PRICING_DOMAIN.md:284`).
- `selectBestQuantityBreak()` (`:58-62`) — filters breaks where `quantity >= minQuantity && (maxQuantity == null || quantity <= maxQuantity)`, then picks the highest `minQuantity` (i.e., best/deepest applicable break, not first match).
- `priceQuoteLine()` (`:68-94`) emits `pricing.line.missing-selling-price` and `pricing.line.missing-dealer-cost` warnings rather than throwing or silently zeroing — an explicit "honest, not fabricated" pattern repeated throughout this codebase.
- **Dealer contract resolution** (`src/domain/dealerContractResolution/`, 4 files): `contractSelector.ts:17-41` selects the eligible contract by matching `contractId`/`dealerId`/`agencyId` context, requiring an eligible `ContractWindow` (`contractWindowEvaluator.ts`, classifies `active`/`upcoming`/`expired`/`expiring-soon` using `expirationAlertDays`), then sorts by `source.priority` descending. `quantityBreakSelector.ts` and `promotionalBundleResolver.ts` follow the same eligible-then-highest-priority pattern.
- **Live adapter status:** `src/adapters/pricing/livePricingAdapter.ts` exists and reuses `dealerContractResolutionService`, but the *default* `pricingService` still binds to `unavailablePricingAdapter` (`docs/architecture/PRICING_DOMAIN.md:251`) — the engine is real, but nothing calls it by default without an explicit adapter injection. **One documented layering violation**: `livePricingAdapter.ts` imports a *service* from an *adapter* — backwards per the stated dependency rules (`docs/architecture/CONFIGURATION_COMMERCE_ARCHITECTURE.md:87`, "Fix the one true layering violation").

### 1.4 Quote-request / delivery pipeline

Flow as implemented, traced end-to-end:
1. `ConfiguratorModule`'s `QuotePanel` opens `QuoteContactModal`, which calls `buildQuoteRequestPayload()` (`src/domain/configuratorQuote/buildQuoteRequestPayload.ts:25-76`, pure, zero side effects) — converts the internal `ConfiguratorQuotePayload` + captured contact info into the wire-shape `QuotePayload` the delivery adapter expects. Notably (`:46-55`): **required** accessories (e.g. an auto-matched HKB kit) are merged into the same `accessories[]` array as optional ones so sales never receives an incomplete package request — comment explicitly calls this out as a footgun avoided.
2. `src/services/quoteRequestService.js:55-57` — the sole submission entry point UI is meant to call; delegates to `quoteDeliveryAdapter.submitQuoteRequest()`.
3. `src/adapters/quoteDelivery/quoteDeliveryAdapter.ts` (193 lines, read in full) — **this is a genuinely-live adapter**, not a stub: if `VITE_QUOTE_DELIVERY_ENDPOINT` is configured it POSTs JSON to that hosted-form endpoint (`:165-186`); otherwise it falls back to opening a pre-filled `mailto:` link (`:188-190`). Comment at `:8-10` states the design intent explicitly: "Every outcome reported to the caller reflects something that actually happened; there is no fabricated success." Reference IDs are derived deterministically from the submission id's last 6 characters (`deriveReferenceId`, `:85-87`, format `QR-XXXXXX`).
4. **The gap:** nothing in this pipeline writes a durable record. Step 3's two outcomes (a third-party form POST, or an email client opening) are both fire-and-forget from this app's point of view. `src/pages/AdminQuotesPage.jsx` reads from `src/services/adminQuoteService.js:36-38`, which calls `fetchAllQuotes()` in `src/adapters/base44/adminQuoteAdapter.ts:16-19` — **confirmed by direct read**, this function is a stub that logs to console and unconditionally `return []`. `updateQuoteStatus()` (`:22-25`) is likewise a no-op that echoes its input back. **The admin quote queue is empty by construction today, regardless of how many quote requests are actually submitted.** This exact gap is independently corroborated by `docs/engineering/REPO_HEALTH.md:96-100` and `docs/engineering/archive/BACKLOG.md:39` (filed as issues #303/#307, P0, "Real customer/dealer quote submissions are silently dropped"). See Section 3 for the friction analysis.

### 1.5 Fleet quote / procurement readiness — deterministic scoring rules

Both engines below are explicitly documented as "no probability model, no AI," deterministic rule tables, and both explicitly reuse a shared, already-computed Guided Upfit Builder checklist rather than re-deriving completion facts (avoiding duplicate-source-of-truth bugs):

- `src/domain/fleetQuote/quoteReadiness.ts:17-57` (`resolveQuoteReadiness`) — priority order: no active project → `blocked`; zero vehicles → `blocked`; any missing-required-equipment or any vehicle missing → `incomplete`; missing department standard / missing-recommended / <100% completion → `minor_issues`; else `ready`. The UI's "Generate Project Quote" button visibility threshold is an explicit, easily-changed constant: `PROJECT_QUOTE_READY_LEVELS = ['ready', 'minor_issues']` (`:60`).
- `src/domain/procurementPackages/packageReadiness.ts:31-81` (`resolvePackageReadiness`) — same structure, four levels (`ready`/`minor_issues`/`needs_review`/`blocked`), documented priority table in the file's own comment (`:20-29`): missing vehicle or missing-required-equipment → `blocked`; duplicate vehicle configuration **or** missing department standard → `needs_review` (explicitly framed as "needs a human judgment call"); missing-recommended or <100% completion → `minor_issues`; else `ready`.
- `src/domain/departmentStandards/standardAssignment.ts:12-29` — a build's effective Department Standard resolves build-level assignment first, falling back to the parent Fleet Project's assignment, else `null`. Simple but load-bearing precedence rule for every downstream readiness computation above.

### 1.6 Shopify integration — live vs. stubbed (verified, not assumed)

`grep` across all 105 files in `src/adapters` for the `unavailable*` adapter pattern found **61 files** implementing or referencing an "unavailable by default" null-object adapter. Concretely, per `docs/architecture/SHOPIFY_STOREFRONT_API_FOUNDATION.md` (read in full) and direct file checks:
- **Architecture-only / stub (`execute()` never calls `fetch`)**: the base Shopify Storefront API foundation (`unavailableShopifyStorefrontAdapter` is the default; `liveShopifyStorefrontAdapter`'s `execute()` always returns `status: 'failed', error: 'live-calls-disabled'` and never calls `fetch`, `SHOPIFY_STOREFRONT_API_FOUNDATION.md:38-40`). Admin auth (`unavailableAdminAuthAdapter`, only `mockAdminAuthAdapter` is wired by default). Pricing (`unavailablePricingAdapter` is default). Vehicle fitment (`src/adapters/vehicleFitment/vehicleFitmentAdapter.ts`, 19 lines — confirmed unavailable-by-default per `CONFIGURATOR_EXPERIENCE.md:91-93`).
- **First genuinely live adapters** (confirmed via `grep` for real `fetch()` calls, not just comments): `src/adapters/catalog/liveShopifyStorefrontCatalogAdapter.ts` ("the first live-capable Storefront integration," `SHOPIFY_STOREFRONT_API_FOUNDATION.md:106`) and `src/adapters/shopifyStorefrontCart/liveShopifyStorefrontCartAdapter.ts` both contain real `fetch()` calls against the Storefront GraphQL endpoint, gated behind explicit config (never build-time env directly).
- **Admin/Base44 surfaces are explicitly retired-but-present**: `src/adapters/base44/` still exists for `adminQuoteAdapter.ts` (confirmed stub, Section 1.4) and other legacy flat services; `docs/architecture/CONFIGURATION_COMMERCE_ARCHITECTURE.md:49` classifies this as glue that stays with this storefront, not shared logic.
- **Net assessment**: this repo has extensive, well-typed, well-tested *scaffolding* for a live Shopify integration (schemas, adapters, services, hooks, dry-run mock adapters — 1217 passing tests attest the scaffolding itself is correct), but as of this commit only the catalog read path and the cart mutation path have a real network call behind them. Everything else (pricing, admin auth, vehicle fitment, most of Storefront) is an "unavailable adapter, honest UI" pattern (`CONFIGURATOR_EXPERIENCE.md:44`) — the UI renders truthfully that data isn't live rather than faking it.

---

## 2. Salvageable Assets Inventory

### Tier 1 — Copy-paste ready (zero React/global-state/env-var coupling, verified)

Verification method for every row: read the full file and grepped for `from 'react'`/`from "react"`, `window.`, `document.`, `import.meta.env`, and any module-scope mutable singleton. All rows below passed.

| Asset | Location | Notes |
| --- | --- | --- |
| Pricing arithmetic engine | `src/domain/pricing/pricingEngine.ts` (161 lines) | Pure functions only; only imports are `@/types` (type-only) and Zod schemas for validation contracts. No adapter, no fetch, no React. |
| Dealer contract resolution (4 files) | `src/domain/dealerContractResolution/{contractSelector,contractWindowEvaluator,quantityBreakSelector,promotionalBundleResolver}.ts` | Pure, deterministic selection logic over supplied arrays — no fetch, no adapter boundary needed by design (`PRICING_DOMAIN.md:194`). |
| SKU filtering algorithm | `filterSkus()` + `wouldHaveMatches()`, `src/components/configurator/ConfiguratorModule.tsx:102-127` | The two functions themselves take plain arrays/objects and return plain arrays — genuinely portable despite living in a `.tsx` file full of JSX elsewhere. Extract these two functions specifically, not the surrounding component. |
| HKB vehicle-fitment-kit matcher | `findMatchingHkbKit()`, `ConfiguratorModule.tsx:84-98` | Pure string-matching function; depends only on its three plain-object/array parameters. |
| Quote delivery adapter | `src/adapters/quoteDelivery/quoteDeliveryAdapter.ts` | Takes `fetch` and an `openMailto` callback as injected parameters (`:154-160`) — already dependency-injected, zero hidden globals. `buildQuoteEmailBody`/`buildQuoteMailtoUrl` (`:120-148`) are pure string builders, independently reusable. |
| Quote request payload builder | `src/domain/configuratorQuote/buildQuoteRequestPayload.ts` | Pure, explicitly documented as "side-effect free" in its own header comment. |
| Fleet/procurement readiness engines | `src/domain/fleetQuote/quoteReadiness.ts`, `src/domain/procurementPackages/packageReadiness.ts`, `src/domain/departmentStandards/standardAssignment.ts` | Pure scoring functions over typed input arrays; only coupling is to sibling `@/types/*` files (also portable). |
| Type contracts | `src/types/*.ts` (50 files) | Confirmed architecture-only: `DOMAIN_MODEL.md:36-37` states the rule ("no React, no service, no API-client imports") and a repo-wide coupling audit independently confirmed zero React imports in `src/domain`/`src/services`/`src/adapters` (`CONFIGURATION_COMMERCE_ARCHITECTURE.md:18`, re-verified directly by this audit's own `grep`). This is the single highest-value Tier 1 asset — it's a working, fairly complete domain vocabulary for a fleet-procurement storefront. |
| Zod validation schemas | `src/schemas/*.ts` (40 files) | Same verification; `src/schemas/pricing.schema.ts` read directly — pure Zod schema declarations paired 1:1 with `src/types/pricing.ts` interfaces via `z.ZodType<T>` casts, a clean pattern worth keeping as-is. |
| UI primitives | `src/components/ui/*` (shadcn/Radix kit, ~50 files) | Classified "Ready. Zero app-layer imports" (`CONFIGURATION_COMMERCE_ARCHITECTURE.md:40`) — not independently re-verified file-by-file by this audit (sampled, not exhaustive), but the shadcn convention plus the doc's own coupling audit make this a reasonably safe Tier 1 claim; treat as **Tier 1, doc-corroborated** rather than independently verified line-by-line. |

**Explicit non-claims:** `ConfiguratorModule.tsx` as a whole is **not** Tier 1 — it's ~1045 lines of React component with inline styles, `useState`/`useMemo`/`useCallback`, and context coupling (`useVehicle()`). Only the two extracted pure functions above are portable; the rest is Tier 2.

### Tier 2 — Refactor needed

| Asset | Location | Refactor need |
| --- | --- | --- |
| `ConfiguratorModule.tsx` (component shell around Tier-1 logic) | `src/components/configurator/ConfiguratorModule.tsx` | 1045 lines, one file, inline styles throughout (`FS`, `TH`, `TD` constants and hundreds of inline `style={{...}}` objects) — no design-token layer (`CONFIGURATION_COMMERCE_ARCHITECTURE.md:41`, 59 `#c8102e` hardcoded-color occurrences found by this audit's own `grep` across `src/components`, corroborating the doc's higher repo-wide figure of ~75). Also flagged by its own backlog item for splitting into subcomponents (`docs/engineering/backlog/GH-290-split-configuratormodule-into-subcomponents.md`, currently blocked/cancelled per GH-289's correction). Salvage the *behavior spec* (Section 1.2), rebuild the component fresh. |
| `ConfiguratorExperience` composition layer + panels | `src/components/configurator/{ConfiguratorExperience,ConfiguratorSummaryPanel,ConfiguratorPricingSummary,ConfiguratorCommerceActions,ConfiguratorFitmentFeedback}.tsx` | Well-factored composition (`CONFIGURATOR_EXPERIENCE.md`, read in full) but each panel is tightly coupled to specific hooks (`useListPrice`, `useDealerCost`, `useCartWorkspace`, `useProductFitment`) that in turn depend on the "unavailable adapter" pattern — the *fallback logic* (e.g. "MSRP falls back to catalog-sourced price when Pricing Engine is unavailable," `:95-107`) is worth re-reading as a spec, but the code itself needs a new data layer underneath it. |
| The ~27 `shopify*` service/adapter/hook triads | `src/{adapters,services,hooks}/shopify*` | Consistent, well-tested pattern (`createXService(adapter = unavailableXAdapter)` + Zod boundary validation, `CONFIGURATION_COMMERCE_ARCHITECTURE.md:23`) but most are pre-integration scaffolding, not working Shopify calls (Section 1.6). Salvage the *typed request/response contracts* (`src/types/shopifyStorefront.ts` etc.) directly; the adapters themselves need real implementations, not a port. |
| Admin dashboards (Sales, Sync, Pricing Import, Quote Builder) | `src/pages/Admin*.jsx`, `src/services/*Dashboard`, `src/domain/*Dashboard`, `src/domain/pricingImportDashboard/duplicateDetector.ts` | Functional against mock/fixture data (`PRICING_DOMAIN.md:209-241` describes the pricing import dashboard's mock adapters explicitly), but every one of them is UI wired to fixtures, not a real backend — refactor-needed to connect to whatever persistence the new stack chooses. |
| Cart / checkout preparation flow | `src/services/{cartWorkspace,checkoutPreparation}`, `src/components/cart/*` | Business rules (readiness gating, checkout blockers) are real and tested but layered over the same unavailable-by-default Shopify cart adapter pattern; the *readiness rule surface* is salvageable, the wiring under it is not. |

### Tier 3 — Trash / do not reuse

| Asset | Location | Why |
| --- | --- | --- |
| Client-side admin authentication | `src/services/adminAccessService.ts` | Self-documented in its own header as "NOT a real security boundary" (`:5`) — `checkAdminAccess()` unconditionally returns `{authorized: true, email: 'demo@tfrsupply.com'}` (`:19-21`). Confirmed live and unconditional, not hypothetical: nothing in the call chain gates this. Flagged Risk: Critical by the repo's own health process (`docs/engineering/REPO_HEALTH.md:95-100`, issue #297) for hardcoded demo credentials readable in the client bundle and effectively-unauthenticated `/admin/*` routes in production. **Do not port this pattern into the new stack even as a placeholder** — build real auth from day one. |
| Base44 admin quote adapter | `src/adapters/base44/adminQuoteAdapter.ts` | Confirmed-by-read stub: `fetchAllQuotes()` always returns `[]`, `updateQuoteStatus()` echoes input without persisting (`:16-25`). This is the concrete mechanism behind the P0 "quotes are silently dropped" finding (Section 1.4, Section 3). |
| Deleted-but-documented configurator engine | `src/docs/configurator-architecture.md` (doc only — code already deleted) | Describes `src/domain/configuration/{models,configuratorEngine}.js` and `ConfigurationContext.jsx`, none of which exist in the current tree (confirmed by `find`). The doc is explicitly labeled obsolete (`:5`) and kept only for historical context. Do not use this document as a design reference for the rewrite's configurator — see Section 3.1 for why it's a trap. |
| `src/components/navigator` hardcoded-SKU UI | `NavigatorTabs.jsx` (`NAVIGATOR_SKUS` hardcoded), `src/data/navigatorData.js`, `sampleData.js` | Explicitly classified "Superseded by JSON configurators; retire rather than extract" (`CONFIGURATION_COMMERCE_ARCHITECTURE.md:52`) — a product-family-specific component the platform's own governing principles (`ARCHITECTURE.md:64`) say shouldn't exist without approval. |
| Internal demo/showcase surfaces | `src/components/showcase`, `src/components/effects`, `src/pages/team44` | Self-documented as "not product code" (`CONFIGURATION_COMMERCE_ARCHITECTURE.md:54`). |

---

## 3. Forensic Post-Mortem & Friction Analysis

This is a young repository (186 commits, first commit 2026-07-06, most recent 2026-08-14 — roughly 5-6 weeks of history at the time of this audit) built under an explicit "architecture-only, additive, not-yet-wired" methodology (`SERVICE_LAYER.md:5`, repeated verbatim across dozens of docs). That methodology produced real, tested, well-typed contracts (Section 1, Section 2 Tier 1), but it also produced several specific, cited traps:

1. **Documentation described dead code as the live system for at least one full sprint cycle.** `src/docs/configurator-architecture.md` (last-updated header: "Sprint 14") described `src/domain/configuration/configuratorEngine.js` as *the* configurator engine — "Pure functions... Framework-independent... can be tested with plain Node.js" (`:205-206`) — while the actual, shipped configurator logic lived inline in `ConfiguratorModule.tsx` the entire time. The dead file's only "tests" were an unreferenced in-browser self-test harness (`docs/engineering/REPO_HEALTH.md:48-51`, `engineTests.js`). This was caught and corrected in PR #321/#323, and the correction itself is preserved as strikethrough text in `CONFIGURATION_COMMERCE_ARCHITECTURE.md:21` and a whole cancelled backlog item (`GH-289`) — a good process outcome, but the underlying cost was real: an entire architecture document, a backlog epic (#281 with children #288/#289/#290), and presumably reviewer/agent attention were spent reasoning about a system that didn't exist. **Trap for the rewrite: when "the pure engine lives in domain/, the messy version lives in the component" sounds architecturally clean, verify by checking what's actually mounted/imported at the app root — not by reading the prettier of the two files.**

2. **The "unavailable adapter by default" pattern, while individually well-reasoned per foundation, compounds into a repo where a majority of adapters are non-functional by design and it's easy to lose track of which.** 61 of 105 files under `src/adapters` reference the `unavailable*` pattern (Section 1.6, independently `grep`-verified). Each individual foundation document justifies this clearly ("preserves existing runtime behavior," `PRICING_DOMAIN.md:99-100`), but the aggregate effect — corroborated by the admin-auth and quote-pipeline findings below — is a codebase where "there's a service and an adapter for X" does not imply "X works." A rewrite team salvaging this code needs a single source of truth for live-vs-stub status per integration (this manifest's Section 1.6 table is a starting point, not exhaustive) rather than inferring it foundation-by-foundation.

3. **Real customer-facing data loss shipped silently and was only caught by a same-day-but-separate audit pass.** The quote-request submission path (Section 1.4) genuinely delivers (mailto/hosted-form, `quoteDeliveryAdapter.ts`), but nothing durably records the submission, and the admin queue that's supposed to show it reads from a stub that always returns empty (`adminQuoteAdapter.ts:16-19`, confirmed by direct read, not inferred). This was escalated as P0/Risk-High in `docs/engineering/REPO_HEALTH.md:96-100` and `docs/engineering/archive/BACKLOG.md:39` — the health report's own text says the finding "was never reflected in this file's escalation section" on first pass and had to be re-escalated 14 days later, still open at last recorded check. **Cost pattern: composing two "intentionally partial by design" pieces (a working outbound notification + a stubbed inbound store) can silently reconstitute a full data-loss bug even though each half is individually well-documented as partial.**

4. **Admin authentication is a placeholder that ships to production unconditionally.** `checkAdminAccess()` always returns authorized (`adminAccessService.ts:19-21`), confirmed live, not commented-out or feature-flagged off by default — `VITE_ADMIN_ENABLED` (`.env.example:6-10`) only controls whether `/admin/*` routes are compiled into the bundle at all, not whether they're authenticated once reachable. Flagged Risk: Critical by the repo's own process (`REPO_HEALTH.md:95-98`). **Trap: a comment saying "this is NOT a real security boundary" does not prevent the code from being real and reachable — treat self-documented insecurity as an active incident, not documentation.**

5. **Two duplicate date libraries shipped simultaneously.** `moment` (`^2.30.1`) and `date-fns` (`^3.6.0`) are both direct dependencies (`package.json`); this audit's own `grep` found `moment` imported in 4 files and `date-fns` imported in 0 — i.e. `date-fns` is dead weight in the bundle. Small in isolation, but a concrete instance of the "add a dependency per-feature without checking what's already there" pattern the repo's own dependency-health review flagged more broadly (`REPO_HEALTH.md:28-31`: 16 `npm audit` findings, several direct dependencies a major version behind, lodash carrying two high-severity advisories as a *direct* dependency).

6. **A build-time warning was silenced rather than fixed, and stayed silenced.** `vite.config.js`'s `logLevel: 'error'` override was found to suppress the build-size warning that would otherwise have surfaced a 1.9MB bundle on every build (`REPO_HEALTH.md:59-60`, filed as #292). **Trap: don't inherit CI/build configuration wholesale from a repo whose own health process flagged it as actively hiding a real signal.**

7. **~75 files (59 confirmed independently by this audit's `grep` for the single color `#c8102e` alone, across `src/components`) hardcode brand color and font values inline rather than through a token layer.** `CONFIGURATION_COMMERCE_ARCHITECTURE.md:41` calls this "the largest single extraction cost" for any UI reuse. Confirms the domain/logic layers (Section 1-2 above) were kept clean while the UI layer was allowed to accumulate this debt — a real, if unsurprising, split in code quality by layer worth replicating deliberately in the rewrite (keep the discipline that produced Tier 1, apply equal discipline to styling from day one instead of retrofitting a token layer later).

8. **A stale-docs claim this audit could *not* corroborate, cited here so nothing is silently trusted:** `REPO_HEALTH.md:40-41` (2026-07-09 entry) states "3 stale zip archives at repository root." This audit's own `find . -maxdepth 1 -iname "*.zip"` at the time of writing found **zero** zip files at the repo root — either already cleaned up since that report (most likely, given the report itself is 5+ weeks old relative to this audit) or the finding was imprecise even then. **This is exactly the failure mode this manifest's own methodology is designed to avoid — a plausible-sounding, previously-published claim that doesn't hold up against the current tree. Flagging rather than repeating it.**

9. **High churn in `src/data` (59 commits, the single highest-churn directory measured — more than double `src/components`'s 27 and `src/pages`'s 18) is catalog data-entry churn, not logic churn**, confirmed by sampling the top individual files: `light-bars.json` (5 commits), several individual product JSONs (3-4 commits each) — these are incremental product-record edits, not architectural rework. This is a useful negative finding: it means the *volume* of historical change in this repo is concentrated in content maintenance, not in reworking business logic, which is consistent with Section 1's finding that the domain layer (`src/domain`, 4 top-level commits measured, but see caveat below) is comparatively young and stable.
   *Caveat on the churn numbers themselves*: because this repository is only ~6 weeks old, most individual `src/domain/*`/`src/services/*`/`src/adapters/*` subdirectories show exactly 1 commit each (`git log --format=oneline -- <dir> | wc -l`) — each foundation was evidently built and merged in a single pass rather than iterated on repeatedly. This flattens the churn signal for young, feature-scoped repos: **low subdirectory churn here means "not yet revisited," not "stable under real usage,"** since there hasn't been enough elapsed time or production traffic for a second pass to be needed yet. Treat the churn signal in Section 3 as directional, not a substitute for actually reading the code (which this audit did for the sections above).

---

## 4. Blueprint for the New Build

Everything in this section traces to a Section 1/2 citation unless explicitly marked **NEW, not salvaged**.

### 4.1 Core entity-relationship model (salvaged from `src/types/*`, Section 1.1)

```
Vertical (Police | Fire/EMS | Work Truck)
  └─ Category
       └─ Product / ProductFamily
            └─ Configurator
                 ├─ ConfiguratorSection[] (skuSelector, accessories, ...)
                 │    └─ ConfiguratorStep[] → ConfiguratorOption[] (skuSegment, auto_bundle[])
                 └─ ConfiguratorSkuOption[] (sku, attributes{}) ── the source of truth SKU list

Vehicle (year, make, model) ── VehicleFitmentKit (matched by make+model+length, Section 1.2)

PricingSubject (sku, productId?, variantId?)
  ├─ ListPrice, DealerCost, ContractPrice (each carries a PriceSource{priority})
  ├─ DealerContract → ContractWindow (active/upcoming/expired/expiring-soon)
  ├─ QuantityBreak[] (minQuantity, maxQuantity?, unitPrice)
  └─ PromotionalBundle (SKU membership + optional promo code + window)

QuoteRequest (buildQuoteRequestPayload shape, Section 1.4)
  ├─ selectedBaseSku, selectedFilters{}, accessorySkus[], requiredComponents[]
  ├─ ContactInfo (name, agency, email, phone?, notes?)
  └─ reviewFlags[] (vehicle missing / unverified attribute / no-SKU required component / variant-resolution flag)

FleetProject
  └─ FleetBuild[] (vehicle, departmentStandardId?)
       └─ DepartmentStandard (resolved: build-level → project-level fallback, Section 1.5)
            └─ QuoteReadiness / PackageReadiness (deterministic 4-level scoring, Section 1.5)
```

### 4.2 Core user flows (as step-by-step specs, salvaged behavior)

**Flow A — Configure and request a quote** (salvaged from `ConfiguratorModule.tsx` + `buildQuoteRequestPayload.ts`, Section 1.2/1.4):
1. User selects a vehicle (or skips — the flow degrades gracefully, flagged for review rather than blocked, Section 1.2).
2. User picks configurator filter steps one at a time; after each pick, recompute the matching SKU set (`filterSkus`) and recompute which *next* options would still leave ≥1 match (`wouldHaveMatches`) — disable dead-end options before the click, not after.
3. When the filtered SKU set narrows to exactly 1, auto/manually select it as the base SKU; resolve its live commerce data (price, variant id, cart-eligibility) as a *separate* concern from the filtering state (Section 1.2's "UI selection vs. variant resolution" split — worth keeping as an explicit architectural seam in the rewrite, not re-merging).
4. Compute any vehicle-based auto-bundle accessories (HKB-style kit matching) and merge them into the package as always-included, non-removable lines.
5. Let the user toggle optional accessories.
6. Assemble review flags (missing vehicle, unverified attribute, missing SKU on a required component, unresolved variant) — surface all of them, never silently drop one to keep the UI clean.
7. On "Add to Quote," collect contact info, build the wire payload (merging required + optional accessory lines, Section 1.4 step 1), and deliver it through **two** paths in priority order: POST to a configured hosted-form endpoint; else open a pre-filled mailto link. Report only outcomes that actually happened (Section 1.4 step 3's design principle) — never fabricate a success.
8. **NEW, not salvaged**: persist the submission durably (a real backend/database) so an admin queue can read it back. This repo never actually did this (Section 1.4 step 4, Section 3.3) — the old admin queue read from a stub that always returned empty. This is the single most important gap to close on day one of the rewrite, not defer.

**Flow B — Price a quote line** (salvaged from `pricingEngine.ts`, Section 1.3):
1. Given a `PricingSubject` (SKU + optional product/variant id) and requested quantity, resolve the unit selling price in this fixed order: explicit override → best-fit quantity break → contract price → list price.
2. If a dealer contract exists, it must pass its `ContractWindow` eligibility check and be the highest-priority (`PriceSource.priority`) match among eligible candidates.
3. Compute line totals (`unitPrice × quantity`), margin (`revenue − cost`, `%` guarded against divide-by-zero), and emit non-fatal warnings for missing selling price or missing dealer cost — never block the flow on missing pricing data, flag it instead.
4. Sum lines into a quote subtotal and aggregate margin; propagate all line-level warnings up to the quote level.

**Flow C — Fleet build readiness → quote generation** (salvaged from `quoteReadiness.ts`/`packageReadiness.ts`, Section 1.5):
1. Resolve each build's effective Department Standard (build-level assignment, else project-level, else none).
2. Reuse a single "Guided Upfit Builder checklist" computation per build (percent complete, missing required/recommended items) as the shared fact base for every readiness rollup — do not let two different readiness features recompute completion independently (this repo's own comments flag this as a deliberate anti-duplication choice worth keeping, `quoteReadiness.ts:14`, `packageReadiness.ts:16-18`).
3. Apply the fixed-priority rule table (blocked → needs_review/incomplete → minor_issues → ready) per feature; expose the threshold for "quote generation allowed" as a simple, obviously-editable constant, not logic buried in a conditional (`PROJECT_QUOTE_READY_LEVELS`, Section 1.5).

### 4.3 Integration seams — what to build fresh vs. what to keep as a contract

| Seam | Recommendation |
| --- | --- |
| Shopify Storefront/Admin API | **NEW implementation, salvaged contracts.** Keep the typed request/response shapes (`src/types/shopifyStorefront.ts` family) and the adapter-interface pattern (`execute()`/`getAvailability()`) as the target shape; build real `fetch()`-based adapters against them from day one rather than re-running the "unavailable by default" scaffolding phase (Section 3, finding 2) — this repo spent real effort building and re-building that scaffold across ~27 Shopify integration surfaces before most of them went live. |
| Quote persistence + admin queue | **NEW, not salvaged** — genuinely never existed as working code here (Section 1.4, Section 3.3). Do this first; it's the most concretely-proven gap in the old system. |
| Admin authentication | **NEW, not salvaged.** Do not reuse `checkAdminAccess()`'s pattern even temporarily (Section 2, Tier 3, Section 3.4) — build real auth (even a minimal real one) before any `/admin/*` route ships. |
| Configurator SKU-matching | **Salvage the algorithm** (Section 1.2's `filterSkus`/`wouldHaveMatches`/auto-bundle rules) as a fresh, framework-independent pure module from the start — this repo's own history shows what happens when the "pure engine" and the "shipped implementation" drift apart (Section 3, finding 1); write the pure engine and use it directly, don't let a component re-implement it inline again. |
| Pricing resolution | **Salvage directly** — `pricingEngine.ts` is close to portable as-is (Tier 1, Section 2); wire it to real data sources instead of `unavailablePricingAdapter`. |
| Fleet/procurement readiness rules | **Salvage directly** — small, pure, well-specified (Section 1.5); the main new work is a real persistence layer underneath, not new business logic. |
| UI/design system | **NEW, not salvaged**, deliberately. The old inline-style, hardcoded-color approach (Section 3, finding 7) is exactly the kind of debt worth not re-inheriting — start with a token layer from day one rather than retrofitting one, which this repo's own architecture doc identifies as its largest unresolved extraction cost. |

---

*Compiled by a forensic salvage audit of `TFRSupply-frontend` at commit range ending 2026-08-14. All test evidence: `npm run test` → 1217/1217 passing. No application code was modified in producing this document.*
