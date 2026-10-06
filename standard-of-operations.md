# Standard of Operations

Team size: 3. This document defines how the team works day-to-day — git workflow, commit conventions, code review, testing, file structure, and environment/secrets handling.

---

## 1. Git Workflow

### 1.1 Branching model

- **No `development`/`staging` branch.** Feature branches are created directly off `main` and merged directly back into `main` via PR.
- `main` has branch protection: no direct pushes, CI (lint, typecheck, build) must pass, and **at least 1 approval** is required before merge (see §3).

### 1.2 Branch naming

Enforced prefix convention:

| Prefix | Use for |
|---|---|
| `feature/` | New functionality (e.g. `feature/returns-api`) |
| `fix/` | Bug fixes (e.g. `fix/cart-upsert-duplicate`) |
| `chore/` | Tooling, config, dependency updates, non-functional cleanup |
| `docs/` | Documentation-only changes |

Branch names are `kebab-case` after the prefix (e.g. `feature/admin-return-dashboard`, not `feature/AdminReturnDashboard`).

### 1.3 Pull requests

- One logical change per PR — one feature, one bug fix, or one focused refactor. **Rule of thumb:** if the PR description needs more than a few bullet points to explain what changed, it's probably two PRs.
- PR description should state what changed and why, and call out anything a reviewer should pay specific attention to (e.g. "touches the checkout transaction — please check the rollback logic").
- **Requires 1 approval** from another team member before merge, in addition to CI passing.
- Author does not merge their own PR without that approval, even if CI is green.

---

## 2. Commit Conventions

**Conventional Commits** format, enforced:

```
<type>(<optional scope>): <short description>

[optional body]
```

**Types:** `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`

**Examples:**
```
feat(orders): add return request endpoint
fix(cart): prevent duplicate line items on upsert
chore(deps): add @upstash/ratelimit
docs(api): document returns and refunds endpoints
```

Commit messages should describe *what* changed, not narrate the debugging process (e.g. `fix(auth): correct rate_limit column type to bigint`, not `fix: finally got this working after 3 hours`).

---

## 3. Code Review

