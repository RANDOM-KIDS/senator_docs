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
        │  Next.js API Routes     │  apps/web/app/api/*
        │  (single shared backend)│
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

- A guest is identified by a random ID generated client-side and stored in a browser cookie (not tied to any `user` row).
- The guest's cart lives as a `carts` row with `userId: null` and a new `guestId` column, rather than being purely client-side — this means a guest's cart survives a page refresh or closing/reopening the browser (as long as the cookie persists), not just a single session in memory.
- **Zustand mirrors this cart client-side** for instant UI feedback (adding an item updates the cart icon immediately, without waiting on a round-trip), while the actual source of truth remains the `carts`/`cart_items` rows on the server — Zustand is a read-through cache/optimistic layer, not a second source of truth that could drift from the database.

### 4.2 Merge on sign-in

When a guest with an existing cart creates an account or signs in (most likely to happen at the checkout step), their guest cart is merged into their real, `userId`-linked cart:
- If the account has no existing cart, the guest cart's `userId` is simply set (claimed).
- If the account already has items in its own cart (e.g. they'd shopped from another device while signed in previously), line items are merged using the same upsert logic already built for `POST /api/cart` (matching `variantId`, summing quantities) rather than overwriting.

### 4.3 Schema impact

This requires a small schema change beyond what's in the current Database Schema document: `carts.userId` becomes **nullable**, and a new `carts.guestId` (text, nullable, unique when set) column is added. Exactly one of `userId`/`guestId` should be set per cart row — enforced at the application layer.

## 5. Data Flow — Checkout (the critical path)

```
1. Guest/customer browses (SSR/ISR pages) → adds to cart
     → POST /api/cart (guest: guestId cookie; customer: session)
2. Proceeds to checkout → prompted to sign in/sign up if guest
     → guest cart merged into account cart on auth success
3. Selects/adds delivery address
     → POST /api/orders { addressId }
     → single DB transaction: validate address ownership → check stock
       per item → create order + order_items → decrement inventory
       → clear cart
4. POST /api/orders/{id}/pay → Paystack checkout session created
5. Customer pays on Paystack's hosted page
6. Paystack → POST /api/webhooks/paystack (signature-verified)
     → order status: pending → paid → processing
7. Admin dashboard reflects the new order (dynamic render, no caching)
     → admin requests dispatch manually (v1) / via courier API (v2)
     → PATCH /api/admin/orders/{id}/delivery as status changes
     → delivery reaches "delivered" → order status → delivered
```

Step 3's transaction boundary is the single most safety-critical piece of the whole system — it's the only place stock, money-intent (order creation), and cart state all change together, and it must remain atomic as the system grows (e.g. adding delivery-fee calculation here later must not weaken this guarantee).

## 6. Admin Dashboard Architecture

- Rendered dynamically (no ISR/caching) — admin needs real-time-enough visibility into orders, stock, and returns.
- Same Next.js app as the storefront (`apps/web`), gated by `requireAdmin()` at the route level — not a separate deployed application. This keeps deployment simple for v1 (one app, one Vercel project) at the cost of shipping admin-only JS in the same bundle boundary as the storefront (mitigated by Next.js's route-based code splitting — a customer browsing products doesn't download admin dashboard code).

## 7. Mobile (v2) Architecture Notes

Not built in v1, but the backend is intentionally API-first specifically so mobile can be added without backend changes:
- Expo app will consume the same REST API as the web storefront.
- Server state on mobile: TanStack Query again (same library works in React Native).
- Client/guest-cart state: same conceptual approach (Zustand works in React Native too), though the guest-identification mechanism will need to use a mobile-appropriate storage mechanism (e.g. `AsyncStorage`-backed ID) instead of a browser cookie.
- SSR/ISR is a web-only concept — mobile will fetch product data directly from the API on each screen load, relying on TanStack Query's client-side caching instead.

## 8. Deployment

| Concern | Service |
|---|---|
| Web app hosting | Vercel |
| Database | Neon (serverless Postgres) |
| Rate-limit storage (custom routes) | Upstash Redis |
| Rate-limit storage (auth routes) | Neon (`rate_limit` table, via better-auth) |
| Payments | Paystack |
| Delivery dispatch | Manual (v1) → third-party courier API (v2, provider TBD) |

Vercel auto-deploys on push to `main`. Environment variables (`DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, Upstash credentials, Paystack secret key) must be set per-environment (Production, Preview) in Vercel's project settings — these are not inherited from local `.env` files.

## 9. Known Architectural Risks

- **Neon cold-start latency**: observed directly during development (intermittent multi-second delays/timeouts on the first query after inactivity). Serverless Postgres trades this for cost efficiency at low traffic. Not addressed by a specific mitigation in v1; worth monitoring once real traffic patterns emerge, and revisiting (e.g. a connection-pooling proxy, or a paid Neon tier with less aggressive scale-to-zero) if it causes user-facing failures.
- **Single Next.js app serving both storefront and admin**: simplest for v1, but means a bug or outage in one technically shares deployment risk with the other. Acceptable tradeoff for a 3-person team's first release; a future split into separate apps is possible without a full rewrite, since the backend is already decoupled via the API.
- **Guest cart merge logic** is new, not-yet-built complexity (per §4) — this is the one part of the architecture with no existing tested code behind it yet, and deserves dedicated testing attention given it's a first-touch experience for every non-returning customer.
