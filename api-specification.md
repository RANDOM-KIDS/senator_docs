# API Specification

Base URL (local): `http://localhost:3000`
Base URL (production): `https://e-commerce-web-psi-sage.vercel.app`

All bodies are JSON. Authenticated routes rely on a session cookie set by better-auth after sign-in.

**Status legend:** ✅ Built and tested · 🔜 Planned (designed, not yet implemented)

---

## Auth

Handled by better-auth (`/api/auth/[...all]`).

### ✅ Sign up
`POST /api/auth/sign-up/email`
**Auth:** none
**Body:** `{ "email": string, "password": string, "name": string }`
**200:** `{ "user": {...}, "session": {...} }`
**Errors:** `400 VALIDATION_ERROR` · `422 USER_ALREADY_EXISTS_USE_ANOTHER_EMAIL` · `429` (3/hour per IP)

### ✅ Sign in
`POST /api/auth/sign-in/email`
**Auth:** none
**Body:** `{ "email": string, "password": string }`
**200:** `{ "user": {...}, "session": {...} }` — sets session cookie
**Errors:** `401` invalid credentials · `429` (5/min per IP)

### ✅ Get session
`GET /api/auth/get-session`
**Auth:** session cookie (optional — returns `null` if absent)
**200:** current user/session or `null`

---

## Products (public)

### ✅ List products
`GET /api/products?search=&category=&page=`
**Auth:** none
**200:**
```json
{ "products": [{ "product": {...}, "category": {...}, "variant": {...}, "stock": 48 }], "page": 1, "limit": 20 }
```
Only `isActive: true` products returned.

### 🔜 Get single product
`GET /api/products/{productId}`
**Auth:** none
**200:** full product with all variants + stock
**404:** not found or inactive

---

## Categories (public read, admin write)

### 🔜 List categories
`GET /api/categories`
**Auth:** none
**200:** `{ "categories": [{ "id", "name", "slug" }] }`

### 🔜 Create category
`POST /api/admin/categories`
**Auth:** admin
**Body:** `{ "name": string, "slug": string }`
**201:** created category
**Errors:** `400` validation · `409` slug already exists

### 🔜 Update category
`PATCH /api/admin/categories/{categoryId}`
**Auth:** admin
**Body:** `{ "name"?: string, "slug"?: string }`
**200:** updated category
**404:** not found

### 🔜 Delete category
`DELETE /api/admin/categories/{categoryId}`
**Auth:** admin
**204:** no content
**409:** blocked if products still reference this category (must reassign/delete products first, or reassign to null)

---

## Addresses (auth required)

### 🔜 List my addresses
`GET /api/addresses`
**Auth:** customer
**200:** `{ "addresses": [...] }` — only the requesting user's own

### 🔜 Add address
`POST /api/addresses`
**Auth:** customer
**Body:** `{ "region": string, "city": string, "landmarkOrGps"?: string, "recipientPhone": string, "isDefault"?: boolean }`
**201:** created address

### 🔜 Update address
`PATCH /api/addresses/{addressId}`
**Auth:** customer, must own the address
**Body:** any subset of the fields above
**200:** updated address
**403:** not the owner
**404:** not found

