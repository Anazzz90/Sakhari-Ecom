# Product Experience Specification (PES)
## Sakhari Ecom — v1.0

**Status:** Draft for review
**Authors:** Product / UX Architecture
**Authoritative inputs:** SRD v2.6, DDD v1.0, Architecture Design Specification, ADRs 0001–0041, Engineering Standards & Development Guidelines, NFR Matrix
**Scope:** Customer Mobile App, Customer Website, Worker Mobile App, Admin Dashboard
**Out of scope:** Implementation details (frameworks, APIs, data models, component code). This document describes *experience*, not *construction*.

---

## How to Read This Document

This is the source of truth for how a human — customer, picker, rider, or admin — experiences Sakhari. It does not redesign anything the backend architecture has already decided (order lifecycle, payment methods, delivery batching, RBAC, localization scope). Where a business rule is referenced, it traces to the SRD, DDD, or an ADR. Where this document makes a *product* decision (tone, layout order, interaction pattern), that decision is new and owned by this spec.

Anything marked **Open Decision** (Section 35) is a genuine unresolved question — not a gap the author forgot to fill.

---

## 1. Product Vision

Sakhari exists so that a household in Sakaka never needs to plan a supermarket trip. Groceries arrive in the time it takes to realize you're out of something — reliably enough that customers stop keeping a mental backup list "just in case," because Sakhari is the backup list.

The product succeeds when ordering from Sakhari requires *less* thought than walking to a physical store: fewer decisions, fewer screens, fewer moments of doubt about whether the order will show up right.

## 2. Product Principles

1. **Speed is the product.** Every screen, every flow is evaluated first against: does this get the customer to "ordered" faster? Does this get the worker to "task complete" faster?
2. **Default to action, not exploration.** A customer who already knows what they want should never be forced through a product page. Discovery is optional; adding to cart is always one tap away.
3. **Trust is earned in the failure states, not the happy path.** How Sakhari handles an out-of-stock item, a failed payment, or a late rider says more about the brand than the home screen does.
4. **One honest state at a time.** The customer, worker, or admin should always be able to answer "what is happening right now?" in under a second of looking at the screen.
5. **Familiar over novel.** Where Instamart, Blinkit, Zepto, or Noon have already taught Saudi/regional shoppers a pattern, Sakhari uses that pattern unless there's a specific reason not to.
6. **Minimal does not mean empty.** Clean UI still needs to carry enough information (prices, ETAs, stock state) that the customer never has to tap to find out.
7. **Store-of-record accuracy over graceful workarounds.** Consistent with the backend's inventory-first philosophy (Section 5, 9), the product never papers over a stock problem with a UX trick (e.g., silent substitution) — it surfaces the truth and asks.

## 3. User Personas

**Customer — "Umm Faisal," 34, Sakaka.** Orders 3–5 times/week, mostly small restock trips (milk, bread, produce) plus occasional bigger weekly baskets. Price-sensitive but time-starved. Primary device: her own phone, occasionally the family iPad/browser. Wants to reorder her usual list in seconds, not rebuild it every time.

**Customer — "Faisal," 22, university student.** Late-night, small-basket, single-item impulse orders (snacks, drinks). Highly sensitive to delivery speed and minimum-order friction. Almost entirely mobile app.

**Picker — store-based worker.** Works inside one Dark Store, on shift, handling a queue of picking sessions. Needs the fastest possible path from "claim order" to "packed," minimal navigation, large touch targets (per NFR usability requirements), and a trustworthy substitution flow that doesn't put the decision on their shoulders.

**Rider — delivery worker.** Mobile-first, often outdoors, sometimes one-handed while managing a vehicle. May carry up to 2 orders in a batch. Needs the current stop obvious at a glance, the next stop visible but out of the way, and OTP-based delivery confirmation to be foolproof under bad lighting or spotty connectivity.

**Admin/Ops/Support — dashboard user.** Store manager watching one store's health; ops manager watching several; support agent triaging tickets; finance reconciling payment ledgers. Needs density and control (filters, bulk actions, exports) that a consumer app would never need — this persona actively wants more information on screen, not less.

## 4. Customer Experience Philosophy

The customer experience is built around **zero-friction restocking**. Every recurring customer eventually converges on a "usual list" — the product should recognize this and get out of the way. First-time and exploratory shopping is supported but treated as the secondary path, not the primary design target.