- Minimum **1 approval** required before merge — any of the other 2 team members can approve.
- Reviewer responsibility: verify the change does what the PR description claims, check for obvious correctness issues (especially around money/inventory-touching code — the checkout transaction, payment webhook, and refund logic deserve extra scrutiny given how much of this project's real bugs lived in exactly that kind of code), and confirm tests exist for new logic (see §4).
- Reviews should be turned around promptly given the 3-person team size — a PR sitting unreviewed for days blocks real progress on a small team more than it would on a larger one.

---

## 4. Testing Requirements

Automated tests are **required** going forward — manual Postman testing (as used to validate the initial backend build) is no longer sufficient as the sole verification method for new work.

### 4.1 What needs tests

- **Every new API route** — at minimum: the happy path, the primary error case(s) (auth failure, validation failure, not-found), and any business-rule edge case specific to that route (e.g. insufficient stock, return outside the 14-day window).
- **Any transaction-wrapped logic** (checkout, cancellation with inventory restoration, refund triggering) — must include a test that verifies rollback behavior on failure, not just the success path. This is directly informed by how the original checkout transaction was validated during manual testing (deliberately triggering insufficient-stock failures and confirming inventory/cart were untouched) — that same rigor should exist as an automated test, not rely on someone remembering to re-check it by hand.
- **Webhook handlers** (Paystack) — must include a test for idempotency (the same event delivered twice should not double-process).

### 4.2 What doesn't strictly need tests

- Simple CRUD reads with no business logic (e.g. `GET /api/categories`) can be lower priority for dedicated tests, though still covered incidentally by integration tests of the flows that depend on them.

### 4.3 CI enforcement

`.github/workflows/ci.yml` should run the test suite alongside lint/typecheck/build once tests exist — a PR should not be mergeable if it introduces new routes/logic with zero test coverage. (Enforcing this automatically via a coverage threshold is a reasonable future addition once a baseline test suite exists — not a blocker for starting to write tests now.)

---

## 5. File Structure & Naming Conventions

### 5.1 Naming

- **`camelCase`** for file names throughout the codebase (e.g. `rateLimit.ts`, not `rate-limit.ts` or `RateLimit.ts`), except where a framework convention overrides it (e.g. Next.js's own required file names like `route.ts`, `page.tsx`, and its bracketed dynamic segments like `[orderId]`, which follow Next.js's own convention regardless).
- Component files: `PascalCase` for the component itself (e.g. `ProductCard.tsx`), matching the exported component name.

### 5.2 Monorepo layout

Follows the structure established in the Software Architecture document — `apps/web`, `apps/mobile` (v2), `packages/db`, `packages/auth`, `packages/config`. New shared logic that both `apps/web` and the future `apps/mobile` will need (e.g. shared TypeScript types for API request/response shapes) should go in a new `packages/types` (or similar) rather than being duplicated once mobile development starts.

### 5.3 Component organization (storefront/admin UI, once built)

Co-located structure per component — a component's file, and anything specific only to it, live together rather than split across parallel `components/`, `styles/`, `tests/` trees:
```
components/
  ProductCard/
    ProductCard.tsx
    ProductCard.test.tsx
```

---

## 6. Environment Variables & Secrets — Checklist

Directly informed by real friction hit during this project (stale/missing Vercel env vars causing production 500s, `pnpm-lock.yaml` sync issues, confusion between `packages/db/.env` and `apps/web/.env.local` being separate files read by separate processes). Follow this checklist whenever adding or changing an environment variable:

1. **Identify which process needs it.** `packages/db/.env` is read only by `drizzle-kit` commands run from that folder. `apps/web/.env.local` is read by the running Next.js app. A variable needed by both (e.g. `DATABASE_URL`) must be added to **both files**, independently — one does not inherit from the other.
2. **Add it locally first**, confirm the app/command actually works with it before touching anything else.
3. **Add it to Vercel — all relevant environments.** Project Settings → Environment Variables (note: this may be nested under an "Environments" page depending on Vercel's current UI). Add to **Production** and **Preview** at minimum. "Development" in Vercel's env var scoping only applies to `vercel dev` (the Vercel CLI's local emulation) — irrelevant if the team runs `pnpm dev` directly, which is the case here.
4. **Use "Secret" type for anything sensitive** (API keys, connection strings, signing secrets) — not "Config," which remains visible in the dashboard after saving. Use "Config" only for genuinely non-sensitive values.
5. **Never commit `.env` or `.env.local` files.** Confirm they're in `.gitignore` before your first commit on a fresh clone — don't assume it's already covered.
6. **Redeploy after adding/changing a Vercel env var.** Existing deployments do not retroactively pick up new environment variables — either trigger a redeploy from the dashboard or push a new commit.
7. **If using a package manager other than the team standard** (e.g. `pnpm` locally while the team's onboarding docs reference `bun`), add your own lockfile to your **personal** git exclude list (`.git/info/exclude`, not the shared `.gitignore`) rather than committing a second, conflicting lockfile format.

---

## 7. Coding Constraints

- **TypeScript strict mode** — no implicit `any`, consistent with what's already configured in `packages/config`.
- **Zod for all request body validation** on API routes — this is already the established pattern (used in the product, inventory, and order-status admin routes) and should be followed for every new route, not just admin ones.
- **Every DB write that touches money or inventory must be wrapped in a transaction** if it involves more than one table write — this is a hard rule, not a suggestion, given that the original checkout logic bug (missing inventory check) and the transaction-rollback behavior were both central to this project's correctness.
- **No raw SQL string concatenation** — all queries go through Drizzle's query builder, which parameterizes inputs and avoids SQL injection risk by construction.
- **Rate limiting check is the first operation in any new write-heavy or auth-adjacent route**, before any DB reads — matches the established pattern in the existing orders/admin routes.
