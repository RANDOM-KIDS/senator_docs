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
| `cancelled` | Order will not be fulfilled | Customer self-cancels (`pending`/`paid`/`processing` only), admin cancels (any status except `delivered`, exception path), or the system cancels an unpaid order that passed its payment deadline (§1.4) |

`paid → processing` happens automatically and immediately — there is no manual admin review step between payment confirmation and preparation.

Every status change is recorded with who made it (customer, admin, system or Paystack) and when, giving each order a full history.

### 1.2 Cancellation rules

- **Customer self-cancellation**: allowed while status is `pending`, `paid`, or `processing`. Not available once `shipped` — a courier has already been requested, typically at a cost to the business.
- **Admin cancellation**: allowed at any status except `delivered`, including `shipped`, as an exception path (e.g. courier request failed, customer called in). A reason is required. Cancelling a `shipped` order requires admin judgment about whether/how to recall the courier — this is a manual coordination problem, not something the system automates. A `delivered` order can't be cancelled; it goes through returns (§4).
- **Inventory on cancellation**: always restored. Cancelling an order returns the reserved quantity back to `inventory.quantity` for each affected variant, regardless of who cancelled it or at what status.
- Cancelling a paid order (`paid`, `processing` or `shipped`) creates a pending refund (§3) — money already collected must be returned, but an admin starts the refund.
- **Moving back from `shipped`**: if a courier request falls through before pickup, an admin can move the order back to `processing` (its delivery record is cancelled) and request dispatch again. This is the only backwards status move.

### 1.3 Inventory decrement timing

Stock decrements at **order creation** (checkout), not at cart-add. This is a first-come-first-served model: if two customers have the same last unit in their carts, whichever one completes checkout first succeeds; the other receives an "insufficient stock" error at their own checkout attempt. No cart-level reservation/hold exists.

### 1.4 Payment deadline for unpaid orders

Because stock is taken at checkout, an order that is never paid would hold its stock forever. So every order gets a payment deadline when it's created:

- **Paystack**: 30 minutes.
- **Manual MoMo**: 24 hours (the customer has to send money outside the app and an admin has to confirm it).

Both values are configuration, not hard-coded. An order still `pending` after its deadline is cancelled automatically (reason: `expired`) and its stock is restored.

**Late payment** — the customer pays after the deadline (e.g. completes the Paystack page slowly):
- If every item is still in stock, the stock is taken again and the order continues as `processing`, as if it had been paid on time.
- If not, the order stays `cancelled`, a pending refund is created and admins are notified to refund the customer.

The same applies when an admin confirms a manual MoMo payment for an expired order.

---

## 2. Delivery / Dispatch

### 2.1 Model

Delivery is fulfilled through a **third-party dispatch courier API** (candidate providers: Shaq Express, Mckot, Traksend — none integrated yet; design is provider-agnostic so any can be plugged in). No live provider API access exists at time of writing — the system is designed against a generic contract, not a specific provider's real API shape.

### 2.2 Delivery sub-status (tracked independently of order status)

Order-level status only reaches as far as `shipped`. The granular delivery progress lives in its own record, separate from the order:

```
requested ──▶ assigned ──▶ picked_up ──▶ en_route ──▶ delivered
    ▲                                       
    └───────── failed ◀── (any step before delivered)

(any step) ──▶ cancelled   — order cancelled, or moved back to processing
```

- Requesting dispatch creates the delivery record (`requested`) and moves the order from `processing` to `shipped` at the same moment.
- Status only moves forward. Steps may be skipped, since a provider may not report every one.
- Displayed to the customer as a simple status label (e.g. "Courier assigned," "Picked up," "On the way," "Delivered"). No live map/GPS tracking in this version — the schema reserves optional fields for a future provider that supports live location, but nothing populates them yet.
- `delivered` at the delivery-record level triggers the order's own status to advance to `delivered`, and starts the 14-day return window.
- `failed` (delivery attempt failed — customer unreachable, address issue, etc.) needs a reason, notifies admins and requires admin intervention. It does not automatically cancel or refund the order. The admin can retry delivery (`failed → requested`) or cancel the order.

### 2.3 Handoff communication

"Communication between the two parties" (business ↔ dispatch provider) happens via whatever integration mechanism the eventual provider supports — likely a request/response API call to request a courier, plus either a webhook or polling for status updates. The exact mechanism is provider-specific and will be finalized once a provider is selected; the system's job is to record the resulting status changes regardless of how they arrive.

---

## 3. Refunds

### 3.1 When a refund applies

- Order cancelled after payment (`paid`, `processing`, or admin-cancelled `shipped`) — by the customer or an admin
- Return approved and inspected successfully (§4)
- Customer paid after their order's deadline and the stock was gone (§1.4)

