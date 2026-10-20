# API Specification

This document describes every endpoint of the backend API: its URL, who can call it, what to send, and what comes back.

Production base URL: https://e-commerce-web-psi-sage.vercel.app
Local base URL: http://localhost:3000

Every URL below is written in full against production. For local development, swap the base URL for `http://localhost:3000`.


## How to read each endpoint

Every endpoint is laid out the same way:

- Method and URL: what to call.
- Auth: who is allowed to call it (none, customer, guest, admin).
- Headers: any headers beyond the usual ones.
- Path parameters: the `{placeholders}` in the URL.
- Query parameters: the `?key=value` options.
- Request body: each field, its type, whether it's required, and what it means.
- Example request and success response: real JSON you can paste into Postman.
- Errors: the HTTP status, the error code, and when it happens.
- Notes: rules and behaviour worth knowing.


# General rules

## Versioning

All application endpoints live under `/api/v1`. Three groups don't, because their paths are fixed by something else:

- `/api/auth/*` is owned by the better-auth library.
- `/api/webhooks/*` URLs are registered in the Paystack dashboard.
- `/api/cron/*` are internal scheduled jobs.

Inside v1, changes are additive only: new endpoints, new optional request fields, new response fields. Clients must ignore response fields they don't recognise. Removing or renaming a field, changing its type, or making validation stricter needs a new version (`/api/v2/...`), and v1 stays alive until the mobile app no longer uses it. This matters once the mobile app ships, because old app versions stay installed on phones for months.

## Authentication

Login is handled by better-auth. There are two ways to send the login:

- Web: the session cookie set by sign-in. The browser sends it automatically.
- Mobile: the header `Authorization: Bearer <token>`. The token comes back in the `set-auth-token` response header on sign-in (better-auth bearer plugin) and is kept in secure storage on the phone.

Every logged-in endpoint accepts either.

Auth levels used in this document:

- None: anyone can call it.
- Customer: must be signed in.
- Customer or guest: signed in, or carrying a guest ID (see below).
- Admin: must be signed in with `role: "admin"`.

## Guest identity

Guests can use the cart without an account.

- Web: the server sets a `guest_id` cookie (random UUID, HttpOnly, SameSite=Lax, Secure, 30-day expiry) the first time a guest uses the cart.
- Mobile: the app generates a UUID once, stores it on the phone, and sends it as the header `X-Guest-Id: <uuid>`.

If a request has a valid login, the login wins and the guest ID is ignored. The one exception is the cart merge endpoint, which needs both.

## Money

- Amounts are always strings in Ghana cedis with two decimals, for example `"149.00"`. Never JSON numbers (floating-point rounding), never pesewas.
- Pesewas (cedis × 100, as a whole number) are used only when talking to Paystack. One shared helper does the conversion.
- Orders, payments and refunds include `"currency": "GHS"`.

## HTTP methods

- GET reads and never changes anything.
- POST on a collection creates something (`POST /api/v1/addresses`).
- PATCH on a resource edits plain fields (`PATCH /api/v1/addresses/{addressId}`).
- DELETE removes or hides something.
- POST on a sub-path performs an action or status change (`POST /api/v1/orders/{orderId}/cancel`). Status is never changed through a generic "set status" edit, because each change has its own rules and side effects (stock, refunds, notifications).

## Pagination

Every list endpoint takes:

- `page`: page number, starts at 1. Default 1.
- `limit`: items per page. Default 20, maximum 50.

And returns:

```json
{
  "items": [],
  "page": 1,
  "limit": 20,
  "total": 134,
  "totalPages": 7
}
```

## Errors

Every error response has this exact shape:

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Only 2 left of Linen Kaftan, Black / M",
    "details": {}
  }
}
```

- `code` never changes wording. Apps decide what to do based on `code`, never on `message`.
- `message` is readable and safe to show a user.
- `details` is optional. For `VALIDATION_ERROR` it lists each bad field and what's wrong with it. For `INSUFFICIENT_STOCK` it lists each short item.

Example validation error:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Some fields are invalid",
    "details": {
      "recipientPhone": ["Required"],
      "region": ["Must be one of the 16 regions of Ghana"]
    }
  }
}
```

HTTP statuses and the codes used with them:

- 400 Bad Request: the request is malformed or breaks a business rule. Codes: `VALIDATION_ERROR`, `CART_EMPTY`, `INSUFFICIENT_STOCK`, `PRODUCT_UNAVAILABLE`, `RETURN_WINDOW_CLOSED`, `INVALID_RETURN_QUANTITY`, `AMOUNT_MISMATCH`, `WRONG_PAYMENT_METHOD`, `MISSING_IDEMPOTENCY_KEY`.
- 401 Unauthorized: not logged in, or the login has expired. Code: `UNAUTHENTICATED`.
- 403 Forbidden: logged in but not allowed (for example, a customer calling an admin endpoint). Code: `FORBIDDEN`.
- 404 Not Found: doesn't exist, or belongs to someone else. The API doesn't say which. Code: `NOT_FOUND`.
- 409 Conflict: the request is valid but clashes with the current state (for example, cancelling an order that has already shipped). Codes: `INVALID_STATE_TRANSITION`, `ALREADY_EXISTS`, `IDEMPOTENCY_KEY_REUSED`, `CATEGORY_IN_USE`.
- 413 Payload Too Large: an upload is too big. Code: `FILE_TOO_LARGE`.
- 429 Too Many Requests: rate limited. Code: `RATE_LIMITED`. The `Retry-After` header says how many seconds to wait.
- 502 Bad Gateway: Paystack returned an error. Code: `PAYMENT_PROVIDER_ERROR`.
- 503 Service Unavailable: Paystack couldn't be reached. Code: `PAYMENT_PROVIDER_UNAVAILABLE`.

better-auth's own endpoints (`/api/auth/*`) return better-auth's error format. The web and mobile apps convert it to the format above in one place.

## Idempotency (no duplicate orders)

Creating an order requires an `Idempotency-Key` header: a UUID the app generates once per checkout attempt (when the checkout page opens, not per click).

- Same key from the same user again: the original order is returned. No second order, no second stock decrement.
- Same key with a different body: 409 `IDEMPOTENCY_KEY_REUSED`.

The key is stored on the order itself. Every other action endpoint is safe to repeat because it checks the current status first and returns 409 `INVALID_STATE_TRANSITION` if the action already happened.

## Rate limits

The rate-limit check is the first thing every endpoint does. A blocked request gets 429 with a `Retry-After` header.

- Sign in: 5 per minute per IP.
- Sign up: 3 per hour per IP.
- Request password reset: 3 per hour per IP, and 3 per hour per email.
- Reset password: 5 per hour per IP.
- Public reads (products, categories): 60 per minute per IP.
- Cart changes: 30 per minute per user or guest.
- Create order: 10 per minute per user.
- Other customer actions (addresses, profile, pay, verify, cancel, return): 20 per minute per user.
- Admin changes (including uploads, returns and refunds): 30 per minute per admin.
- Paystack webhook: no limit. The signature check protects it.
- Cron jobs: no limit. The `CRON_SECRET` protects them.

better-auth stores its limits in the database. Everything else uses Upstash Redis.


# Auth

These paths belong to better-auth. Their request and response shapes are better-auth's own.

## Sign up

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/auth/sign-up/email
- Auth: none

Request body:

- `email` (string, required): the customer's email address.
- `password` (string, required): at least 8 characters.
- `name` (string, required): full name. Stored as `fullName`.
- `phone` (string, optional): phone number, for example `"0241234567"`.

Example request:

```json
{
  "email": "ama.mensah@example.com",
  "password": "s3cure-Passw0rd",
  "name": "Ama Mensah",
  "phone": "0241234567"
}
```

Success response: 200 OK

```json
{
  "token": "Vn2x8...session-token...",
  "user": {
    "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
    "email": "ama.mensah@example.com",
    "name": "Ama Mensah",
    "phone": "0241234567",
    "role": "customer",
    "emailVerified": false,
    "createdAt": "2026-10-20T10:00:00.000Z",
    "updatedAt": "2026-10-20T10:00:00.000Z"
  }
}
```

Also sets the session cookie, and the `set-auth-token` header for mobile.

Errors:

- 400 `VALIDATION_ERROR`: a field is missing or invalid.
- 422 `USER_ALREADY_EXISTS`: an account with this email already exists.
- 429 `RATE_LIMITED`: more than 3 sign-ups per hour from this IP.

Notes:

- `role` can't be set from here. Every new account is a customer.
- If the visitor has a guest cart, the app calls Merge guest cart right after this succeeds.

## Sign in

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/auth/sign-in/email
- Auth: none

Request body:

- `email` (string, required)
- `password` (string, required)

Example request:

```json
{
  "email": "ama.mensah@example.com",
  "password": "s3cure-Passw0rd"
}
```

Success response: 200 OK

```json
{
  "token": "Vn2x8...session-token...",
  "user": {
    "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
    "email": "ama.mensah@example.com",
    "name": "Ama Mensah",
    "phone": "0241234567",
    "role": "customer",
    "emailVerified": false,
    "createdAt": "2026-10-20T10:00:00.000Z",
    "updatedAt": "2026-10-20T10:00:00.000Z"
  }
}
```

Also sets the session cookie, and the `set-auth-token` header for mobile.

Errors:

- 401 `INVALID_EMAIL_OR_PASSWORD`: wrong email or password. The message doesn't say which.
- 429 `RATE_LIMITED`: more than 5 attempts per minute from this IP.

Notes:

