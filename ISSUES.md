# Tracked Issues

No GitHub connector/CLI was available in this environment when these were found, so they're recorded here to be pasted into GitHub manually.

---

## Issue 1: changePlan does not persist new plan locally (stale plan until webhook)

**Label:** bug

**Body:**

`SubscriptionService.changePlan` (`payments-ms/src/subscription/subscription.service.ts`, upgrade + downgrade branches) calls Stripe correctly but never saves the new plan locally; the DB `plan` column only updates when the `customer.subscription.updated` webhook arrives. Because that webhook path has no re-verification in payments-ms and silently drops unknown event types, a lost/delayed webhook leaves `GET /subscription/me` showing a stale plan with no reconciliation path.

Pinned by tests in `payments-ms/src/subscription/subscription.service.spec.ts` (changePlan → UPGRADE/DOWNGRADE tests) that will go red when fixed.

---

## Issue 2: resume silently discards a pending downgrade schedule

**Label:** question / needs-triage

**Body:**

`SubscriptionService.resume` (`payments-ms/src/subscription/subscription.service.ts`) unconditionally calls `releaseScheduleIfAny` before undoing cancellation, so resuming after scheduling a downgrade silently drops the scheduled plan change, with no user-facing signal. May be intentional ("resume = restore exactly as before") but is non-obvious from the API contract. Needs product-intent confirmation, not a code fix yet.

Pinned by a test in `payments-ms/src/subscription/subscription.service.spec.ts` (resume → schedule-discard test).

---

## Issue 3: /analytics all-or-nothing fan-out — FIXED

**Label:** bug (resolved)

**Body:**

`AnalyticsService.getAnalytics` (`client-gateway/src/analytics/analytics.service.ts`) fanned out to 6 RPC calls (orders-ms x4, payments-ms x1, auth-ms x1) via a bare `Promise.all`, so any single downstream failure rejected the entire `/analytics` response — one service being briefly unavailable took out the whole dashboard, even though the other 5 calls may have succeeded.

**Resolution:** each of the 6 calls is now wrapped individually (`AnalyticsService.safeFetch`) so a rejection degrades only that section to a zeroed/empty fallback instead of failing the whole request.

**New response shape (additive only — happy-path fields unchanged):**
- A new top-level `unavailableSections: string[]` field lists which of the 6 section keys (`overview`, `salesByType`, `topProducts`, `categoryBreakdown`, `paymentTotals`, `customerGrowth`) failed and fell back to a default. Empty array when everything succeeds.
- On failure, a section's contributed fields default to zero/empty rather than being omitted:
  - `overview` → `revenueTotal/revenueGrowth/ordersTotal/ordersGrowth: 0`, `series: []` (cascades to `revenueSeries`, `seriesLabels`, `growth`, `growthChange` all being `0`/`[]`)
  - `salesByType` → `salesDistribution: []`
  - `topProducts` → `topProducts: []`
  - `categoryBreakdown` → `categoryDistribution: []`
  - `paymentTotals` → `paymentBreakdown: []`
  - `customerGrowth` → `customerGrowth/customerGrowthChange/totalCustomers/newClients/clientsChange: 0`
- Frontend consumers should check `unavailableSections` to distinguish "genuinely zero" from "temporarily unavailable" rather than trusting a zero value alone.

Tests: `client-gateway/src/analytics/analytics.service.spec.ts` (happy path, single-section failure, multi-section failure).

---

## Issue 4: /orders silently degrades product snapshot on product-ms failure — OPEN, needs a decision

**Label:** question / needs-triage

**Body:**

`OrdersService.create` (`client-gateway/src/orders-ms/orders/orders.service.ts`) resolves each order item's product-category snapshot (`countsTowardKitchenCapacity`, `categoryId`, `categoryName`) by calling product-ms before `order.create`. Each per-product lookup is wrapped in `.catch(() => null)`, so if product-ms is down or a product was deleted, the order is still created with default/degraded snapshot data (`countsTowardKitchenCapacity: true`, category fields `undefined`) — silently, with no indication to the caller that anything failed.

Pinned by `client-gateway/src/orders-ms/orders/orders.service.spec.ts` (partial-failure test) so current behavior stays locked until a decision is made.

**Fix options (pick one):**

1. **Fail loud** — if any product-ms lookup fails, reject the whole order before `order.create` runs. Guarantees data integrity (no order with a wrong/default kitchen-capacity flag or missing category), but is a functional behavior change: an order that succeeds today (with degraded data) would start failing outright whenever product-ms has a hiccup, even a transient one. Riskiest for order-taking uptime during a product-ms blip.
2. **Create but surface** — keep creating the order with the current default-degrade logic, but stop doing it silently: return which `productId`s failed to resolve (e.g. a `snapshotWarnings: string[]` field) so the caller/UI can flag it. No functional change to whether orders succeed; adds visibility only. Requires a small response-shape addition (additive, similar to `unavailableSections` above) but no change to create-order availability.
3. **Keep as-is** — accept the current silent degrade permanently. Only reasonable if the conservative default (`countsTowardKitchenCapacity: true`) is considered an acceptable worst case and category display is non-critical. Lowest effort, but leaves the "no indication anything failed" gap open.

No code change has been made for this issue — awaiting a decision on which option to take.