The experience never asks the customer to do work the system can already do for them: no manual store selection (the system resolves the serving Dark Store from the delivery address), no re-entering a delivery address each order, no rebuilding a cart from scratch when reordering.

Cart state is a *server-side, cross-device* concept — a customer who adds items on the website and finishes on the phone should see the same cart, because Sakhari treats "the cart" as one continuous session, not a per-device artifact.

## 5. Worker Experience Philosophy

Worker apps optimize for **task throughput under time pressure**, not exploration. A worker should never need to think about navigation — the app tells them what to do next, and getting there is the only decision left to make.

- **Single-capability workers** (picker-only or rider-only) skip mode selection entirely — the app opens directly into their one mode.
- **Dual-capability workers** choose a mode after login and cannot switch mid-task — this prevents a picker from being pulled into a delivery mid-pick, or vice versa.
- **Substitutions are rare by design.** The product does not build a rich "swap this item" browsing experience for pickers, because the intended fix for stock gaps is inventory accuracy, not a slicker substitution UI. When a substitution does happen, it is a narrow, constrained flow: picker flags the item as unavailable or proposes one specific substitute, the customer approves or rejects, and picking continues on other items in the meantime.
- **Rider mode is one-handed by design.** Current stop dominates the screen; upcoming stop(s) in a batch are visible but visually secondary; delivery confirmation is a single OTP-entry step, not a multi-field form.
- Manual search exists in both modes as an escape hatch (e.g., picker needs to find an item out of scan order, rider needs to check an address manually) but is never the primary interaction.

## 6. Admin Experience Philosophy

The Admin Dashboard is a **command center**, not a content-management tool. Its default view answers "is anything on fire right now?" — active orders, revenue, inventory alerts, low stock, customer complaints, driver status, and store health are the permanent above-the-fold view, not a page a manager has to navigate to.

Because RBAC is store-scoped, the dashboard experience changes shape by role: a single-store manager's default view *is* their store; a multi-store ops manager's default view is a store-comparison rollup with drill-down. The dashboard never asks a scoped user to pick from a list of stores they can't act on.

Enterprise data (orders, inventory, tickets, ledger entries) is always presented as filterable, exportable tables with saved filters and bulk actions — this is the one place in the product where density and control outrank simplicity, because the persona explicitly wants more information, not less.

Financially or operationally sensitive actions (refund approval, permission changes, ledger adjustments) surface a **step-up authentication challenge at the moment of the action**, not at login — the dashboard should make clear that this is a deliberate, elevated action, not a routine one.

## 7. Platform Strategy

| Platform | Primary user | Role |
|---|---|---|
| Customer Mobile App (Flutter) | Customer | Primary ordering channel — highest usage, richest interaction |
| Customer Website (Next.js) | Customer | Full feature parity where practical; desktop-optimized layout, not a scaled-up mobile screen |
| Worker Mobile App (Flutter) | Picker, Rider | Single-purpose task tool |
| Admin Dashboard (Next.js) | Admin, Ops, Support, Finance | Multi-store command center |

Feature parity between the customer mobile app and website is a goal, not a constraint on layout: a desktop shopper has more screen, a mouse, and no thumb-reach limitation, so the website earns the right to show more per view (e.g., multi-column product grids, persistent cart sidebar) while preserving the same information architecture and vocabulary as the app.

## 8. Navigation Philosophy

Customer bottom navigation: **Home · Categories · Search · Orders · Profile.**

Cart is deliberately **not** a navigation tab. A tab implies "a place you go to manage something"; Sakhari's cart is closer to a running receipt that should always be visible without ever needing a dedicated destination. It appears as a floating/sticky element (bottom-anchored bar or button) that persists across Home, Categories, Search, and Promotions, shows item count and running total, and expands into the cart review sheet on tap — consistent with the Instamart/Blinkit pattern.

Worker and Admin navigation are not tab-based in the consumer sense: the Worker App is close to single-screen-per-state (current task dominates; no tab bar competing for attention), and the Admin Dashboard uses a persistent side-rail appropriate to dense, multi-module enterprise navigation.

## 9. Information Architecture

**Customer Platform** is organized around three entry points into buying (Home, Categories, Search) and two entry points into managing what's already been bought (Orders, Profile). Product discovery and product commitment are deliberately decoupled — nothing in the IA forces a customer through a product detail page to buy something.