- If the visitor has a guest cart, the app calls Merge guest cart right after this succeeds.

## Get current session

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/auth/get-session
- Auth: optional

Success response: 200 OK (logged in)

```json
{
  "session": {
    "id": "b8a7c6d5-e4f3-4a2b-9c1d-0e9f8a7b6c5d",
    "userId": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
    "expiresAt": "2026-10-27T10:00:00.000Z"
  },
  "user": {
    "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
    "email": "ama.mensah@example.com",
    "name": "Ama Mensah",
    "phone": "0241234567",
    "role": "customer",
    "emailVerified": false
  }
}
```

Success response: 200 OK (not logged in)

```json
null
```

## Sign out

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/auth/sign-out
- Auth: customer or admin

Request body: none.

Success response: 200 OK

```json
{
  "success": true
}
```

Notes:

- Ends the session on the server and clears the cookie. Mobile also deletes its stored token.

## Request password reset

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/auth/request-password-reset
- Auth: none

Request body:

- `email` (string, required): the account's email.
- `redirectTo` (string, required): the page the email link opens, for example the store's reset-password page.

Example request:

```json
{
  "email": "ama.mensah@example.com",
  "redirectTo": "https://e-commerce-web-psi-sage.vercel.app/reset-password"
}
```

Success response: 200 OK

```json
{
  "status": true,
  "message": "If this email exists, a reset link has been sent."
}
```

Errors:

- 429 `RATE_LIMITED`: more than 3 requests per hour from this IP or for this email.

Notes:

- The response is the same whether or not the email has an account, so nobody can use this to find out who's registered.
- Needs an email provider (for example Resend) configured in better-auth. This is the one customer email that can't wait for the notification-channel decision, because without it a customer who forgets their password is locked out for good.

## Reset password

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/auth/reset-password
- Auth: none (the token from the email link proves who it is)

Request body:

- `token` (string, required): from the reset link.
- `newPassword` (string, required): at least 8 characters.

Example request:

```json
{
  "token": "f9e8d7c6b5a4...",
  "newPassword": "n3w-Passw0rd!"
}
```

Success response: 200 OK

```json
{
  "status": true
}
```

Errors:

- 400 `INVALID_TOKEN`: the token is wrong or has expired.
- 429 `RATE_LIMITED`: more than 5 attempts per hour from this IP.

Notes:

- Logs the user out everywhere. They sign in again with the new password.


# Profile

## Get my profile

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/me
- Auth: customer

Success response: 200 OK

```json
{
  "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
  "email": "ama.mensah@example.com",
  "fullName": "Ama Mensah",
  "phone": "0241234567",
  "role": "customer",
  "createdAt": "2026-10-20T10:00:00.000Z"
}
```

Errors:

- 401 `UNAUTHENTICATED`

## Update my profile

- Method: PATCH
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/me
- Auth: customer

Request body (send only the fields you want to change):

- `fullName` (string, optional)
- `phone` (string, optional)

Example request:

```json
{
  "phone": "0209876543"
}
```

Success response: 200 OK