### 🔜 Delete address
`DELETE /api/addresses/{addressId}`
**Auth:** customer, must own the address
**204:** no content
**409:** blocked if referenced by an existing order (addresses are historical record on past orders — don't hard-delete out from under them; consider a soft-delete/`isArchived` flag instead if this comes up)

---

## Cart (auth required)

### ✅ Get cart
`GET /api/cart`
**Auth:** customer
**200:** cart with nested items + variant + product, or `{ "items": [] }`

### ✅ Add / upsert item
`POST /api/cart`
**Auth:** customer
**Body:** `{ "variantId": string, "quantity": number }`
**200:** `{ "message": "Item added to cart" }`
Behavior: increments existing line quantity rather than duplicating.

### 🔜 Update item quantity (set, not add)
`PATCH /api/cart/{cartItemId}`
**Auth:** customer, must own the cart
**Body:** `{ "quantity": number }`
**200:** updated item
**404:** not found / not owned

### 🔜 Remove item
`DELETE /api/cart/{cartItemId}`
**Auth:** customer, must own the cart
**204:** no content

### 🔜 Clear cart
`DELETE /api/cart`
**Auth:** customer
**204:** no content

---

## Orders (auth required)

### ✅ Create order (checkout)
`POST /api/orders`
**Auth:** customer
**Body:** `{ "addressId": string }`
**200:** `{ "orderId": string }`
**Errors:** `400` missing/invalid body · `400` cart empty · `400` insufficient stock (per-item message) · `429` (10/min per user)
Transactional: stock check → order + order_items → inventory decrement → cart clear, all-or-nothing.

### ✅ Order history
`GET /api/orders`
**Auth:** customer
**200:** `{ "orders": [{ ...order, items, address, payments }] }`

### 🔜 Get single order
`GET /api/orders/{orderId}`
**Auth:** customer, must own the order (or admin)
**200:** full order detail including delivery + return status if present
**404:** not found / not owned

### ✅ Initialize payment
`POST /api/orders/{orderId}/pay`
**Auth:** customer, must own the order
**200:** `{ "authorizationUrl": string, "reference": string }`
**Errors:** `404` · `400` order not `pending` · `503` Paystack unreachable

### 🔜 Cancel order (self-service)
`POST /api/orders/{orderId}/cancel`
**Auth:** customer, must own the order
**Allowed when:** status is `pending`, `paid`, or `processing`
**200:** updated order, `status: "cancelled"`; inventory restored; if previously paid, a `refunds` row is created with `status: "initiated"`
**Errors:** `409` — order already `shipped`/`delivered`/`cancelled`, cannot self-cancel

### 🔜 Request a return
`POST /api/orders/{orderId}/return`
**Auth:** customer, must own the order
**Body:** `{ "reason"?: string, "items": [{ "orderItemId": string, "quantity": number }] }`
**Allowed when:** order status is `delivered`, within 14 days of `deliveredAt`, and no existing `returns` row for this order (one-shot)
**201:** created `returns` row, `status: "return_requested"`
**Errors:** `400` outside 14-day window · `409` a return already exists for this order · `404` order not found/not owned

---

## Payments Webhook

### 🔜 Paystack webhook
`POST /api/webhooks/paystack`
**Auth:** none (public endpoint) — **must verify `x-paystack-signature` header** (HMAC of the raw body using the Paystack secret key) before trusting any payload
**Body:** Paystack's event payload (`event: "charge.success"`, `event: "refund.processed"`, etc.)
**200:** acknowledged (always return 200 quickly, even if internal processing is deferred, per Paystack's retry behavior expectations)
**Behavior:**
- `charge.success` → find order via `metadata.orderId`, create `payments` row (`provider: "paystack"`, `status: "success"`), advance order `pending → paid → processing`
- `refund.processed` → find the matching `refunds` row via `paystackRefundReference`, set `status: "completed"`, `completedAt`; if return-originated, advance `returns.status → "refunded"`
- Must be **idempotent** — the same event arriving twice (Paystack's documented retry behavior) must not double-process (e.g. check if the order is already `paid` before acting again)

---

## Admin — Products

### ✅ Create product
`POST /api/admin/products`
**Auth:** admin
**Body:** `{ "categoryId"?, "name", "description"?, "basePrice", "variants": [{ "size", "color", "priceOverride"?, "imageUrl"?, "initialStock" }] }`
**201:** created product
Transactional: product + variants + inventory rows, all-or-nothing.

### 🔜 Update product
`PATCH /api/admin/products/{productId}`
**Auth:** admin
**Body:** any subset of `name`, `description`, `basePrice`, `categoryId`, `isActive`
**200:** updated product
**404:** not found

### 🔜 Deactivate product (soft delete)
`DELETE /api/admin/products/{productId}`
**Auth:** admin
Sets `isActive: false` rather than a hard delete — preserves referential integrity with existing `order_items` history.
**204:** no content

### 🔜 Add variant to existing product
`POST /api/admin/products/{productId}/variants`
**Auth:** admin
**Body:** `{ "size", "color", "priceOverride"?, "imageUrl"?, "initialStock" }`
**201:** created variant + inventory row

---

## Admin — Inventory

### ✅ Update inventory
`PATCH /api/admin/inventory/{variantId}`
**Auth:** admin
**Body:** `{ "quantity": number, "lowStockThreshold"?: number }`
**200:** updated inventory row
**404:** variant/inventory not found

---

## Admin — Orders

### ✅ Update order status
`PATCH /api/admin/orders/{orderId}/status`
**Auth:** admin
**Body:** `{ "status": "pending" | "paid" | "processing" | "shipped" | "delivered" | "cancelled" }`
**200:** updated order
**404:** not found · **400:** invalid status
Note: no transition restrictions currently enforced.

### 🔜 Mark order paid manually
`PATCH /api/admin/orders/{orderId}/mark-paid`
**Auth:** admin
**Body:** `{ "note"?: string }` — optional MoMo reference for record-keeping
**200:** updated order (`status: paid → processing`); creates `payments` row with `provider: "manual"`, `markedByAdminId`, `markedAt`
**409:** order not `pending`

### 🔜 Admin cancel order
`PATCH /api/admin/orders/{orderId}/cancel`
**Auth:** admin
**Body:** `{ "reason"?: string }`
**200:** updated order, `status: "cancelled"`; inventory restored; refund initiated if previously paid
Allowed at any status, including `shipped` (exception path).

### 🔜 Update delivery status
`PATCH /api/admin/orders/{orderId}/delivery`
**Auth:** admin
**Body:** `{ "status": "requested" | "assigned" | "picked_up" | "en_route" | "delivered" | "failed", "provider"?: string, "providerReference"?: string, "failureReason"?: string }`
**200:** updated `deliveries` row; `status: "delivered"` also advances the parent order to `delivered`
Manual/admin-driven in v1 — no live courier API to consume yet.

---

## Admin — Returns

### 🔜 List return requests
`GET /api/admin/returns?status=`
**Auth:** admin
**200:** `{ "returns": [{ ...return, order, items, customer }] }`

### 🔜 Approve or reject a return request
`PATCH /api/admin/returns/{returnId}/decision`
**Auth:** admin
**Body:** `{ "decision": "approve" | "reject", "rejectionReason"?: string }` — required if `decision: "reject"`
**200:** updated return (`return_approved` or `return_rejected`)
**409:** return not in `return_requested` state

### 🔜 Mark item received
`PATCH /api/admin/returns/{returnId}/receive`
**Auth:** admin
**200:** updated return, `status: "return_received"`
**409:** not in `return_approved` state

### 🔜 Confirm inspection
`PATCH /api/admin/returns/{returnId}/inspect`
**Auth:** admin
**Body:** `{ "result": "pass" | "fail", "rejectionReason"?: string }` — required if `result: "fail"`
**200:** updated return — `pass` → triggers refund initiation (creates `refunds` row, calls Paystack refund API); `fail` → `status: "return_rejected_on_inspection"`
**409:** not in `return_received` state

---

## Admin — Refunds

### 🔜 List refunds
`GET /api/admin/refunds?status=`
**Auth:** admin
**200:** `{ "refunds": [...] }`

### 🔜 Get refund detail
`GET /api/admin/refunds/{refundId}`
**Auth:** admin
**200:** refund with linked order/return/payment

Refunds are otherwise created automatically as a side effect of cancellation or inspection-pass — no standalone "create refund" endpoint is needed for the designed flow.

---

## Admin — Notifications

### 🔜 List notifications
`GET /api/admin/notifications?unreadOnly=`
**Auth:** admin
**200:** `{ "notifications": [...] }`

### 🔜 Mark notification read
`PATCH /api/admin/notifications/{notificationId}/read`
**Auth:** admin
**200:** updated notification

---

## Admin — Dashboard

### ✅ Sales aggregates
`GET /api/admin/dashboard`
**Auth:** admin
**200:** `{ "totalOrders": number, "totalRevenue": string, "ordersByStatus": {...} }`
Revenue excludes cancelled orders.

---

## Rate Limiting Summary

| Scope | Limit | Storage |
|---|---|---|
| Sign-in | 5/min per IP | Database (better-auth) |
| Sign-up | 3/hour per IP | Database (better-auth) |
| Orders (create) | 10/min per user | Upstash Redis |
| Admin writes | 30/min per admin | Upstash Redis |
| 🔜 Return/refund actions | *not yet decided* — likely fold into the existing admin-write limiter rather than a new one | — |

## Error Shape Conventions

- `400` — malformed request / business-rule validation
- `401` — no valid session
- `403` — valid session, wrong role, or not the resource owner
- `404` — not found / not owned
- `409` — valid request, but conflicts with current resource state (e.g. wrong order status for the action)
- `429` — rate limited
- `503` — upstream dependency unreachable (Paystack)

Most handlers return `{ "error": "..." }`; Zod validation failures return `{ "error": <flatten() output> }`.