**Worker App** IA is task-state-driven, not menu-driven: what the worker sees is determined by what the worker is currently doing (idle/available, claimed a picking session, packing, en route), with manual search as a secondary utility, never a top-level destination.

**Admin Dashboard** IA is module-based (Orders, Inventory, Workers/Drivers, Customers/Support, Finance, Promotions, Stores/Settings), each module supporting the shared enterprise-table interaction pattern (filter, save filter, export, bulk action), scoped automatically to the signed-in user's Assigned Stores.

## 10. User Journey Principles

- **The fastest journey is always available.** A returning customer should be able to go from "opens app" to "ordered" without ever leaving Home, by re-adding items directly from Recently Bought or Recommended cards.
- **No journey requires knowledge the customer doesn't have yet.** A first-time customer isn't assumed to know Sakhari's categories or vocabulary — Search and Promotions carry the discovery weight.
- **Every journey ends at a state, not a dead end.** Checkout ends at Order Tracking, not a static confirmation screen. A failed payment ends at a retry action, not a wall.
- **Workers' journeys are sequences of single tasks**, never open-ended sessions — claim, pick, pack, hand off, repeat.
- **Admin journeys start from an alert or a number**, not a search — the dashboard's job is to make the next needed action obvious before the admin goes looking for it.

## 11. Product Discovery Experience

Home screen order, top to bottom:

1. **Delivery Address** — always visible at the top; tapping it is the only way to change the resolved Dark Store/service zone, reinforcing that store selection is never manual.
2. **Search Bar** — persistent, one tap from anywhere on Home.
3. **Promotions** — active, time-boxed offers.
4. **Categories** — visual entry into browsing.
5. **Recently Bought** — the fastest reorder path for the primary "restocking" persona.
6. **Recommended** — personalized surfacing.
7. **Popular** — social-proof / trending items.
8. **Products** — general catalog feed, the long tail of discovery.

This order is deliberate: address and search come first because they're the two fastest paths to "found what I need." Recently Bought sits above Recommended and Popular because a repeat customer's own history is a stronger predictor of what they want next than any generic ranking.

Categories and Promotions are entry points for browsing-mode customers; Recently Bought, Recommended, and Popular are entry points for restocking-mode customers. Both personas are served on the same screen without either one having to scroll past content meant for the other persona for long.

## 12. Search Experience

Search is treated as **the fastest way to buy**, not a fallback for when browsing fails. Result ranking, top to bottom:

1. Exact Matches
2. Brand Matches
3. Synonyms
4. Related Products
5. Categories
6. Previous Searches
7. Trending Searches

Every product variant (e.g., Almarai Milk 500ml / 1L / 2L) surfaces as its own distinct, addable result — search never collapses variants behind a single card that then requires a product-page visit to pick a size. Each result card carries its own **+ → (− 1 +)** quantity control, identical to the Home/Category/Promotion card pattern (Section 13), so a search result can be added without navigating away from the results list.

When a query returns zero exact/brand/synonym/related matches, the experience degrades gracefully into category suggestions and previous/trending searches rather than a bare "no results" message — search should always leave the customer with a next action.

## 13. Product Browsing Experience

Product cards are the atomic unit of the entire discovery surface (Home sections, Categories, Search results, Promotions) and behave identically wherever they appear:

- Card shows product image, name (with distinguishing variant detail — size/pack), price, and stock state.
- A **+** button adds one unit and immediately becomes an inline **(− 1 +)** stepper — no intermediate confirmation, no navigation away from the current screen.
- Tapping the card body (not the + control) opens the Product Detail page — this is an optional path, never required to purchase.
- Out-of-stock items (store-scoped, per the inventory model) are visually distinguished or hidden rather than shown as addable and then failing at checkout — the customer should never be able to add something the store doesn't actually have.
- Category browsing is a grid/list of the same cards, filterable and sortable, with no separate "category landing page" pattern beyond the card grid itself.

## 14. Product Detail Experience

The product page is optional infrastructure for customers who want more than the card gives them. It exists to answer questions the card can't: nutrition, ingredients, reviews, detailed product information, and — critically — **variant switching**, for a customer who opened one size but wants to compare or switch to another.

The product page still carries the same **+ / (− 1 +)** add-to-cart control as the card — opening the page is never a one-way commitment to "more steps," it's simply more information before the same one-tap add.

