# Software Architecture

## 1. System Overview

```
┌──────────────┐     ┌──────────────┐
│  Web (Next.js)│     │Mobile (Expo)  │  v2 — not built yet
│  storefront +  │     │               │
│  admin dashboard│    │               │
└───────┬────────┘     └───────┬───────┘
        │                      │
        └──────────┬───────────┘
                    │  HTTPS (JSON)
                    ▼
        ┌───────────────────────┐
        │  Next.js API Routes     │  apps/web/app/api/v1/*
        │  (single shared backend)│  (+ /api/auth, /api/webhooks, /api/cron)
        └────────────┬────────────┘
                    │
        ┌───────────┼────────────┬─────────────┐
        ▼           ▼            ▼             ▼
   ┌────────┐  ┌─────────┐  ┌──────────┐  ┌──────────┐
   │  Neon   │  │ Upstash │  │ Paystack │  │ Dispatch │
   │ Postgres│  │  Redis  │  │   API    │  │ Courier  │
   │ (Drizzle)│  │(rate    │  │(payments)│  │ API (v2, │
   │         │  │ limit)  │  │          │  │ TBD)     │
   └────────┘  └─────────┘  └──────────┘  └──────────┘
```

Web and (eventually) mobile are separate frontends consuming **one shared backend API** and **one shared database** — no business logic duplicated per client. This is why cart, checkout, and order logic live entirely in API routes rather than Next.js Server Actions, which only the web app could call.

Application routes are versioned under `/api/v1`. Within a version, changes are additive only; a breaking change gets a new version path. This matters once the mobile app ships: the web app is redeployed in step with the API, but old mobile app versions stay installed on phones for months (see API Specification — Conventions).

Not shown above: **Vercel Blob** stores product images (the admin uploads straight to it), **Vercel Cron** runs scheduled jobs (§5.1), and an **email provider** sends password-reset emails.

## 2. Rendering Strategy

### 2.1 Storefront — Server-Side Rendering (SSR) with Incremental Static Regeneration (ISR)

The public storefront (product listing, product detail pages) is server-rendered, prioritizing SEO — customers should be able to find products via search engines, not just direct/social links.

- **Product listing & detail pages**: rendered via Next.js ISR. Pages are cached and served statically after first render, then **revalidated on a timer** (e.g. every 60 seconds) so stock/price changes propagate without needing a full redeploy. On-demand revalidation (triggered by the admin product/inventory update endpoints calling `revalidatePath`) is used for cases where a change needs to appear immediately — e.g. a product going out of stock shouldn't wait up to 60 seconds to disappear from view if that matters to the business; this can be tuned once real usage patterns are known.
- **Cart, checkout, order history, account pages**: rendered client-side / dynamically — these are inherently per-user and not cacheable in the way product pages are.
- **Admin dashboard**: rendered dynamically, no caching — admin needs to see live data (new orders, current stock) without delay.

### 2.2 Why not a separate caching layer (e.g. Redis) for product data

Upstash Redis is already in the stack for rate limiting, but a dedicated Redis cache for product data is deliberately **not** part of v1 — Next.js's built-in ISR solves the "catalog doesn't change often, is read constantly" problem without adding new infrastructure or a second source of truth to keep in sync. This can be revisited if traffic scale later makes ISR's revalidation granularity insufficient.

## 3. Frontend State Management

Two distinct categories of state, deliberately handled by different tools:

### 3.1 Server state — TanStack Query
Data that originates from the database via the API: products, cart contents (once synced), order history, admin dashboard data. TanStack Query owns:
- Caching fetched data and avoiding redundant requests
- Background refetching / staleness handling
- Loading and error states, consistently, without hand-rolled logic per component
- Invalidating and refetching related data after a mutation (e.g. placing an order invalidates the cart query and the order-history query)

### 3.2 Client state — Zustand
UI-only state that doesn't live in the database, or data that needs to be read from many unrelated components without prop-drilling. Concretely:
- **Guest cart** (see §4) — before a guest's cart is synced to the server, its contents live in a Zustand store, read by the header cart icon, the cart drawer, and the checkout page alike.
- Transient UI state: modal open/closed, mobile nav state, selected product image in a gallery, etc.

