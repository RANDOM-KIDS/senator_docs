# Operations & Business Flow Specification

## 1. Order Lifecycle

### 1.1 Primary statuses

```
pending ──▶ paid ──▶ processing ──▶ shipped ──▶ delivered
   │           │           │
   └───────────┴───────────┴──▶ cancelled
```

| Status | Meaning | Entered when |
|---|---|---|
| `pending` | Order created, awaiting payment | Customer completes checkout (cart → order) |
| `paid` | Payment confirmed | Paystack webhook confirms payment, **or** admin manually marks paid (offline/MoMo payment) |
| `processing` | Order is being prepared for dispatch | Immediately on entering `paid` — no separate manual "review" gate |
| `shipped` | A courier has been requested/assigned via the dispatch provider | Admin/system requests a courier through the dispatch integration |
| `delivered` | Courier confirms delivery | Dispatch provider reports delivery complete (webhook or admin confirmation, depending on provider capability) |
| `cancelled` | Order will not be fulfilled | Customer self-cancels (`pending`/`paid`/`processing` only) or admin cancels (any status, exception path) |

`paid → processing` happens automatically and immediately — there is no manual admin review step between payment confirmation and preparation.

### 1.2 Cancellation rules

- **Customer self-cancellation**: allowed while status is `pending`, `paid`, or `processing`. Not available once `shipped` — a courier has already been requested, typically at a cost to the business.
- **Admin cancellation**: allowed at any status, including `shipped`, as an exception path (e.g. courier request failed, customer called in). Cancelling a `shipped` order requires admin judgment about whether/how to recall the courier — this is a manual coordination problem, not something the system automates.
- **Inventory on cancellation**: always restored. Cancelling an order returns the reserved quantity back to `inventory.quantity` for each affected variant, regardless of who cancelled it or at what status.
- Cancelling a `paid` order triggers the refund flow (§3) rather than a simple status change — money already collected must be returned.

### 1.3 Inventory decrement timing

Stock decrements at **order creation** (checkout), not at cart-add. This is a first-come-first-served model: if two customers have the same last unit in their carts, whichever one completes checkout first succeeds; the other receives an "insufficient stock" error at their own checkout attempt. No cart-level reservation/hold exists.

---

## 2. Delivery / Dispatch

### 2.1 Model

Delivery is fulfilled through a **third-party dispatch courier API** (candidate providers: Shaq Express, Mckot, Traksend — none integrated yet; design is provider-agnostic so any can be plugged in). No live provider API access exists at time of writing — the system is designed against a generic contract, not a specific provider's real API shape.

### 2.2 Delivery sub-status (tracked independently of order status)

Order-level status only reaches as far as `shipped`. The granular delivery progress lives in its own record, separate from the order:

```
requested ──▶ assigned ──▶ picked_up ──▶ en_route ──▶ delivered
                                                    └──▶ failed
```

- Displayed to the customer as a simple status label (e.g. "Courier assigned," "Picked up," "On the way," "Delivered"). No live map/GPS tracking in this version — the schema reserves optional fields for a future provider that supports live location, but nothing populates them yet.
- `delivered` at the delivery-record level triggers the order's own status to advance to `delivered`.
- `failed` (delivery attempt failed — customer unreachable, address issue, etc.) requires admin intervention; does not automatically cancel or refund the order.

### 2.3 Handoff communication

"Communication between the two parties" (business ↔ dispatch provider) happens via whatever integration mechanism the eventual provider supports — likely a request/response API call to request a courier, plus either a webhook or polling for status updates. The exact mechanism is provider-specific and will be finalized once a provider is selected; the system's job is to record the resulting status changes regardless of how they arrive.

---

## 3. Refunds

### 3.1 When a refund applies

- Order cancelled after payment (`paid`, `processing`, or admin-cancelled `shipped`)
- Return approved and inspected successfully (§4)

### 3.2 Refund mechanism

- Refunds are triggered manually by an admin action (not automatic) — admin explicitly initiates the refund via Paystack's refund API.
- Paystack processes the refund and sends a webhook confirming completion.
- Refund amount **excludes the original delivery fee** — only the product cost is refunded.
- Refund status is tracked (`initiated` → `completed`) pending the confirming webhook; the order/return is not considered fully closed until that confirmation arrives.

---

## 4. Returns

### 4.1 Eligibility

- Return window: **14 days from delivery**.
- Reason: optional free-text field — customer may state why, not required to.
- Applies to delivered orders only (`orders.status = "delivered"`).

### 4.2 Full lifecycle

```
(order: delivered)
        │
        ▼  customer requests return, within 14 days
return_requested
        │
        ├──▶ admin rejects ──▶ return_rejected  [terminal — order remains "delivered"]
        │
        ▼  admin approves the request
return_approved
        │  customer ships the item back
        ▼
return_received
        │
        ├──▶ admin inspects, finds damage ──▶ return_rejected_on_inspection  [terminal — no refund]
        │
        ▼  admin inspects, confirms no damage
refund_initiated
        │  Paystack processes, webhook confirms
        ▼
refunded  [terminal]
```

- `return_received` is a distinct, trackable state — separate from "approved" — so admin can distinguish "approved, waiting for the item to physically arrive" from "arrived, needs inspection." This is operationally necessary: a customer asking "where's my refund?" needs an answerable status, not a guess.
- Rejection (at either the initial request stage or after inspection) **always includes a reason shown to the customer** — admin provides a short explanation, which is displayed back rather than a bare "rejected."
- Admin receives a notification when a return is requested, requiring action.

### 4.3 Partial returns

- The data model supports returning a subset of items from a multi-item order (not just the entire order), even though the initial admin UI may only support "return everything" at launch. This avoids a schema migration later if partial returns become an operational need.

---

## 5. Payments

Two supported payment paths:

### 5.1 Paystack (primary, automatic)

Customer pays via Paystack's hosted checkout inside the app. Payment confirmation arrives via webhook, which:
- Records the payment in the `payments` table
- Advances the order from `pending` → `paid` → `processing`

### 5.2 Manual / offline payment (business MoMo number)

Customer pays directly to the business's mobile money number outside the app (e.g. arranged via phone/WhatsApp). Admin manually marks the order as paid in the admin dashboard.

- This is a trust-boundary action: every manual "mark as paid" **must record which admin performed it and when** — same rigor as any other manual financial override, since it bypasses automatic verification.
- Once marked paid, the order proceeds through the same `paid → processing` flow as a Paystack payment.

---

## 6. Notifications

- **Admin** is notified when: a new order is placed, a return is requested, a delivery fails.
- Customer-facing notifications (order confirmation, status updates, return decision) are assumed necessary but the specific channel (email, SMS, WhatsApp, in-app) is **not yet decided** — flagged as an open question for the PRD.

---

## Open questions carried forward

- Confirmed dispatch provider (Shaq Express / Mckot / Traksend / other) — pending, affects the exact integration contract once selected.
- Customer notification channel(s) — pending.
- Whether `return_rejected_on_inspection` items are returned to the customer at their own cost, or handled some other way — not yet decided.