## 15. Cart Experience

The cart is a **persistent, always-reachable summary**, not a destination page reached through navigation. It manifests as:

- A floating/sticky bar or button, visible across Home, Categories, Search, and Promotions, showing item count and running subtotal.
- Tapping it expands a cart review surface (sheet or panel) listing line items, quantities (editable via the same − 1 + stepper), subtotal, and a path forward into checkout.
- Because cart state is server-side and shared across the Customer Mobile App and Customer Website, a customer switching devices mid-session sees the identical cart — this should be invisible/automatic, never something the customer has to trigger ("sync cart") themselves.
- The cart is where minimum-order and free-delivery-threshold feedback lives: if a basket is below the minimum order amount, the cart clearly states how much more is needed to check out; the same pattern applies to a free-delivery threshold, computed on the eligible subtotal.

## 16. Checkout Experience

Checkout is **one page**, top to bottom:

1. Address
2. Delivery Slot
3. Payment
4. Place Order

No multi-step wizard, no separate "review order" page inserted between cart and payment — the one-page layout itself is the review. Customers choose among the three available payment options at this step: **online payment** (mada / credit-debit card / Apple Pay via the payment gateway), **Cash on Delivery**, or **Card on Delivery** (rider-carried terminal, charged at drop-off). All three appear as equal, clearly labeled options — the product does not visually bury COD/Card-on-Delivery as a lesser option even though online payment is architecturally the "cleaner" path.

Because checkout can fail outright if any cart item can't be reserved at the requested quantity, checkout failure is never partial — the customer sees one clear reason (e.g., an item went out of stock at the last second) and one clear action: adjust the cart and retry. There is no state where "some of the order went through."

A successful checkout transitions immediately and automatically into **Order Tracking** — there is no static "thank you" confirmation screen that dead-ends the flow; tracking *is* the confirmation.

## 17. Order Tracking Experience

Order Tracking is a live, staged progress view the customer lands on immediately after placing an order and can always return to from the Orders tab. It should reflect the order's real backend progression (placed → store processing → picking → packed → rider assigned → out for delivery → delivered) as a small number of customer-legible stages — not the full internal state machine's granularity, but never contradicting it either.

Once a rider is assigned, tracking surfaces rider-specific information: rider en route status and, where the order is part of a delivery batch, the customer sees their own position in the trip without needing to know or care that another order shares the ride.

If an item requires substitution mid-fulfillment, the approve/reject decision surfaces as an interruption inside this same tracking experience — the customer should not need to leave tracking to respond to it.

If a delivery fails (customer unreachable, address issue, etc.), tracking reflects the failure state plainly and routes the customer toward support/resolution rather than leaving the last-known "out for delivery" status stale and misleading.

## 18. Delivery Experience

From the customer's side, delivery is a single continuous story: rider assigned → rider en route → rider nearby → arrived → delivered. Delivery confirmation is OTP-based — the customer may be asked to share or confirm a code with the rider at handoff; this should be framed as a simple, expected security step, not an obstacle.

Where a rider is carrying a batch of up to two orders, this is invisible to the customer as a concept — they experience their own delivery timeline; batching is purely an operational optimization, never a customer-facing idea ("your order is sharing a ride" is not something the customer needs to know or manage).

Card-on-Delivery payment is completed as part of the same handoff moment as OTP confirmation — from the customer's perspective, drop-off, payment (if applicable), and delivery confirmation happen together as one brief interaction, not three separate steps.

## 19. Notification Experience

High-priority, always-delivered notification moments: **Order Accepted, Out of Stock (substitution needed), Payment Failed, Driver Assigned, Driver Nearby, Delivery Completed, Promotions.**

Notifications are always a *reflection* of order/business state, never a dependency of it — the customer's order proceeds correctly even if a push notification is delayed or lost, so the product must never require the customer to act on a notification to keep an order moving (e.g., Order Tracking is always the source of truth, notifications are a convenience layer on top).

In-app notification history is shared and consistent across the Customer Mobile App and Website — a customer who dismissed a push on their phone should still find it in their notification history on web.

## 20. Error Experience

Errors are written and designed around **what the customer can do next**, never around the internal reason. Categories:

- **Recoverable / retryable** (payment gateway hiccup, transient network failure, checkout reservation conflict): clear restatement of what happened in plain language, and a single obvious retry action. Cash on Delivery / Card on Delivery remain available as a fallback path when online payment specifically is degraded, so an outage in the payment gateway is never a full stop for the customer.
- **Decision-required** (substitution approval, address outside service area): framed as a choice, not a failure — "here's what we can do" rather than "something went wrong."
- **Hard stops** (address outside every service zone): stated plainly and immediately, with a path to be notified when service expands, rather than a generic error.

Worker-facing errors (claim conflict, picking session expired, delivery rejected) are terse and action-first, consistent with the worker persona's time pressure — no explanatory paragraphs, just "this happened, here's your next tap."

## 21. Empty States

Every empty state gives the user a next action, never a dead end:

- Empty cart → surfaces Recently Bought / Recommended to restart the shopping motion immediately.
- No search results → falls into the same graceful-degradation ladder as Section 12 (categories, previous/trending searches).
- No orders yet (new customer) → invites first order via Categories/Promotions rather than showing a bare "no orders."
- Worker with no current task → clearly states "no task assigned" and shows what's next in queue, not a blank screen.
- Admin table with no results matching a filter → distinguishes "no data exists" from "no data matches this filter," with an obvious way to clear the filter.

## 22. Loading States

Loading is communicated honestly and locally: a section of Home loading its recommendations shouldn't block the rest of Home from being usable. Cart and checkout actions (add to cart, apply promo, place order) give immediate optimistic feedback where safe, with a clear rollback/error state if the backend ultimately rejects the action (e.g., a reservation conflict at checkout). Worker task transitions (claim, pack, mark delivered) show immediate in-progress feedback given the operational time pressure on that persona.

## 23. Offline Behaviour

The Customer Platform should degrade gracefully on poor connectivity rather than block: previously loaded catalog/cart content stays visible and browsable, with actions that need connectivity (add to cart, checkout, search) queued or clearly marked unavailable until reconnected, never silently failing. The Worker App, given the field-conditions persona (Section 3), needs the same tolerance for connectivity gaps — a rider or picker action taken just as connectivity drops should be retried automatically once reconnected rather than lost, with the worker told plainly that a sync is pending.

## 24. Accessibility

Sakhari's UI commitments (soft rounded, minimal, purposeful animation) must not come at the cost of legibility or usability: sufficient color contrast in both light and dark mode, tap targets sized for one-handed and gloved/field use (directly relevant to the Worker App), and no information conveyed by color alone (e.g., stock state, order status must carry text/iconography, not just a color chip). Arabic RTL layouts must mirror correctly, not just flip text direction while leaving iconography/flow untouched.

## 25. Localization

The Customer Platform (Mobile App and Website) supports **English and Arabic with full RTL layout** at launch — this is a hard MVP requirement, not a fast-follow. The Worker App and Admin Dashboard launch **English-only**, with Arabic as a confirmed fast-follow, not a "maybe." Product content (names, descriptions) must render Arabic text natively and correctly — never transliterated or lossily converted.

Currency displays as Saudi Riyal (SAR) throughout the customer experience; VAT is always shown as its own explicit line in order totals, never folded silently into the item price, consistent with Saudi tax transparency expectations.

## 26. Responsive Behaviour