No Redux — unnecessary complexity at this scale. No Context API for the above, since Zustand covers the "shared state without prop-drilling" need more simply than Context + reducer boilerplate would.

## 4. Cart Persistence & Guest Checkout

Cart access does **not** require an account. A visitor can browse and add items to a cart before signing up — account creation is only required at checkout, when payment and delivery details are actually needed.

### 4.1 Mechanism

- A guest is identified by a random UUID (not tied to any `user` row). On web it's a `guest_id` cookie set by the server (`HttpOnly`, 30-day expiry) the first time a guest uses the cart; on mobile it's generated on the device and sent as an `X-Guest-Id` header.
- The guest's cart lives as a `carts` row with `userId: null` and a new `guestId` column, rather than being purely client-side — this means a guest's cart survives a page refresh or closing/reopening the browser (as long as the cookie persists), not just a single session in memory.
- **Zustand mirrors this cart client-side** for instant UI feedback (adding an item updates the cart icon immediately, without waiting on a round-trip), while the actual source of truth remains the `carts`/`cart_items` rows on the server — Zustand is a read-through cache/optimistic layer, not a second source of truth that could drift from the database.

### 4.2 Merge on sign-in

When a guest with an existing cart creates an account or signs in (most likely to happen at the checkout step), the client calls `POST /api/v1/cart/merge`, which merges the guest cart into their real, `userId`-linked cart in one transaction:
- If the account has no existing cart, the guest cart's `userId` is simply set (claimed).
- If the account already has items in its own cart (e.g. they'd shopped from another device while signed in previously), line items are merged using the same upsert logic as adding to the cart (matching `variantId`, summing quantities, capped at current stock) rather than overwriting, and the guest cart is deleted.
- The `guest_id` cookie is cleared. Calling merge twice is harmless.

Guest carts untouched for 30 days are deleted by a daily cron job.

### 4.3 Schema impact

`carts.userId` becomes **nullable**, and a new `carts.guestId` (text, nullable, unique) column is added. Exactly one of `userId`/`guestId` is set per cart row — enforced by a database `CHECK` as well as in application code, since the merge logic is new and untested. Already reflected in the Database Schema document.

## 5. Data Flow — Checkout (the critical path)

```
1. Guest/customer browses (SSR/ISR pages) → adds to cart
     → POST /api/v1/cart/items (guest: guest_id cookie; customer: session)
2. Proceeds to checkout → prompted to sign in/sign up if guest
     → POST /api/v1/cart/merge on auth success
3. Selects/adds delivery address, picks payment method
     → POST /api/v1/orders { addressId, paymentMethod }
       with Idempotency-Key header (one key per checkout attempt)
     → single DB transaction: validate address ownership → lock and
       check stock per item → create order (copying prices and the
       shipping address, setting the payment deadline) + order_items
       → decrement inventory → clear cart
4. POST /api/v1/orders/{id}/pay → Paystack checkout session created
5. Customer pays on Paystack's hosted page
6. Payment confirmed by whichever arrives first (same shared function):
     a. Paystack → POST /api/webhooks/paystack (signature-verified,
        duplicate events skipped)
     b. Customer lands on /checkout/complete
        → POST /api/v1/orders/{id}/verify-payment (asks Paystack)
     → amount + currency checked against the order
     → order status: pending → paid → processing
   If never paid: the expiry cron cancels the order after its deadline
   and restores stock.
7. Admin dashboard reflects the new order (dynamic render, no caching)
     → admin requests dispatch: POST /api/v1/admin/orders/{id}/delivery
       (manual in v1 / courier API in v2) → order status → shipped
     → PATCH /api/v1/admin/orders/{id}/delivery as status changes
     → delivery reaches "delivered" → order status → delivered
```

Step 3's transaction boundary is the single most safety-critical piece of the whole system — it's the only place stock, money-intent (order creation), and cart state all change together, and it must remain atomic as the system grows (e.g. adding delivery-fee calculation here later must not weaken this guarantee). The idempotency key makes a double-click or network retry return the same order instead of creating a second one.

Step 6's shared confirmation function is the second most critical piece: it is the only code that turns an order into "paid", and both entry points must go through it.

### 5.1 Scheduled jobs (Vercel Cron)

| Job | Schedule | Does |
|---|---|---|
| `POST /api/cron/expire-orders` | every 5 minutes | Cancels `pending` orders past their payment deadline and restores stock (checks Paystack first in case the payment actually went through) |
| `POST /api/cron/cleanup-guest-carts` | daily | Deletes guest carts not updated in 30 days |

Cron routes only accept requests carrying `CRON_SECRET`. The expiry check also runs whenever a single order is read or written, so correctness doesn't depend on the cron running on time — the cron only makes sure stock is freed even if nobody looks at the order.

## 6. Admin Dashboard Architecture

- Rendered dynamically (no ISR/caching) — admin needs real-time-enough visibility into orders, stock, and returns.
- Same Next.js app as the storefront (`apps/web`), gated by `requireAdmin()` at the route level — not a separate deployed application. This keeps deployment simple for v1 (one app, one Vercel project) at the cost of shipping admin-only JS in the same bundle boundary as the storefront (mitigated by Next.js's route-based code splitting — a customer browsing products doesn't download admin dashboard code).

## 7. Mobile (v2) Architecture Notes

Not built in v1, but the backend is intentionally API-first specifically so mobile can be added without backend changes:
- Expo app will consume the same REST API (`/api/v1`) as the web storefront.
- Auth uses better-auth's `bearer` plugin: the app sends `Authorization: Bearer <token>` instead of a cookie, with the token kept in secure storage on the device. The API accepts either, so no route changes are needed.
- Payment: the app opens Paystack's checkout page in an in-app browser and calls the same verify-payment endpoint when the customer returns.
- Server state on mobile: TanStack Query again (same library works in React Native).
- Client/guest-cart state: same conceptual approach (Zustand works in React Native too). The guest ID is stored on the device (e.g. `AsyncStorage`) and sent as the `X-Guest-Id` header instead of a cookie.
- SSR/ISR is a web-only concept — mobile will fetch product data directly from the API on each screen load, relying on TanStack Query's client-side caching instead.

## 8. Deployment

| Concern | Service |
|---|---|
| Web app hosting | Vercel |
| Database | Neon (serverless Postgres) |
| Rate-limit storage (custom routes) | Upstash Redis |
| Rate-limit storage (auth routes) | Neon (`rate_limit` table, via better-auth) |
| Payments | Paystack |
| Product images | Vercel Blob |
| Scheduled jobs | Vercel Cron |
| Password-reset email | Email provider (e.g. Resend) — TBD |
| Delivery dispatch | Manual (v1) → third-party courier API (v2, provider TBD) |

Vercel auto-deploys on push to `main`. Environment variables (`DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, Upstash credentials, Paystack secret key, `CRON_SECRET`, `BLOB_READ_WRITE_TOKEN`, email provider API key) must be set per-environment (Production, Preview) in Vercel's project settings — these are not inherited from local `.env` files.

The Paystack webhook URL (`/api/webhooks/paystack`) must be registered in the Paystack dashboard for the live and test keys separately.

## 9. Known Architectural Risks

- **Neon cold-start latency**: Neon's free tier pauses the database when idle, so the first query after a quiet period can take several seconds or time out. Serverless Postgres trades this for cost efficiency at low traffic. Not addressed by a specific mitigation in v1; worth monitoring once real traffic patterns emerge, and revisiting (e.g. a connection-pooling proxy, or a paid Neon tier with less aggressive scale-to-zero) if it causes user-facing failures.
- **Single Next.js app serving both storefront and admin**: simplest for v1, but means a bug or outage in one technically shares deployment risk with the other. Acceptable tradeoff for a 3-person team's first release; a future split into separate apps is possible without a full rewrite, since the backend is already decoupled via the API.
- **Guest cart merge logic** is new, not-yet-built complexity (per §4) and deserves dedicated testing attention given it's a first-touch experience for every non-returning customer.
- **Payment confirmation races**: the webhook, the verify-payment call and the expiry cron can all act on the same order at nearly the same time. Correctness relies on each doing its work in a transaction that re-checks the order's current status, plus the database rules (one successful payment per order, unique webhook event IDs). This needs automated tests that run them concurrently, not just one at a time.