```json
{
  "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
  "email": "ama.mensah@example.com",
  "fullName": "Ama Mensah",
  "phone": "0209876543",
  "role": "customer",
  "createdAt": "2026-10-20T10:00:00.000Z"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 401 `UNAUTHENTICATED`

Notes:

- `email` and `role` can't be changed here.


# Products (public)

## List products

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/products
- Auth: none

Query parameters (all optional):

- `search`: text to match in the product name, for example `kaftan`.
- `category`: a category slug, for example `kaftans`.
- `sort`: `newest` (default), `price_asc` or `price_desc`.
- `page`, `limit`: see Pagination.

Example request:

```
GET https://e-commerce-web-psi-sage.vercel.app/api/v1/products?category=kaftans&sort=price_asc&page=1&limit=20
```

Success response: 200 OK

```json
{
  "items": [
    {
      "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
      "slug": "linen-kaftan",
      "name": "Linen Kaftan",
      "category": {
        "id": "c1a2b3c4-d5e6-4f70-8a91-b2c3d4e5f6a7",
        "name": "Kaftans",
        "slug": "kaftans"
      },
      "basePrice": "350.00",
      "priceRange": {
        "min": "350.00",
        "max": "380.00"
      },
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
      "inStock": true,
      "colors": ["Black", "White"],
      "sizes": ["S", "M", "L"]
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 1,
  "totalPages": 1
}
```

Errors:

- 400 `VALIDATION_ERROR`: bad `sort`, `page` or `limit`.
- 429 `RATE_LIMITED`

Notes:

- One item per product, not per variant. A product grid shows products.
- Only active products with at least one active variant are listed.
- `imageUrl` is the first variant's image. `inStock` is true if any variant has stock.

## Get a product

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/products/{slug}
- Auth: none

Path parameters:

- `slug`: the product's URL name, for example `linen-kaftan`.

Example request:

```
GET https://e-commerce-web-psi-sage.vercel.app/api/v1/products/linen-kaftan
```

Success response: 200 OK

```json
{
  "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
  "slug": "linen-kaftan",
  "name": "Linen Kaftan",
  "description": "Breathable linen kaftan, relaxed fit.",
  "category": {
    "id": "c1a2b3c4-d5e6-4f70-8a91-b2c3d4e5f6a7",
    "name": "Kaftans",
    "slug": "kaftans"
  },
  "basePrice": "350.00",
  "currency": "GHS",
  "variants": [
    {
      "id": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
      "size": "M",
      "color": "Black",
      "price": "350.00",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
      "stock": 12
    },
    {
      "id": "2a3b4c5d-6e7f-4809-9a1b-2c3d4e5f6071",
      "size": "L",
      "color": "White",
      "price": "380.00",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-white.jpg",
      "stock": 0
    }
  ]
}
```

Errors:

- 404 `NOT_FOUND`: no product with this slug, or it's inactive.
- 429 `RATE_LIMITED`

Notes:

- Looked up by slug so store pages have readable URLs like `/products/linen-kaftan`, which helps search engines.
- `price` is the variant's own price if it has one, otherwise the product's `basePrice`.
- Inactive variants aren't listed.


# Categories

## List categories

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/categories
- Auth: none

Success response: 200 OK

```json
{
  "items": [
    {
      "id": "c1a2b3c4-d5e6-4f70-8a91-b2c3d4e5f6a7",
      "name": "Kaftans",
      "slug": "kaftans"
    },
    {
      "id": "d2b3c4d5-e6f7-4081-9a2b-c3d4e5f6a7b8",
      "name": "Dresses",
      "slug": "dresses"
    }
  ]
}
```

Notes:

- Not paginated. There will only ever be a handful of categories.

## Create category

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/categories
- Auth: admin

Request body:

- `name` (string, required): display name.
- `slug` (string, required): lowercase letters, numbers and hyphens only.

Example request:

```json
{
  "name": "Shirts",
  "slug": "shirts"
}
```

Success response: 201 Created

```json
{
  "id": "e3c4d5e6-f708-4192-8a3b-d4e5f6a7b8c9",
  "name": "Shirts",
  "slug": "shirts"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 409 `ALREADY_EXISTS`: the slug is taken.

## Update category

- Method: PATCH
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/categories/{categoryId}
- Auth: admin

Path parameters:

- `categoryId`: the category's ID.

Request body (send only what changes):

- `name` (string, optional)
- `slug` (string, optional)

Example request:

```json
{
  "name": "Shirts & Tops"
}
```

Success response: 200 OK

```json
{
  "id": "e3c4d5e6-f708-4192-8a3b-d4e5f6a7b8c9",
  "name": "Shirts & Tops",
  "slug": "shirts"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 404 `NOT_FOUND`
- 409 `ALREADY_EXISTS`: the new slug is taken.

## Delete category

- Method: DELETE
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/categories/{categoryId}
- Auth: admin

Path parameters:

- `categoryId`: the category's ID.

Success response: 204 No Content (empty body)

Errors:

- 404 `NOT_FOUND`
- 409 `CATEGORY_IN_USE`: products are still in this category. Move them to another category first.


# Addresses

## List my addresses

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/addresses
- Auth: customer

Success response: 200 OK

```json
{
  "items": [
    {
      "id": "3f1c2a9e-8b7d-4e6f-9a0b-1c2d3e4f5a6b",
      "recipientName": "Ama Mensah",
      "recipientPhone": "0241234567",
      "region": "Greater Accra",
      "city": "East Legon",
      "landmarkOrGps": "GA-492-7461, near A&C Mall",
      "isDefault": true
    }
  ]
}
```

Errors:

- 401 `UNAUTHENTICATED`

Notes:

- Only the caller's own addresses. Deleted (archived) addresses aren't listed.

## Add an address

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/addresses
- Auth: customer

Request body:

- `recipientName` (string, required): who receives the parcel.
- `recipientPhone` (string, required): the number the courier calls.
- `region` (string, required): one of Ghana's 16 regions.
- `city` (string, required): city or town.
- `landmarkOrGps` (string, optional): a landmark or a GhanaPost GPS code.
- `isDefault` (boolean, optional): make this the default address. Default false.

Example request:

```json
{
  "recipientName": "Ama Mensah",
  "recipientPhone": "0241234567",
  "region": "Greater Accra",
  "city": "East Legon",
  "landmarkOrGps": "GA-492-7461, near A&C Mall",
  "isDefault": true
}
```

Success response: 201 Created

```json
{
  "id": "3f1c2a9e-8b7d-4e6f-9a0b-1c2d3e4f5a6b",
  "recipientName": "Ama Mensah",
  "recipientPhone": "0241234567",
  "region": "Greater Accra",
  "city": "East Legon",
  "landmarkOrGps": "GA-492-7461, near A&C Mall",
  "isDefault": true
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 401 `UNAUTHENTICATED`

Notes:

- Setting `isDefault: true` removes the default from the customer's other addresses.

## Update an address

- Method: PATCH
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/addresses/{addressId}
- Auth: customer, must own the address

Path parameters:

- `addressId`: the address's ID.

Request body: any of the fields from Add an address. Send only what changes.

Example request:

```json
{
  "landmarkOrGps": "GA-492-7461, second gate"
}
```

Success response: 200 OK

```json
{
  "id": "3f1c2a9e-8b7d-4e6f-9a0b-1c2d3e4f5a6b",
  "recipientName": "Ama Mensah",
  "recipientPhone": "0241234567",
  "region": "Greater Accra",
  "city": "East Legon",
  "landmarkOrGps": "GA-492-7461, second gate",
  "isDefault": true
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 404 `NOT_FOUND`: doesn't exist or isn't yours.

Notes:

- Doesn't affect past orders. Each order keeps its own copy of the address it shipped to.

## Delete an address

- Method: DELETE
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/addresses/{addressId}
- Auth: customer, must own the address

Path parameters:

- `addressId`: the address's ID.

Success response: 204 No Content (empty body)

Errors:

- 404 `NOT_FOUND`

Notes:

- The address is hidden (archived), not erased. It disappears from lists and checkout. Always allowed, because past orders don't depend on it.


# Cart

Every cart endpoint works for a signed-in customer or a guest. The response is the same shape for both.

## Get my cart

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/cart
- Auth: customer or guest

Success response: 200 OK

```json
{
  "id": "7c6b5a49-3827-4165-9f8e-7d6c5b4a3928",
  "items": [
    {
      "id": "e1d2c3b4-a596-4877-8695-a4b3c2d1e0f9",
      "quantity": 2,
      "product": {
        "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
        "slug": "linen-kaftan",
        "name": "Linen Kaftan"
      },
      "variant": {
        "id": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
        "size": "M",
        "color": "Black",
        "price": "350.00",
        "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
        "stock": 12
      },
      "lineTotal": "700.00",
      "available": true
    }
  ],
  "subtotal": "700.00",
  "currency": "GHS"
}
```

Success response when there's no cart yet: 200 OK

```json
{
  "id": null,
  "items": [],
  "subtotal": "0.00",
  "currency": "GHS"
}
```

Notes:

- `available` is false when the product or variant was deactivated, or there's less stock than the quantity in the cart. The cart page should flag that line instead of letting checkout fail.

## Add to cart

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/cart/items
- Auth: customer or guest

Request body:

- `variantId` (string, required): the size/colour being added.
- `quantity` (integer, required): 1 or more.

Example request:

```json
{
  "variantId": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
  "quantity": 1
}
```

Success response: 200 OK. Returns the whole cart, same as Get my cart.

```json
{
  "id": "7c6b5a49-3827-4165-9f8e-7d6c5b4a3928",
  "items": [
    {
      "id": "e1d2c3b4-a596-4877-8695-a4b3c2d1e0f9",
      "quantity": 3,
      "product": {
        "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
        "slug": "linen-kaftan",
        "name": "Linen Kaftan"
      },
      "variant": {
        "id": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
        "size": "M",
        "color": "Black",
        "price": "350.00",
        "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
        "stock": 12
      },
      "lineTotal": "1050.00",
      "available": true
    }
  ],
  "subtotal": "1050.00",
  "currency": "GHS"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 400 `INSUFFICIENT_STOCK`: the cart would hold more than is in stock.
- 404 `NOT_FOUND`: the variant doesn't exist, or the product or variant is inactive.
- 429 `RATE_LIMITED`

Notes:

- If the variant is already in the cart, the quantity is added to that line instead of creating a second line.
- For a first-time guest, this creates the cart and sets the `guest_id` cookie.
- The stock check here is only a courtesy. Stock is actually taken at checkout.

## Change quantity

- Method: PATCH
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/cart/items/{cartItemId}
- Auth: customer or guest, must own the cart

Path parameters:

- `cartItemId`: the cart line's ID (`items[].id` from Get my cart).

Request body:

- `quantity` (integer, required): the new quantity, 1 or more. To remove the line, use Remove from cart.

Example request:

```json
{
  "quantity": 1
}
```

Success response: 200 OK. Returns the whole cart, same as Get my cart.

```json
{
  "id": "7c6b5a49-3827-4165-9f8e-7d6c5b4a3928",
  "items": [
    {
      "id": "e1d2c3b4-a596-4877-8695-a4b3c2d1e0f9",
      "quantity": 1,
      "product": {
        "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
        "slug": "linen-kaftan",
        "name": "Linen Kaftan"
      },
      "variant": {
        "id": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
        "size": "M",
        "color": "Black",
        "price": "350.00",
        "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
        "stock": 12
      },
      "lineTotal": "350.00",
      "available": true
    }
  ],
  "subtotal": "350.00",
  "currency": "GHS"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 400 `INSUFFICIENT_STOCK`
- 404 `NOT_FOUND`: no such line in your cart.

## Remove from cart

- Method: DELETE
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/cart/items/{cartItemId}
- Auth: customer or guest, must own the cart

Path parameters:

- `cartItemId`: the cart line's ID.

Success response: 200 OK. Returns the whole cart.

```json
{
  "id": "7c6b5a49-3827-4165-9f8e-7d6c5b4a3928",
  "items": [],
  "subtotal": "0.00",
  "currency": "GHS"
}
```

Errors:

- 404 `NOT_FOUND`

## Empty the cart

- Method: DELETE
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/cart
- Auth: customer or guest

Success response: 204 No Content (empty body)

## Merge guest cart

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/cart/merge
- Auth: customer, and the guest ID (cookie on web, `X-Guest-Id` header on mobile)

Request body: none.

Success response: 200 OK. Returns the customer's cart after the merge.

```json
{
  "id": "7c6b5a49-3827-4165-9f8e-7d6c5b4a3928",
  "items": [
    {
      "id": "e1d2c3b4-a596-4877-8695-a4b3c2d1e0f9",
      "quantity": 2,
      "product": {
        "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
        "slug": "linen-kaftan",
        "name": "Linen Kaftan"
      },
      "variant": {
        "id": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
        "size": "M",
        "color": "Black",
        "price": "350.00",
        "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
        "stock": 12
      },
      "lineTotal": "700.00",
      "available": true
    }
  ],
  "subtotal": "700.00",
  "currency": "GHS"
}
```

Errors:

- 401 `UNAUTHENTICATED`

Notes:

- Call this right after sign-in or sign-up when the visitor had a guest cart.
- If the customer had no cart, the guest cart simply becomes theirs.
- If they already had a cart, the guest items are added in. The same variant has its quantities added together, capped at what's in stock. Then the guest cart is deleted.
- Clears the `guest_id` cookie.
- Safe to call twice. The second time there's no guest cart left, so nothing happens.
- Guest carts untouched for 30 days are deleted by a daily cron job.


# Orders (customer)

## Place an order (checkout)

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/orders
- Auth: customer (guests sign in and merge their cart first)

Headers:

- `Idempotency-Key` (required): a UUID generated once when the checkout page opens. See Idempotency.

Request body:

- `addressId` (string, required): one of the customer's saved addresses.
- `paymentMethod` (string, required): `"paystack"` or `"manual_momo"`.

Example request:

```
POST https://e-commerce-web-psi-sage.vercel.app/api/v1/orders
Idempotency-Key: 8f14e45f-ceea-467f-a8c4-3e2b1d9a0c11
Content-Type: application/json
```

```json
{
  "addressId": "3f1c2a9e-8b7d-4e6f-9a0b-1c2d3e4f5a6b",
  "paymentMethod": "paystack"
}
```

Success response: 201 Created

```json
{
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "status": "pending",
  "paymentMethod": "paystack",
  "subtotal": "700.00",
  "deliveryFee": "0.00",
  "totalAmount": "700.00",
  "currency": "GHS",
  "expiresAt": "2026-10-20T10:30:00.000Z"
}
```

If the same `Idempotency-Key` is sent again, the same body comes back with 200 OK.

Errors:

- 400 `MISSING_IDEMPOTENCY_KEY`
- 400 `VALIDATION_ERROR`
- 400 `CART_EMPTY`
- 400 `INSUFFICIENT_STOCK`: `details.items` lists each short item and how many are left.
- 400 `PRODUCT_UNAVAILABLE`: something in the cart was deactivated.
- 404 `NOT_FOUND`: the address doesn't exist or isn't yours.
- 409 `IDEMPOTENCY_KEY_REUSED`: same key, different body.
- 429 `RATE_LIMITED`

Example `INSUFFICIENT_STOCK` error:

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Some items don't have enough stock",
    "details": {
      "items": [
        {
          "variantId": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
          "name": "Linen Kaftan, Black / M",
          "requested": 3,
          "available": 1
        }
      ]
    }
  }
}
```

Notes:

- Everything happens in one database transaction. Either all of it happens or none of it does:
  1. Check the address is yours and not deleted.
  2. Lock the stock rows and check every item is still active and in stock, so two people can't both buy the last one.
  3. Create the order, copying prices, product names, sizes, colours and the full shipping address onto it.
  4. Take the stock.
  5. Empty the cart.
  6. Notify admins of the new order.
- `expiresAt` is the payment deadline: 30 minutes for Paystack, 24 hours for manual MoMo. An unpaid order past its deadline is cancelled and its stock goes back on sale.
- `deliveryFee` is always `"0.00"` until delivery pricing is decided.

## My orders

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/orders
- Auth: customer

Query parameters (all optional):

- `status`: only orders with this status, for example `delivered`.
- `page`, `limit`: see Pagination.

Success response: 200 OK. Newest first.

```json
{
  "items": [
    {
      "id": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
      "status": "processing",
      "paymentMethod": "paystack",
      "totalAmount": "700.00",
      "currency": "GHS",
      "itemCount": 2,
      "firstItem": {
        "productName": "Linen Kaftan",
        "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg"
      },
      "createdAt": "2026-10-20T10:00:00.000Z"
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 1,
  "totalPages": 1
}
```

Errors:

- 401 `UNAUTHENTICATED`

Notes:

- A short summary per order for the order history list. Use Get an order for the full detail.

## Get an order

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/orders/{orderId}
- Auth: customer, must own the order

Path parameters:

- `orderId`: the order's ID.

Success response: 200 OK

```json
{
  "id": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "status": "delivered",
  "paymentMethod": "paystack",
  "subtotal": "700.00",
  "deliveryFee": "0.00",
  "totalAmount": "700.00",
  "currency": "GHS",
  "expiresAt": "2026-10-20T10:30:00.000Z",
  "shippingAddress": {
    "recipientName": "Ama Mensah",
    "recipientPhone": "0241234567",
    "region": "Greater Accra",
    "city": "East Legon",
    "landmarkOrGps": "GA-492-7461, near A&C Mall"
  },
  "items": [
    {
      "id": "0a9b8c7d-6e5f-4a3b-9c2d-1e0f9a8b7c6d",
      "productName": "Linen Kaftan",
      "size": "M",
      "color": "Black",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
      "unitPrice": "350.00",
      "quantity": 2,
      "lineTotal": "700.00"
    }
  ],
  "payment": {
    "provider": "paystack",
    "status": "success",
    "amount": "700.00",
    "paidAt": "2026-10-20T10:04:12.000Z"
  },
  "delivery": {
    "status": "delivered",
    "provider": "Shaq Express",
    "providerReference": "SHQ-58213",
    "deliveredAt": "2026-10-22T15:20:00.000Z"
  },
  "return": null,
  "refunds": [],
  "cancellation": null,
  "returnEligibleUntil": "2026-11-05T15:20:00.000Z",
  "createdAt": "2026-10-20T10:00:00.000Z",
  "updatedAt": "2026-10-22T15:20:00.000Z"
}
```

What the optional parts look like when they're filled in:

```json
{
  "return": {
    "id": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
    "status": "return_approved",
    "reason": "Too small",
    "rejectionReason": null,
    "items": [
      {
        "orderItemId": "0a9b8c7d-6e5f-4a3b-9c2d-1e0f9a8b7c6d",
        "productName": "Linen Kaftan",
        "quantity": 1
      }
    ],
    "requestedAt": "2026-10-24T09:00:00.000Z"
  },
  "refunds": [
    {
      "id": "8293a4b5-c6d7-4e8f-90a1-b2c3d4e5f607",
      "status": "initiated",
      "amount": "350.00",
      "currency": "GHS",
      "completedAt": null
    }
  ],
  "cancellation": {
    "cancelledAt": "2026-10-20T10:31:00.000Z",
    "cancelledBy": "system",
    "reason": "expired"
  }
}
```

Errors:

- 404 `NOT_FOUND`: doesn't exist or isn't yours.

Notes:

- `payment`, `delivery`, `return` and `cancellation` are `null` when they don't apply. `refunds` is an empty list when there are none.
- `returnEligibleUntil` is the delivery date plus 14 days, or `null` if not delivered. The app uses it to show or hide the return button.
- `cancelledBy` is `customer`, `admin` or `system` (expired).

## Pay with Paystack

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/orders/{orderId}/pay
- Auth: customer, must own the order

Path parameters:

- `orderId`: the order's ID.

Request body: none.

Success response: 200 OK

```json
{
  "authorizationUrl": "https://checkout.paystack.com/0peioxfhpn",
  "accessCode": "0peioxfhpn",
  "reference": "ord_a7d9e3f1_1729418400"
}
```

Errors:

- 400 `WRONG_PAYMENT_METHOD`: the order was placed as manual MoMo.
- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the order isn't `pending`, or its payment deadline has passed.
- 502 `PAYMENT_PROVIDER_ERROR`
- 503 `PAYMENT_PROVIDER_UNAVAILABLE`

Notes:

- Send the customer to `authorizationUrl`. On web, open it in the same tab. On mobile, open it in an in-app browser.
- After paying, Paystack sends the customer back to the store's `/checkout/complete?orderId=...` page.
- Can be called again if the customer abandoned the Paystack page, until the deadline. Each attempt gets its own reference.
- The order ID is sent to Paystack in the payment's metadata.

## Check payment after returning from Paystack

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/orders/{orderId}/verify-payment
- Auth: customer, must own the order

Path parameters:

- `orderId`: the order's ID.

Request body:

- `reference` (string, required): the reference Paystack adds to the return URL.

Example request:

```json
{
  "reference": "ord_a7d9e3f1_1729418400"
}
```

Success response: 200 OK

```json
{
  "paymentStatus": "paid",
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "orderStatus": "processing"
}
```

`paymentStatus` is `paid`, `pending` (Paystack hasn't finished) or `failed`.

Errors:

- 400 `VALIDATION_ERROR`: the reference doesn't belong to this order.
- 404 `NOT_FOUND`
- 502 `PAYMENT_PROVIDER_ERROR`
- 503 `PAYMENT_PROVIDER_UNAVAILABLE`

Notes:

- The checkout-complete page calls this as soon as the customer lands. It asks Paystack directly instead of waiting for the webhook, which can take seconds or minutes.
- It runs the exact same confirmation code as the webhook, including the amount check. Whichever arrives first does the work. The second does nothing.

## Cancel my order

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/orders/{orderId}/cancel
- Auth: customer, must own the order

Path parameters:

- `orderId`: the order's ID.

Request body:

- `reason` (string, optional): why they're cancelling.

Example request:

```json
{
  "reason": "Ordered the wrong size"
}
```

Success response: 200 OK

```json
{
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "status": "cancelled",
  "cancellation": {
    "cancelledAt": "2026-10-20T11:00:00.000Z",
    "cancelledBy": "customer",
    "reason": "Ordered the wrong size"
  },
  "refund": {
    "id": "8293a4b5-c6d7-4e8f-90a1-b2c3d4e5f607",
    "status": "pending",
    "amount": "700.00",
    "currency": "GHS"
  }
}
```

`refund` is `null` if the order hadn't been paid.

Errors:

- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the order is already `shipped`, `delivered` or `cancelled`.

Notes:

- Allowed while the order is `pending`, `paid` or `processing`.
- The stock goes back on sale.
- If the order was paid, a refund is created in `pending` and admins are notified. No money moves until an admin starts the refund. Customers can't start refunds themselves.

## Request a return

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/orders/{orderId}/return
- Auth: customer, must own the order

Path parameters:

- `orderId`: the order's ID.

Request body:

- `reason` (string, optional): why they're returning it.
- `items` (list, required): what's being returned. Each entry has:
  - `orderItemId` (string, required): the order line (`items[].id` from Get an order).
  - `quantity` (integer, required): how many, from 1 up to how many were bought.

Example request:

```json
{
  "reason": "Too small",
  "items": [
    {
      "orderItemId": "0a9b8c7d-6e5f-4a3b-9c2d-1e0f9a8b7c6d",
      "quantity": 1
    }
  ]
}
```

Success response: 201 Created

```json
{
  "id": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "status": "return_requested",
  "reason": "Too small",
  "items": [
    {
      "orderItemId": "0a9b8c7d-6e5f-4a3b-9c2d-1e0f9a8b7c6d",
      "productName": "Linen Kaftan",
      "size": "M",
      "color": "Black",
      "quantity": 1,
      "unitPrice": "350.00"
    }
  ],
  "refundAmount": "350.00",
  "requestedAt": "2026-10-24T09:00:00.000Z"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 400 `RETURN_WINDOW_CLOSED`: more than 14 days since delivery.
- 400 `INVALID_RETURN_QUANTITY`: a line isn't from this order, a quantity is more than was bought, or the same line is listed twice.
- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the order isn't `delivered`.
- 409 `ALREADY_EXISTS`: this order already has a return. Only one return attempt is allowed per order, ever.

Notes:

- Admins are notified.
- `refundAmount` is the price of the returned items only. The delivery fee is never refunded.


# Webhooks

## Paystack webhook

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/webhooks/paystack
- Auth: none. Paystack calls this, and the signature proves it.

Headers:

- `x-paystack-signature` (required): HMAC-SHA512 of the raw request body, made with the Paystack secret key.

Example request (sent by Paystack):

```json
{
  "event": "charge.success",
  "data": {
    "id": 4099260516,
    "reference": "ord_a7d9e3f1_1729418400",
    "amount": 70000,
    "currency": "GHS",
    "status": "success",
    "paid_at": "2026-10-20T10:04:12.000Z",
    "metadata": {
      "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60"
    }
  }
}
```

Success response: 200 OK (empty body)

Errors:

- 401: the signature is wrong. Nothing is processed.
- 500: something broke while processing. Paystack will retry.

Notes:

- The signature is checked against the raw body before anything is read or trusted.
- Every event is saved in a `webhook_events` table with Paystack's event ID. If the same event arrives again (Paystack retries), it's skipped. Saving the event and acting on it happen in one transaction, so a crash halfway through leaves nothing behind and the retry works cleanly.
- What each event does:
  - `charge.success`: runs the same payment confirmation as Check payment.
    1. Find the payment by `reference` and the order by `metadata.orderId`. They must match.
    2. Check the amount and currency. `amount` is in pesewas and must equal the order total × 100, and `currency` must be `GHS`. If not, the payment is marked `amount_mismatch`, the order is not moved forward, and admins are notified.
    3. If the order is `pending`: payment becomes `success`, order goes `pending` → `paid` → `processing`.
    4. If the order is already paid or further along: nothing happens (duplicate).
    5. If the order was cancelled because its deadline passed (late payment): try to take the stock again. If every item is available, the order goes to `processing` as normal. If not, it stays cancelled, a `pending` refund is created and admins are notified.
  - `charge.failed`: the payment is marked `failed`. The order stays `pending`, so the customer can try again before the deadline.
  - `refund.processed`: the refund becomes `completed`. If it came from a return, the return becomes `refunded`.
  - `refund.failed`: the refund becomes `failed` with Paystack's reason, and admins are notified so they can retry.
  - Any other event: saved and ignored.


# Scheduled jobs (cron)

Run by Vercel Cron. Each must carry the header `Authorization: Bearer <CRON_SECRET>`, which Vercel adds automatically. Anyone else gets 401.

## Expire unpaid orders

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/cron/expire-orders
- Schedule: every 5 minutes

Success response: 200 OK

```json
{
  "expired": 3,
  "recoveredAsPaid": 1
}
```

Notes:

- Finds every `pending` order past its `expiresAt`. For each, in its own transaction: checks it's still `pending`, cancels it with reason `expired`, and puts the stock back.
- For Paystack orders, it first asks Paystack whether the payment actually went through. If it did, the order is confirmed as paid instead of cancelled (counted in `recoveredAsPaid`).
- The same deadline check also runs whenever a single order is opened, so a customer never sees a stale `pending` order just because the job hasn't run yet.

## Clean up old guest carts

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/cron/cleanup-guest-carts
- Schedule: daily

Success response: 200 OK

```json
{
  "deleted": 42
}
```

Notes:

- Deletes guest carts that haven't changed in 30 days.


# Admin: uploads

## Get an image upload link

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/uploads
- Auth: admin

Request body:

- `filename` (string, required): original file name.
- `contentType` (string, required): `image/jpeg`, `image/png` or `image/webp`.
- `size` (integer, required): file size in bytes. Max 5 MB (5242880).

Example request:

```json
{
  "filename": "linen-kaftan-black.jpg",
  "contentType": "image/jpeg",
  "size": 845210
}
```

Success response: 200 OK

```json
{
  "uploadUrl": "https://blob.vercel-storage.com/upload?token=...",
  "publicUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
  "expiresAt": "2026-10-20T10:15:00.000Z"
}
```

Errors:

- 400 `VALIDATION_ERROR`: unsupported file type.
- 413 `FILE_TOO_LARGE`: over 5 MB.

Notes:

- The admin page uploads the file straight to `uploadUrl`, then sends `publicUrl` as `imageUrl` when creating or editing a variant. Big files never pass through our own server.
- Storage is Vercel Blob.


# Admin: products

## List products (admin)

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/products
- Auth: admin

Query parameters (all optional):

- `search`: text in the product name.
- `category`: category slug.
- `isActive`: `true` or `false`.
- `lowStock`: `true` to show only products with at least one variant at or below its low-stock level.
- `page`, `limit`: see Pagination.

Success response: 200 OK

```json
{
  "items": [
    {
      "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
      "slug": "linen-kaftan",
      "name": "Linen Kaftan",
      "description": "Breathable linen kaftan, relaxed fit.",
      "category": {
        "id": "c1a2b3c4-d5e6-4f70-8a91-b2c3d4e5f6a7",
        "name": "Kaftans",
        "slug": "kaftans"
      },
      "basePrice": "350.00",
      "isActive": true,
      "variants": [
        {
          "id": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
          "size": "M",
          "color": "Black",
          "priceOverride": null,
          "price": "350.00",
          "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
          "isActive": true,
          "stock": 3,
          "lowStockThreshold": 5,
          "isLowStock": true
        }
      ],
      "createdAt": "2026-10-01T09:00:00.000Z",
      "updatedAt": "2026-10-18T14:00:00.000Z"
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 1,
  "totalPages": 1
}
```

Notes:

- Includes inactive products and variants, unlike the public list.

## Get a product (admin)

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/products/{productId}
- Auth: admin

Path parameters:

- `productId`: the product's ID.

Success response: 200 OK. One product, same shape as one item in List products (admin).

```json
{
  "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
  "slug": "linen-kaftan",
  "name": "Linen Kaftan",
  "description": "Breathable linen kaftan, relaxed fit.",
  "category": {
    "id": "c1a2b3c4-d5e6-4f70-8a91-b2c3d4e5f6a7",
    "name": "Kaftans",
    "slug": "kaftans"
  },
  "basePrice": "350.00",
  "isActive": true,
  "variants": [
    {
      "id": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
      "size": "M",
      "color": "Black",
      "priceOverride": null,
      "price": "350.00",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
      "isActive": true,
      "stock": 3,
      "lowStockThreshold": 5,
      "isLowStock": true
    }
  ],
  "createdAt": "2026-10-01T09:00:00.000Z",
  "updatedAt": "2026-10-18T14:00:00.000Z"
}
```

Errors:

- 404 `NOT_FOUND`

## Create a product

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/products
- Auth: admin

Request body:

- `name` (string, required)
- `slug` (string, optional): made from the name if left out, for example "Linen Kaftan" becomes `linen-kaftan`.
- `description` (string, optional)
- `categoryId` (string, optional)
- `basePrice` (string, required): for example `"350.00"`.
- `variants` (list, required, at least one). Each entry has:
  - `size` (string, required)
  - `color` (string, required)
  - `priceOverride` (string, optional): use when this variant costs something other than `basePrice`.
  - `imageUrl` (string, optional): the `publicUrl` from Get an image upload link.
  - `initialStock` (integer, required): 0 or more.
  - `lowStockThreshold` (integer, optional): default 5.

Example request:

```json
{
  "name": "Linen Kaftan",
  "description": "Breathable linen kaftan, relaxed fit.",
  "categoryId": "c1a2b3c4-d5e6-4f70-8a91-b2c3d4e5f6a7",
  "basePrice": "350.00",
  "variants": [
    {
      "size": "M",
      "color": "Black",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
      "initialStock": 12
    },
    {
      "size": "L",
      "color": "White",
      "priceOverride": "380.00",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-white.jpg",
      "initialStock": 8,
      "lowStockThreshold": 3
    }
  ]
}
```

Success response: 201 Created

```json
{
  "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
  "slug": "linen-kaftan",
  "name": "Linen Kaftan",
  "description": "Breathable linen kaftan, relaxed fit.",
  "category": {
    "id": "c1a2b3c4-d5e6-4f70-8a91-b2c3d4e5f6a7",
    "name": "Kaftans",
    "slug": "kaftans"
  },
  "basePrice": "350.00",
  "isActive": true,
  "variants": [
    {
      "id": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
      "size": "M",
      "color": "Black",
      "priceOverride": null,
      "price": "350.00",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
      "isActive": true,
      "stock": 12,
      "lowStockThreshold": 5,
      "isLowStock": false
    },
    {
      "id": "2a3b4c5d-6e7f-4809-9a1b-2c3d4e5f6071",
      "size": "L",
      "color": "White",
      "priceOverride": "380.00",
      "price": "380.00",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-white.jpg",
      "isActive": true,
      "stock": 8,
      "lowStockThreshold": 3,
      "isLowStock": false
    }
  ],
  "createdAt": "2026-10-20T10:00:00.000Z",
  "updatedAt": "2026-10-20T10:00:00.000Z"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 404 `NOT_FOUND`: the category doesn't exist.
- 409 `ALREADY_EXISTS`: the slug is taken, or the same size and colour appear twice.

Notes:

- The product, its variants and their stock rows are created in one transaction.

## Edit a product

- Method: PATCH
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/products/{productId}
- Auth: admin

Path parameters:

- `productId`: the product's ID.

Request body (send only what changes):

- `name` (string, optional)
- `slug` (string, optional)
- `description` (string, optional)
- `categoryId` (string or null, optional)
- `basePrice` (string, optional)
- `isActive` (boolean, optional): `true` to bring a deactivated product back.

Example request:

```json
{
  "basePrice": "320.00",
  "description": "Breathable linen kaftan, relaxed fit. Hand wash only."
}
```

Success response: 200 OK. The full product, same shape as Get a product (admin).

```json
{
  "id": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
  "slug": "linen-kaftan",
  "name": "Linen Kaftan",
  "description": "Breathable linen kaftan, relaxed fit. Hand wash only.",
  "category": {
    "id": "c1a2b3c4-d5e6-4f70-8a91-b2c3d4e5f6a7",
    "name": "Kaftans",
    "slug": "kaftans"
  },
  "basePrice": "320.00",
  "isActive": true,
  "variants": [
    {
      "id": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
      "size": "M",
      "color": "Black",
      "priceOverride": null,
      "price": "320.00",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
      "isActive": true,
      "stock": 12,
      "lowStockThreshold": 5,
      "isLowStock": false
    }
  ],
  "createdAt": "2026-10-20T10:00:00.000Z",
  "updatedAt": "2026-10-21T08:00:00.000Z"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 404 `NOT_FOUND`
- 409 `ALREADY_EXISTS`: the new slug is taken.

Notes:

- The store's product pages are refreshed straight away.
- Changing `slug` breaks old links to the product. The admin page should warn about this.

## Deactivate a product

- Method: DELETE
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/products/{productId}
- Auth: admin

Path parameters:

- `productId`: the product's ID.

Success response: 204 No Content (empty body)

Errors:

- 404 `NOT_FOUND`

Notes:

- Hides the product (`isActive: false`). It isn't erased, because past orders refer to it.
- To bring it back, use Edit a product with `isActive: true`.

## Add a variant

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/products/{productId}/variants
- Auth: admin

Path parameters:

- `productId`: the product's ID.

Request body:

- `size` (string, required)
- `color` (string, required)
- `priceOverride` (string, optional)
- `imageUrl` (string, optional)
- `initialStock` (integer, required)
- `lowStockThreshold` (integer, optional): default 5.

Example request:

```json
{
  "size": "XL",
  "color": "Black",
  "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
  "initialStock": 6
}
```

Success response: 201 Created

```json
{
  "id": "3b4c5d6e-7f80-4912-8a3b-4c5d6e7f8091",
  "productId": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
  "size": "XL",
  "color": "Black",
  "priceOverride": null,
  "price": "350.00",
  "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
  "isActive": true,
  "stock": 6,
  "lowStockThreshold": 5,
  "isLowStock": false
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 404 `NOT_FOUND`: the product doesn't exist.
- 409 `ALREADY_EXISTS`: this product already has that size and colour.

## Edit a variant

- Method: PATCH
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/variants/{variantId}
- Auth: admin

Path parameters:

- `variantId`: the variant's ID.

Request body (send only what changes):

- `size` (string, optional)
- `color` (string, optional)
- `priceOverride` (string or null, optional): `null` goes back to the product's base price.
- `imageUrl` (string, optional)
- `isActive` (boolean, optional)

Example request:

```json
{
  "priceOverride": "300.00"
}
```

Success response: 200 OK

```json
{
  "id": "3b4c5d6e-7f80-4912-8a3b-4c5d6e7f8091",
  "productId": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
  "size": "XL",
  "color": "Black",
  "priceOverride": "300.00",
  "price": "300.00",
  "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
  "isActive": true,
  "stock": 6,
  "lowStockThreshold": 5,
  "isLowStock": false
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 404 `NOT_FOUND`
- 409 `ALREADY_EXISTS`: another variant of this product already has that size and colour.

Notes:

- Stock isn't changed here. Use Set stock.

## Deactivate a variant

- Method: DELETE
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/variants/{variantId}
- Auth: admin

Path parameters:

- `variantId`: the variant's ID.

Success response: 204 No Content (empty body)

Errors:

- 404 `NOT_FOUND`

Notes:

- Hides the variant from the store. It's kept for order history. Carts that contain it show the line as unavailable.


# Admin: stock

## List stock

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/inventory
- Auth: admin

Query parameters (all optional):

- `lowStock`: `true` to show only variants at or below their low-stock level, lowest first.
- `search`: text in the product name.
- `page`, `limit`: see Pagination.

Success response: 200 OK

```json
{
  "items": [
    {
      "variantId": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
      "productId": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
      "productName": "Linen Kaftan",
      "size": "M",
      "color": "Black",
      "quantity": 3,
      "lowStockThreshold": 5,
      "isLowStock": true
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 1,
  "totalPages": 1
}
```

## Set stock

- Method: PATCH
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/inventory/{variantId}
- Auth: admin

Path parameters:

- `variantId`: the variant's ID.

Request body:

- `quantity` (integer, required): the new total in stock, 0 or more. This replaces the number, it doesn't add to it (a stock count).
- `lowStockThreshold` (integer, optional)

Example request:

```json
{
  "quantity": 20,
  "lowStockThreshold": 4
}
```

Success response: 200 OK

```json
{
  "variantId": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
  "productId": "9a1b2c3d-4e5f-4a6b-8c7d-0e1f2a3b4c5d",
  "productName": "Linen Kaftan",
  "size": "M",
  "color": "Black",
  "quantity": 20,
  "lowStockThreshold": 4,
  "isLowStock": false
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 404 `NOT_FOUND`

Notes:

- The store's product pages are refreshed straight away.


# Admin: orders

## List orders (admin)

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/orders
- Auth: admin

Query parameters (all optional):

- `status`: for example `processing`.
- `paymentMethod`: `paystack` or `manual_momo`.
- `search`: the start of an order ID, or a customer's name, email or phone.
- `from`, `to`: dates, for example `2026-10-01`. Filters on when the order was placed.
- `page`, `limit`: see Pagination.

Success response: 200 OK. Newest first.

```json
{
  "items": [
    {
      "id": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
      "status": "processing",
      "paymentMethod": "paystack",
      "totalAmount": "700.00",
      "currency": "GHS",
      "itemCount": 2,
      "customer": {
        "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
        "fullName": "Ama Mensah",
        "email": "ama.mensah@example.com",
        "phone": "0241234567"
      },
      "deliveryStatus": null,
      "hasReturn": false,
      "createdAt": "2026-10-20T10:00:00.000Z"
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 1,
  "totalPages": 1
}
```

## Get an order (admin)

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/orders/{orderId}
- Auth: admin

Path parameters:

- `orderId`: the order's ID.

Success response: 200 OK

```json
{
  "id": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "status": "shipped",
  "paymentMethod": "manual_momo",
  "subtotal": "700.00",
  "deliveryFee": "0.00",
  "totalAmount": "700.00",
  "currency": "GHS",
  "expiresAt": "2026-10-21T10:00:00.000Z",
  "customer": {
    "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
    "fullName": "Ama Mensah",
    "email": "ama.mensah@example.com",
    "phone": "0241234567"
  },
  "shippingAddress": {
    "recipientName": "Ama Mensah",
    "recipientPhone": "0241234567",
    "region": "Greater Accra",
    "city": "East Legon",
    "landmarkOrGps": "GA-492-7461, near A&C Mall"
  },
  "items": [
    {
      "id": "0a9b8c7d-6e5f-4a3b-9c2d-1e0f9a8b7c6d",
      "variantId": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b",
      "productName": "Linen Kaftan",
      "size": "M",
      "color": "Black",
      "imageUrl": "https://blob.vercel-storage.com/products/linen-kaftan-black.jpg",
      "unitPrice": "350.00",
      "quantity": 2,
      "lineTotal": "700.00"
    }
  ],
  "payments": [
    {
      "id": "4b5c6d7e-8f90-4a1b-8c2d-3e4f5a6b7c8d",
      "provider": "manual",
      "providerReference": "MP241020.1015.C12345",
      "status": "success",
      "amount": "700.00",
      "paidAt": "2026-10-20T10:15:00.000Z",
      "markedBy": {
        "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
        "fullName": "Kofi Boateng"
      },
      "note": "Confirmed on business MoMo app"
    }
  ],
  "delivery": {
    "id": "d4e5f607-1829-4a3b-8c4d-5e6f708192a3",
    "status": "en_route",
    "provider": "Shaq Express",
    "providerReference": "SHQ-58213",
    "requestedAt": "2026-10-21T09:00:00.000Z",
    "deliveredAt": null,
    "failureReason": null
  },
  "return": null,
  "refunds": [],
  "cancellation": null,
  "history": [
    {
      "fromStatus": null,
      "toStatus": "pending",
      "actor": "customer",
      "admin": null,
      "note": null,
      "at": "2026-10-20T10:00:00.000Z"
    },
    {
      "fromStatus": "pending",
      "toStatus": "paid",
      "actor": "admin",
      "admin": {
        "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
        "fullName": "Kofi Boateng"
      },
      "note": "Manual MoMo payment",
      "at": "2026-10-20T10:15:00.000Z"
    },
    {
      "fromStatus": "paid",
      "toStatus": "processing",
      "actor": "system",
      "admin": null,
      "note": null,
      "at": "2026-10-20T10:15:00.000Z"
    },
    {
      "fromStatus": "processing",
      "toStatus": "shipped",
      "actor": "admin",
      "admin": {
        "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
        "fullName": "Kofi Boateng"
      },
      "note": null,
      "at": "2026-10-21T09:00:00.000Z"
    }
  ],
  "createdAt": "2026-10-20T10:00:00.000Z",
  "updatedAt": "2026-10-21T09:00:00.000Z"
}
```

Errors:

- 404 `NOT_FOUND`

Notes:

- `return`, `refunds` and `cancellation` look the same as in the customer's Get an order when filled in.
- `payments` lists every attempt, including failed Paystack attempts.
- `history` is every status change: who made it (customer, admin, system or Paystack) and when.

## Mark a MoMo order as paid

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/orders/{orderId}/mark-paid
- Auth: admin

Path parameters:

- `orderId`: the order's ID.

Request body:

- `amount` (string, required): the amount actually received. Must equal the order total.
- `momoReference` (string, required): the MoMo transaction ID, as proof the money arrived.
- `note` (string, optional)

Example request:

```json
{
  "amount": "700.00",
  "momoReference": "MP241020.1015.C12345",
  "note": "Confirmed on business MoMo app"
}
```

Success response: 200 OK

```json
{
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "status": "processing",
  "payment": {
    "id": "4b5c6d7e-8f90-4a1b-8c2d-3e4f5a6b7c8d",
    "provider": "manual",
    "providerReference": "MP241020.1015.C12345",
    "status": "success",
    "amount": "700.00",
    "paidAt": "2026-10-20T10:15:00.000Z",
    "markedBy": {
      "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
      "fullName": "Kofi Boateng"
    }
  }
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 400 `AMOUNT_MISMATCH`: `amount` isn't the order total.
- 400 `WRONG_PAYMENT_METHOD`: the order was placed for Paystack.
- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the order isn't `pending`.
- 409 `ALREADY_EXISTS`: this MoMo reference was already used on another payment.

Notes:

- Records which admin confirmed it and when. This skips automatic checks, so it must always be traceable.
- The order goes `pending` → `paid` → `processing`, same as a Paystack payment.
- Also works on an order cancelled for missing its deadline, if the stock can be taken again (same rule as a late Paystack payment).

## Cancel an order (admin)

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/orders/{orderId}/cancel
- Auth: admin

Path parameters:

- `orderId`: the order's ID.

Request body:

- `reason` (string, required): why it's being cancelled.

Example request:

```json
{
  "reason": "Customer called to cancel; courier not yet picked up"
}
```

Success response: 200 OK

```json
{
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "status": "cancelled",
  "cancellation": {
    "cancelledAt": "2026-10-21T09:30:00.000Z",
    "cancelledBy": "admin",
    "admin": {
      "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
      "fullName": "Kofi Boateng"
    },
    "reason": "Customer called to cancel; courier not yet picked up"
  },
  "refund": {
    "id": "8293a4b5-c6d7-4e8f-90a1-b2c3d4e5f607",
    "status": "pending",
    "amount": "700.00",
    "currency": "GHS"
  },
  "deliveryStatus": "cancelled"
}
```

`refund` is `null` if the order wasn't paid. `deliveryStatus` is `null` if there was no delivery.

Errors:

- 400 `VALIDATION_ERROR`: no reason given.
- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the order is `delivered` (use returns) or already `cancelled`.

Notes:

- Allowed at any status except `delivered` and `cancelled`, including `shipped`.
- The stock goes back on sale.
- If it was paid, a `pending` refund is created. The admin then starts it from Refunds, and the admin page can offer that as the next step.
- Any delivery is marked cancelled. Getting back a parcel a courier already picked up is arranged by phone, outside the system.

## Send for delivery

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/orders/{orderId}/delivery
- Auth: admin

Path parameters:

- `orderId`: the order's ID.

Request body:

- `provider` (string, optional): courier company, for example `"Shaq Express"`.
- `providerReference` (string, optional): the courier's tracking or job number.

Example request:

```json
{
  "provider": "Shaq Express",
  "providerReference": "SHQ-58213"
}
```

Success response: 201 Created

```json
{
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "orderStatus": "shipped",
  "delivery": {
    "id": "d4e5f607-1829-4a3b-8c4d-5e6f708192a3",
    "status": "requested",
    "provider": "Shaq Express",
    "providerReference": "SHQ-58213",
    "requestedAt": "2026-10-21T09:00:00.000Z",
    "deliveredAt": null,
    "failureReason": null
  }
}
```

Errors:

- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the order isn't `processing`, or already has an active delivery.

Notes:

- Creates the delivery and moves the order from `processing` to `shipped` in one step.
- In v1 the admin books the courier by phone and records it here. In v2 this is where the courier's API will be called. The request and response stay the same.

## Update delivery progress

- Method: PATCH
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/orders/{orderId}/delivery
- Auth: admin

Path parameters:

- `orderId`: the order's ID.

Request body:

- `status` (string, required): `assigned`, `picked_up`, `en_route`, `delivered`, `failed`, or `requested` (only to retry after `failed`).
- `provider` (string, optional)
- `providerReference` (string, optional)
- `failureReason` (string): required when `status` is `failed`.

Example request:

```json
{
  "status": "delivered"
}
```

Success response: 200 OK

```json
{
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "orderStatus": "delivered",
  "delivery": {
    "id": "d4e5f607-1829-4a3b-8c4d-5e6f708192a3",
    "status": "delivered",
    "provider": "Shaq Express",
    "providerReference": "SHQ-58213",
    "requestedAt": "2026-10-21T09:00:00.000Z",
    "deliveredAt": "2026-10-22T15:20:00.000Z",
    "failureReason": null
  }
}
```

Example request for a failed delivery:

```json
{
  "status": "failed",
  "failureReason": "Recipient unreachable on 0241234567 after three calls"
}
```

Errors:

- 400 `VALIDATION_ERROR`: for example, `failed` without a `failureReason`.
- 404 `NOT_FOUND`: no order, or no delivery for it yet.
- 409 `INVALID_STATE_TRANSITION`: the move isn't allowed (see notes).

Notes:

- The order of steps is `requested`, `assigned`, `picked_up`, `en_route`, `delivered`. Steps only move forward, but skipping is fine, because a courier may not report every step.
- Any step before `delivered` can go to `failed`. From `failed`, the only move is back to `requested` to try again.
- `delivered` sets the delivery date, moves the order to `delivered`, and starts the 14-day return window.
- `failed` notifies admins. The order stays `shipped` until someone decides to retry or cancel.

## Undo dispatch

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/orders/{orderId}/undo-dispatch
- Auth: admin

Path parameters:

- `orderId`: the order's ID.

Request body:

- `reason` (string, required)

Example request:

```json
{
  "reason": "Courier cancelled the booking before pickup"
}
```

Success response: 200 OK

```json
{
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "orderStatus": "processing",
  "deliveryStatus": "cancelled"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the order isn't `shipped`, or the courier has already picked it up.

Notes:

- For when a courier booking falls through before pickup. The order goes back to `processing`, its delivery is cancelled, and the admin can use Send for delivery again.
- This is the only way an order's status moves backwards.


# Admin: returns

## List returns

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/returns
- Auth: admin

Query parameters (all optional):

- `status`: for example `return_requested`.
- `page`, `limit`: see Pagination.

Success response: 200 OK

```json
{
  "items": [
    {
      "id": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
      "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
      "status": "return_requested",
      "reason": "Too small",
      "customer": {
        "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
        "fullName": "Ama Mensah",
        "email": "ama.mensah@example.com",
        "phone": "0241234567"
      },
      "itemCount": 1,
      "refundAmount": "350.00",
      "requestedAt": "2026-10-24T09:00:00.000Z"
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 1,
  "totalPages": 1
}
```

## Get a return

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/returns/{returnId}
- Auth: admin

Path parameters:

- `returnId`: the return's ID.

Success response: 200 OK

```json
{
  "id": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "status": "return_received",
  "reason": "Too small",
  "rejectionReason": null,
  "customer": {
    "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
    "fullName": "Ama Mensah",
    "email": "ama.mensah@example.com",
    "phone": "0241234567"
  },
  "items": [
    {
      "orderItemId": "0a9b8c7d-6e5f-4a3b-9c2d-1e0f9a8b7c6d",
      "productName": "Linen Kaftan",
      "size": "M",
      "color": "Black",
      "quantity": 1,
      "unitPrice": "350.00"
    }
  ],
  "refundAmount": "350.00",
  "restocked": false,
  "refund": null,
  "requestedAt": "2026-10-24T09:00:00.000Z",
  "decidedAt": "2026-10-24T11:00:00.000Z",
  "decidedBy": {
    "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
    "fullName": "Kofi Boateng"
  },
  "receivedAt": "2026-10-27T13:00:00.000Z",
  "inspectedAt": null,
  "inspectedBy": null
}
```

Errors:

- 404 `NOT_FOUND`

Notes:

- `refundAmount` is the price of the returned items. The delivery fee is never included.

## Approve a return

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/returns/{returnId}/approve
- Auth: admin

Path parameters:

- `returnId`: the return's ID.

Request body: none.

Success response: 200 OK

```json
{
  "id": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
  "status": "return_approved",
  "decidedAt": "2026-10-24T11:00:00.000Z",
  "decidedBy": {
    "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
    "fullName": "Kofi Boateng"
  }
}
```

Errors:

- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the return isn't `return_requested`.

Notes:

- The customer now sends the item back.

## Reject a return

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/returns/{returnId}/reject
- Auth: admin

Path parameters:

- `returnId`: the return's ID.

Request body:

- `rejectionReason` (string, required): shown to the customer.

Example request:

```json
{
  "rejectionReason": "Item shows signs of wear"
}
```

Success response: 200 OK

```json
{
  "id": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
  "status": "return_rejected",
  "rejectionReason": "Item shows signs of wear",
  "decidedAt": "2026-10-24T11:00:00.000Z",
  "decidedBy": {
    "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
    "fullName": "Kofi Boateng"
  }
}
```

Errors:

- 400 `VALIDATION_ERROR`: no reason given.
- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the return isn't `return_requested`.

Notes:

- Final. The order stays `delivered`, and the customer can't open another return for it.

## Mark a return as received

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/returns/{returnId}/receive
- Auth: admin

Path parameters:

- `returnId`: the return's ID.

Request body: none.

Success response: 200 OK

```json
{
  "id": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
  "status": "return_received",
  "receivedAt": "2026-10-27T13:00:00.000Z"
}
```

Errors:

- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the return isn't `return_approved`.

## Record inspection result

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/returns/{returnId}/inspect
- Auth: admin

Path parameters:

- `returnId`: the return's ID.

Request body:

- `result` (string, required): `"pass"` or `"fail"`.
- `rejectionReason` (string): required when `result` is `fail`. Shown to the customer.
- `restock` (boolean, optional): only for `pass`. Put the returned items back into stock. Default true.

Example request (passed):

```json
{
  "result": "pass",
  "restock": true
}
```

Success response (passed): 200 OK

```json
{
  "id": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
  "status": "refund_pending",
  "restocked": true,
  "inspectedAt": "2026-10-27T14:00:00.000Z",
  "inspectedBy": {
    "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
    "fullName": "Kofi Boateng"
  },
  "refund": {
    "id": "8293a4b5-c6d7-4e8f-90a1-b2c3d4e5f607",
    "status": "pending",
    "amount": "350.00",
    "currency": "GHS"
  }
}
```

Example request (failed):

```json
{
  "result": "fail",
  "rejectionReason": "Stain on the front panel not present at dispatch"
}
```

Success response (failed): 200 OK

```json
{
  "id": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
  "status": "return_rejected_on_inspection",
  "rejectionReason": "Stain on the front panel not present at dispatch",
  "inspectedAt": "2026-10-27T14:00:00.000Z",
  "inspectedBy": {
    "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
    "fullName": "Kofi Boateng"
  },
  "refund": null
}
```

Errors:

- 400 `VALIDATION_ERROR`: `fail` without a reason.
- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the return isn't `return_received`.

Notes:

- On pass, a `pending` refund is created for the returned items. No money moves until an admin starts it in Refunds. Starting it moves the return to `refund_initiated`.
- On fail, the return is closed with no refund.


# Admin: refunds

How refunds work:

- A refund is created as `pending` when a paid order is cancelled, a return passes inspection, or a late payment can't be honoured. No money moves at this point.
- An admin always starts the money moving. This is how "refunds are always started by an admin" is enforced.
- Paystack payments: `pending` → `initiated` → `completed`. If Paystack fails it: `failed`, and the admin can start it again.
- Manual MoMo payments: Paystack can't refund money it never collected. The admin sends the money back by MoMo, then records it. `pending` → `completed`.

## List refunds

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/refunds
- Auth: admin

Query parameters (all optional):

- `status`: `pending`, `initiated`, `completed` or `failed`.
- `page`, `limit`: see Pagination.

Success response: 200 OK

```json
{
  "items": [
    {
      "id": "8293a4b5-c6d7-4e8f-90a1-b2c3d4e5f607",
      "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
      "returnId": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
      "reason": "return",
      "status": "pending",
      "amount": "350.00",
      "currency": "GHS",
      "paymentProvider": "paystack",
      "customer": {
        "id": "5d2c8f1e-7a3b-4c9d-8e6f-1a2b3c4d5e6f",
        "fullName": "Ama Mensah"
      },
      "createdAt": "2026-10-27T14:00:00.000Z"
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 1,
  "totalPages": 1
}
```

Notes:

- `reason` is `cancellation`, `return` or `late_payment`.

## Get a refund

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/refunds/{refundId}
- Auth: admin

Path parameters:

- `refundId`: the refund's ID.

Success response: 200 OK

```json
{
  "id": "8293a4b5-c6d7-4e8f-90a1-b2c3d4e5f607",
  "orderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
  "returnId": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
  "reason": "return",
  "status": "failed",
  "amount": "350.00",
  "currency": "GHS",
  "payment": {
    "id": "4b5c6d7e-8f90-4a1b-8c2d-3e4f5a6b7c8d",
    "provider": "paystack",
    "providerReference": "ord_a7d9e3f1_1729418400",
    "amount": "700.00"
  },
  "paystackRefundReference": "rfnd_3920193",
  "manualReference": null,
  "failureReason": "Customer's bank declined the reversal",
  "initiatedBy": {
    "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
    "fullName": "Kofi Boateng"
  },
  "createdAt": "2026-10-27T14:00:00.000Z",
  "initiatedAt": "2026-10-27T14:05:00.000Z",
  "completedAt": null
}
```

Errors:

- 404 `NOT_FOUND`

## Start a Paystack refund

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/refunds/{refundId}/initiate
- Auth: admin

Path parameters:

- `refundId`: the refund's ID.

Request body: none.

Success response: 200 OK

```json
{
  "id": "8293a4b5-c6d7-4e8f-90a1-b2c3d4e5f607",
  "status": "initiated",
  "amount": "350.00",
  "currency": "GHS",
  "paystackRefundReference": "rfnd_3920193",
  "initiatedBy": {
    "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
    "fullName": "Kofi Boateng"
  },
  "initiatedAt": "2026-10-27T14:05:00.000Z"
}
```

Errors:

- 400 `WRONG_PAYMENT_METHOD`: the original payment was manual MoMo. Use Record a manual MoMo refund.
- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the refund isn't `pending` or `failed`.
- 502 `PAYMENT_PROVIDER_ERROR`
- 503 `PAYMENT_PROVIDER_UNAVAILABLE`

Notes:

- Sends the refund to Paystack against the original payment.
- If it came from a return, the return moves to `refund_initiated`.
- Paystack confirms completion through the webhook (`refund.processed`), which sets the refund to `completed` and the return to `refunded`.
- If Paystack errors here, nothing changes and it's safe to try again.

## Record a manual MoMo refund

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/refunds/{refundId}/record-manual
- Auth: admin

Path parameters:

- `refundId`: the refund's ID.

Request body:

- `momoReference` (string, required): the MoMo transaction ID of the money sent back.
- `note` (string, optional)

Example request:

```json
{
  "momoReference": "MP241028.0930.R55821",
  "note": "Sent to 0241234567"
}
```

Success response: 200 OK

```json
{
  "id": "8293a4b5-c6d7-4e8f-90a1-b2c3d4e5f607",
  "status": "completed",
  "amount": "350.00",
  "currency": "GHS",
  "manualReference": "MP241028.0930.R55821",
  "initiatedBy": {
    "id": "9f8e7d6c-5b4a-4392-8170-6f5e4d3c2b1a",
    "fullName": "Kofi Boateng"
  },
  "initiatedAt": "2026-10-28T09:30:00.000Z",
  "completedAt": "2026-10-28T09:30:00.000Z"
}
```

Errors:

- 400 `VALIDATION_ERROR`
- 400 `WRONG_PAYMENT_METHOD`: the original payment was Paystack. Use Start a Paystack refund.
- 404 `NOT_FOUND`
- 409 `INVALID_STATE_TRANSITION`: the refund isn't `pending`.

Notes:

- Send the money first, then record it here.
- If it came from a return, the return becomes `refunded`.


# Admin: notifications

Types: `new_order`, `return_requested`, `delivery_failed`, `refund_pending`, `refund_failed`, `payment_issue`.

## List notifications

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/notifications
- Auth: admin

Query parameters (all optional):

- `unreadOnly`: `true` for unread only.
- `page`, `limit`: see Pagination.

Success response: 200 OK

```json
{
  "items": [
    {
      "id": "f6071829-3a4b-4c5d-8e6f-708192a3b4c5",
      "type": "return_requested",
      "message": "Ama Mensah requested a return on order a7d9e3f1",
      "relatedOrderId": "a7d9e3f1-2b4c-4d6e-8f0a-1b2c3d4e5f60",
      "relatedReturnId": "6e7f8091-a2b3-4c4d-9e5f-60718293a4b5",
      "relatedRefundId": null,
      "isRead": false,
      "createdAt": "2026-10-24T09:00:00.000Z"
    }
  ],
  "unreadCount": 4,
  "page": 1,
  "limit": 20,
  "total": 12,
  "totalPages": 1
}
```

Notes:

- `unreadCount` is for the badge on the notification bell.

## Mark a notification as read

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/notifications/{notificationId}/read
- Auth: admin

Path parameters:

- `notificationId`: the notification's ID.

Request body: none.

Success response: 200 OK

```json
{
  "id": "f6071829-3a4b-4c5d-8e6f-708192a3b4c5",
  "isRead": true
}
```

Errors:

- 404 `NOT_FOUND`

## Mark all notifications as read

- Method: POST
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/notifications/read-all
- Auth: admin

Request body: none.

Success response: 200 OK

```json
{
  "updated": 4
}
```


# Admin: dashboard

## Sales summary

- Method: GET
- URL: https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/dashboard
- Auth: admin

Query parameters (all optional):

- `from`, `to`: dates, for example `2026-10-01`. Default is the last 30 days.

Example request:

```
GET https://e-commerce-web-psi-sage.vercel.app/api/v1/admin/dashboard?from=2026-10-01&to=2026-10-31
```

Success response: 200 OK

```json
{
  "range": {
    "from": "2026-10-01",
    "to": "2026-10-31"
  },
  "totalOrders": 120,
  "paidOrders": 104,
  "grossRevenue": "45210.00",
  "refundedAmount": "1300.00",
  "netRevenue": "43910.00",
  "currency": "GHS",
  "ordersByStatus": {
    "pending": 3,
    "paid": 0,
    "processing": 8,
    "shipped": 5,
    "delivered": 88,
    "cancelled": 16
  },
  "lowStockCount": 7,
  "pendingReturns": 2,
  "pendingRefunds": 1
}
```

Errors:

- 400 `VALIDATION_ERROR`: bad dates, or `from` after `to`.

Notes:

- Revenue counts only orders with a successful payment. An unpaid `pending` order isn't revenue.
- `netRevenue` is `grossRevenue` minus completed refunds.
- `pendingReturns` counts returns waiting on an admin (requested or received). `pendingRefunds` counts refunds still `pending` or `failed`.