### 3.2 Refund mechanism

Refunds are **always started by an admin**, never automatically. The events above only *create* a refund in `pending` status and notify admins; no money moves until an admin starts it.

```
pending ──▶ initiated ──▶ completed
               │
               └──▶ failed ──▶ initiated (admin retries)

pending ──▶ completed   (manual MoMo refund, recorded by admin)
```

- **Paystack payments**: the admin starts the refund, which calls Paystack's refund API. Paystack sends a webhook when the refund completes, or when it fails. A failed refund notifies admins and can be retried.
- **Manual MoMo payments**: Paystack can't refund money it never collected. The admin sends the money back to the customer's MoMo number outside the system, then records it with the MoMo transaction reference. This completes the refund directly.
- Every started refund records which admin started it and when.
- Refund amount **excludes the original delivery fee** — only the product cost is refunded. For a return, that's the price of the returned items only.
- The order/return is not considered fully closed until the refund is `completed`.

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
        ▼  admin inspects, confirms no damage (items restocked by default)
refund_pending
        │  admin starts the refund (§3.2)
        ▼
refund_initiated
        │  Paystack processes, webhook confirms
        │  (manual MoMo refunds skip straight to refunded once recorded)
        ▼
refunded  [terminal]
```

- `refund_pending` exists so the customer and admins can tell "inspection passed, waiting for an admin to send the money" apart from "money sent, waiting for Paystack to confirm."
- When an inspection passes, the returned items go back into stock by default. The admin can opt out (e.g. item is resellable only after cleaning, or is being written off).

- `return_received` is a distinct, trackable state — separate from "approved" — so admin can distinguish "approved, waiting for the item to physically arrive" from "arrived, needs inspection." This is operationally necessary: a customer asking "where's my refund?" needs an answerable status, not a guess.
- Rejection (at either the initial request stage or after inspection) **always includes a reason shown to the customer** — admin provides a short explanation, which is displayed back rather than a bare "rejected."
- Admin receives a notification when a return is requested, requiring action.

### 4.3 Partial returns

- The data model supports returning a subset of items from a multi-item order (not just the entire order), even though the initial admin UI may only support "return everything" at launch. This avoids a schema migration later if partial returns become an operational need.

---

## 5. Payments

Two supported payment paths:

### 5.1 Paystack (primary, automatic)

Customer pays via Paystack's hosted checkout inside the app. Payment is confirmed by whichever arrives first: Paystack's webhook, or the app checking with Paystack when the customer lands back on the store after paying. Both run the same confirmation, which:
- Checks that the amount and currency (GHS) Paystack received match the order total. On a mismatch the order is **not** advanced and admins are notified.
- Records the payment in the `payments` table
- Advances the order from `pending` → `paid` → `processing`

The second arrival is ignored, and Paystack resending the same webhook is ignored, so an order can never be paid twice.

If the customer abandons the Paystack page, they can retry payment until the order's deadline (§1.4).

### 5.2 Manual / offline payment (business MoMo number)

Customer pays directly to the business's mobile money number outside the app (e.g. arranged via phone/WhatsApp). Admin manually marks the order as paid in the admin dashboard.

- The customer chooses this payment method at checkout, which gives the order a 24-hour payment deadline (§1.4).
- This is a trust-boundary action: every manual "mark as paid" **must record which admin performed it and when** — same rigor as any other manual financial override, since it bypasses automatic verification.
- The admin must enter the amount received (which must equal the order total) and the MoMo transaction reference as evidence.
- Once marked paid, the order proceeds through the same `paid → processing` flow as a Paystack payment.
- Refunds of manual payments are also manual (§3.2).

---

## 6. Notifications

- **Admin** is notified when: a new order is placed, a return is requested, a delivery fails, a refund is waiting to be started, a Paystack refund fails, or a payment has a problem (amount mismatch, or a late payment that couldn't be honoured).
- **Password reset** emails are sent from day one, whatever channel is chosen below — without them a customer who forgets their password is locked out.
- Customer-facing notifications (order confirmation, status updates, return decision) are assumed necessary but the specific channel (email, SMS, WhatsApp, in-app) is **not yet decided** — flagged as an open question for the PRD.

---

## Open questions carried forward

- Confirmed dispatch provider (Shaq Express / Mckot / Traksend / other) — pending, affects the exact integration contract once selected.
- Customer notification channel(s) — pending.
- Whether `return_rejected_on_inspection` items are returned to the customer at their own cost, or handled some other way — not yet decided.
- Delivery pricing (flat, per region, or quoted by the provider) — not yet decided. Orders carry a delivery fee field, which is `0.00` until this is settled.