The Customer Website is a desktop-optimized experience, not a stretched mobile layout: multi-column product grids, a persistently visible cart panel (rather than a floating button competing for thumb reach that doesn't exist on desktop), and layout density appropriate to a mouse-and-keyboard session — while preserving identical information architecture, vocabulary, and flow order (Address → Slot → Payment → Place Order) as the mobile app. The Admin Dashboard is desktop-first by nature of its persona and enterprise-table-heavy content, with tablet as the practical minimum supported breakpoint.

## 27. Dark Mode Strategy

Dark mode is a first-class, fully supported appearance across the Customer Mobile App, Customer Website, and Admin Dashboard — not an inverted color filter. Product photography, stock-state indicators, and brand accent colors must be legible and true in both modes. The Worker App, given frequent outdoor/bright-daylight field use, should default to whichever mode maximizes on-screen legibility in sunlight regardless of system-wide dark mode preference, while still respecting an explicit user override.

## 28. Animation Philosophy

Animation is used only where it clarifies state change, never as decoration: the + → (−1+) stepper transition, cart bar updating on add, order-tracking stage progression, and substitution/approval prompts appearing should all animate smoothly enough to feel intentional, but briefly enough to never be perceived as a delay. Nothing animates purely for delight at the cost of perceived speed — speed (Principle 1) always outranks polish when the two are in tension.

## 29. UX Writing Guidelines

- Plain, warm, direct language in both English and Arabic — no jargon inherited from the backend (customers never see internal state names like "Awaiting Rider" or "Picking Session Expired"; they see human equivalents like "Your order is being packed" or "Looking for a nearby rider").
- Error and empty-state copy always names the next action, not just the problem (Sections 20–21).
- Worker-facing copy is terse and imperative ("Claim next order," "Confirm delivery") — no marketing tone in operational tools.
- Admin-facing copy is precise and unambiguous, favoring clarity over friendliness — this is the one surface where a slightly clinical tone is correct, because the persona needs unambiguous operational facts.

## 30. Customer Trust Principles

- **Never claim a state that isn't true.** Order Tracking, stock availability, and delivery ETAs must reflect real backend state — no optimistic UI that could mislead about whether an item is actually available or an order is actually on its way.
- **Payment transparency.** Every charge, refund, and partial refund is traceable by the customer to a specific line item — consistent with the backend's line-item-aware partial refund model, refund history should never present as an unexplained lump sum.
- **Substitutions require consent.** No item is silently swapped for a different one and charged without the customer's explicit approval.
- **Failure is acknowledged, not hidden.** A failed delivery, a payment failure, or an out-of-stock situation is surfaced plainly and immediately, with a path forward — Sakhari's trust is built in how it handles the exception, not just the happy path (Principle 3).

## 31. Analytics & User Behaviour Tracking

Product experience should be instrumented to answer, at minimum: time to first product added, search success rate (query → add-to-cart without reformulation), cart-to-checkout conversion, checkout completion rate, substitution approval/rejection rate, and delivery satisfaction signals (on-time rate as perceived by the customer, not just backend SLA). Worker-side instrumentation should track claim-to-pack time and pack-to-handoff time as the core throughput signals for the picker and rider experience respectively. This section defines *what* to measure from a product standpoint; instrumentation mechanics are an implementation concern outside this document's scope.

## 32. Future Experience Opportunities

(Explicitly not MVP — noted here so future design work has a home, not to be built against now.)

- Arabic localization for Worker App and Admin Dashboard.
- Buy-now-pay-later presentation at checkout (Tabby/Tamara), once available.
- In-app turn-by-turn navigation for riders, replacing the current deep-link-to-Maps pattern.
- Photo-based proof-of-delivery alongside OTP.
- Richer, hand-drawn service-zone shapes replacing simple radius zones, which may change how "outside service area" messaging reads.
- Cross-store fallback/dispatch automation surfaced to ops, beyond today's manual reassignment.

## 33. Screen Inventory

Legend: **C-App** = Customer Mobile App, **C-Web** = Customer Website, **W-App** = Worker Mobile App, **Admin** = Admin Dashboard. "Shared" = same experience/IA intentionally mirrored across platforms (not literally the same codebase).

### Customer Platform (Shared across C-App and C-Web unless noted)

| Screen | C-App | C-Web | Notes |
|---|---|---|---|
| Onboarding / Phone + OTP Login | ✓ | ✓ | Phone-based OTP, no passwords |
| Home | ✓ | ✓ | Order per Section 11 |
| Categories | ✓ | ✓ | |
| Search Results | ✓ | ✓ | Ranking per Section 12 |
| Product Detail | ✓ | ✓ | Optional path |
| Cart Review (sheet/panel) | ✓ | ✓ | Floating on mobile, persistent panel on web |
| Checkout (one-page) | ✓ | ✓ | Address → Slot → Payment → Place Order |
| Order Tracking | ✓ | ✓ | Post-checkout landing |
| Substitution Approval Prompt | ✓ | ✓ | Interrupts tracking |
| Order History | ✓ | ✓ | |
| Order Detail / Refund Breakdown | ✓ | ✓ | Line-item-level refund display |
| Address Management | ✓ | ✓ | |
| Profile / Account | ✓ | ✓ | |
| Notification History | ✓ | ✓ | Shared, synced |
| Promotions Listing | ✓ | ✓ | |
| Outside-Service-Area State | ✓ | ✓ | |
| Support / Ticket Contact | ✓ | ✓ | |

### Worker Mobile App

| Screen | W-App |
|---|---|
| Login (admin-provisioned account, OTP) | ✓ |
| Mode Selection (dual-capability workers only) | ✓ |
| Availability / Shift Toggle | ✓ |
| Picking Queue / Current Picking Session | ✓ |
| Substitution Flag/Propose | ✓ |
| Packing Confirmation | ✓ |
| Delivery Assignment (single or batch) | ✓ |
| Batch Stop Sequence | ✓ |
| Navigation Hand-off (deep link) | ✓ |
| OTP Delivery Confirmation | ✓ |
| Cash / Card-on-Delivery Collection Confirmation | ✓ |
| Failed Delivery Reason Capture | ✓ |
| Manual Search (picker item lookup) | ✓ |

### Admin Dashboard

| Screen | Admin |
|---|---|
| Login (email/password) | ✓ |
| Step-Up Authentication Challenge | ✓ |
| Command Center / Store Health Overview | ✓ |
| Active Orders (table, store-scoped) | ✓ |
| Order Detail / Timeline | ✓ |
| Inventory Management & Low-Stock Alerts | ✓ |
| Worker Roster & Status (Pickers/Riders) | ✓ |
| Delivery / Rider Status Board | ✓ |
| Customer Complaints / Support Ticket Queue | ✓ |
| Refund Approval (step-up gated) | ✓ |
| Payment Ledger / Reconciliation | ✓ |
| Promotions Management | ✓ |
| Store & Service Zone Settings | ✓ |
| Roles & Permissions Management (step-up gated) | ✓ |
| Multi-Store Rollup / Comparison View | ✓ |

## 34. Experience Success Metrics

| Metric | What it tells us |
|---|---|
| Time to first product added | Discovery-to-intent speed (Home + Search effectiveness) |
| Search success rate | % of searches resulting in add-to-cart without query reformulation |
| Cart-to-checkout conversion | Friction in cart review / checkout entry |
| Checkout completion rate | Friction and failure rate within the one-page checkout |
| Time to checkout (cart open → order placed) | End-to-end ordering speed |
| Substitution approval rate & response time | Trust and clarity of the substitution flow |
| Delivery on-time perception (customer-reported vs. SLA) | Whether tracking accurately sets expectations |
| Customer effort score (post-delivery) | Overall experience friction, self-reported |
| Picker claim-to-pack time | Worker App throughput |
| Rider pickup-to-delivery time (per stop) | Worker App + batching effectiveness |
| Admin time-to-resolution on flagged orders/tickets | Admin dashboard's alerting effectiveness |

## 35. Open Decisions

Genuine unresolved experience questions — not yet decided, and not to be silently resolved by whoever designs the screen first:

1. **Substitution response timeout UX.** The backend removes an unapproved substitution after a configured timeout (SRD FR-SUB-003). What should the customer *see* as that timeout approaches — a countdown, a passive wait, or nothing until it resolves? Product has not yet decided the visible urgency level.
2. **Rider wait-time-at-door UX.** The backend caps rider wait time before a failed-delivery path is available. What, if anything, does the customer see counting down during this window?
3. **"Outside service area" waitlist mechanic.** Section 32 proposes notifying customers when service expands to their area — no decision yet on whether this is a simple "notify me" capture or a richer waitlist/referral mechanic.
4. **Minimum-order and free-delivery-threshold messaging tone.** Both are confirmed backend rules (Section 16), but whether the product nudges toward the threshold (e.g., suggested add-on items) or simply states the gap neutrally is undecided.
5. **VAT/ZATCA e-invoicing customer-facing requirements.** The SRD flags VAT treatment as pending accountant confirmation, with no ZATCA e-invoicing requirement currently specified. If e-invoicing is required, it will affect the order-confirmation and receipt experience — currently out of scope pending that legal/finance confirmation.
6. **Card-on-Delivery UX when the rider's POS terminal fails.** A backend failure/retry path is not detailed in the SRD for this specific hardware scenario; the in-app experience for the customer and rider in that moment is undecided.
7. **Batch visibility to the customer, if any.** Section 18 currently specifies batching is invisible to the customer. Whether the customer should ever see "your rider has one more stop before you" (which could set more honest ETA expectations) is an open product question, not a closed one.

---

*End of Product Experience Specification v1.0.*
