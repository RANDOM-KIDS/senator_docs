# Product Requirements Document (PRD)

**Project:** [Brand Name] — E-Commerce Platform
**Team size:** 3
**Target timeline:** 1 month (v1 — web only)
**Market:** Ghana

---

## 1. Overview

A single-brand e-commerce platform for a Ghanaian clothing brand: a web storefront for customers, an admin dashboard for the business, and a shared backend API. Payments via Paystack (primary) and manual mobile money confirmation (secondary). Delivery fulfilled through manual dispatch coordination in v1, with the schema and API designed to plug into a real third-party courier API once one is selected.

This is a **single-brand, single-seller** platform — not a multi-vendor marketplace. There is no plan to onboard other sellers in the current scope.

## 2. Goals

- Let customers browse the catalog, check out, and pay online without manual intervention for the common case.
- Give the business a working admin dashboard to manage products, inventory, orders, returns, and refunds without touching the database directly.
- Support the full order lifecycle a real clothing store needs: payment, dispatch, delivery confirmation, returns, and refunds — not just "place an order."
- Ship a genuinely usable v1 within one month with a 3-person team, by deferring the mobile app and live courier-API integration to v2, rather than compressing the full scope into an unrealistic timeline.

## 3. Non-Goals (explicitly out of scope for v1)

- **Mobile app (iOS/Android)** — confirmed as a v1 requirement conceptually, but **deferred to v2** given the 1-month timeline; building it alongside the web storefront in month one is not realistic for a 3-person team without cutting corners elsewhere.
- **Live third-party courier API integration** — no provider (Shaq Express, Mckot, Traksend, or other) has confirmed API access yet. v1 ships with manual/admin-updated delivery status instead of an automated dispatch integration.
- **Live GPS/map tracking** for deliveries.
- **Multi-vendor / multi-tenant support.**
- Any language beyond English.

## 4. Target Users

- **Customers** — browse and buy clothing online, in Ghana, paying via card/mobile money (through Paystack) or direct MoMo transfer with manual confirmation.
- **Admin/business staff** — manage the catalog, fulfill orders, handle returns and refunds, monitor sales, all via a web dashboard (no admin mobile requirement).

## 5. V1 Feature Scope

### 5.1 Customer-facing (web storefront)
- Product browsing: listing, search, category filter
- Product detail view with size/color variant selection
- Cart: add, update quantity, remove, view
- Checkout: select/add delivery address, place order
- Payment: Paystack checkout (primary path)
- Manual payment path: customer pays to a business MoMo number and awaits admin confirmation (order stays `pending` until confirmed)
- Order history and status view (`pending` → `paid` → `processing` → `shipped` → `delivered`)
- Return request: within 14 days of delivery, optional reason, tracked through to refund or rejection
- Account: sign up, sign in, session persistence

### 5.2 Admin dashboard (web)
- Auth: same login system as customers, gated by `role: admin`
- Product management: create, edit, (soft-)deactivate products and variants
- Inventory management: view and update stock per variant, low-stock visibility
- Order management: view all orders, update status, mark manual payments as paid, cancel orders (with automatic inventory restoration)
- Delivery status updates: manually advance an order's delivery sub-status (`requested → assigned → picked_up → en_route → delivered`) until a real courier API is integrated
- Returns management: view requests, approve/reject with reason, mark received, confirm inspection, trigger refund
- Notifications: in-dashboard list of new orders, return requests, and delivery failures, markable as read
- Sales dashboard: total orders, total revenue, orders by status

### 5.3 Backend / infrastructure
- All of the above backed by a documented REST API (see API Specification)
- Rate limiting on auth and write-heavy routes
- Transactional integrity on checkout (stock validation + decrement + order creation as one atomic operation)

## 6. V2+ (explicitly deferred, not designed against yet beyond the data model)

- Mobile app (Expo — iOS + Android), full customer-facing feature parity with web
- Real third-party dispatch/courier API integration (provider TBD)
- Live delivery location tracking
- Automated refund triggering (currently always admin-initiated by design, not necessarily a v2 change — revisit if volume demands it)
- Partial-return admin UI (data model already supports it; UI work deferred)

## 7. Success Criteria for v1

- A customer can complete a full purchase (browse → cart → checkout → pay via Paystack) without any manual intervention.
- An admin can fully manage the order lifecycle — from a new order notification through to marking it delivered — entirely from the dashboard, no direct database access required.
- A return can be requested, approved, received, inspected, and refunded entirely through the app + Paystack's refund API, with correct order/inventory state at every step.
- The system correctly handles the two payment paths (Paystack automatic, manual MoMo with admin confirmation) without ambiguity about which orders are actually paid.
- No critical bugs in the checkout transaction path (stock oversell, double-decrement, lost cart data) — this is the one part of the system that directly touches money and inventory, and needs to be bulletproof before launch.

## 8. Assumptions & Constraints

- Team of 3, working in parallel across roughly: backend completion, storefront UI, admin dashboard UI — with a final integration/testing phase before the 1-month mark.
- No fixed external deadline beyond the internally-set 1-month target — scope was deliberately shaped to fit this timeline rather than the timeline being stretched to fit an unshaped scope.
- No delivery provider is confirmed yet; this is a real dependency for v2 delivery automation and should be pursued in parallel with development, not after.
- Must support both Android and iOS eventually (mobile, v2) — no early architectural decision should make either platform harder to support later (this is why the backend was built API-first, shared by design, rather than coupled to the web frontend).

## 9. Risks

- **1-month timeline is tight even with web-only scope and 3 people** — the honest estimate for backend gaps + storefront + admin dashboard done sequentially by one person was 6–8.5 weeks; hitting 1 month depends on genuine parallelization and low coordination overhead. If it slips, the returns/refunds workflow (the most complex remaining piece) is the most defensible thing to soft-launch without, with manual handling as a stopgap — not something to cut from the PRD, but a candidate for "launch day one, harden day five" sequencing.
- **No delivery provider confirmed** — v1 leans on manual status updates as a deliberate stopgap, but this means real operational effort (someone manually updating delivery status per order) until a provider is integrated.
- **Manual payment path is a fraud/trust surface** — relies entirely on an admin correctly verifying a MoMo payment actually arrived before marking an order paid; no automated verification exists for this path by design.

## 10. Open Questions (carried from prior discussion)

- Confirmed dispatch provider selection — pending.
- Customer notification channel (email/SMS/WhatsApp/in-app) for order and return updates — not yet decided.
- Whether return items rejected on inspection are shipped back to the customer, and at whose cost — not yet decided.
