# New Haven Restaurant — Online Ordering & Delivery Platform

A production-oriented restaurant ordering, delivery, payment, and administration
platform, built with Next.js (App Router), PostgreSQL/Prisma, and Stripe.

This is real, working full-stack code — every price is computed server-side,
every payment goes through Stripe's actual API, every order persists to a
real database, and every admin action is authorized server-side. There is no
mock data path for checkout.

## Contents

- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Getting started](#getting-started)
- [Environment variables](#environment-variables)
- [Database setup](#database-setup)
- [Stripe setup](#stripe-setup)
- [Cloudinary setup](#cloudinary-setup)
- [Resend setup](#resend-setup)
- [Google Maps setup](#google-maps-setup)
- [Twilio / WhatsApp setup (optional)](#twilio--whatsapp-setup-optional)
- [Local development](#local-development)
- [Testing](#testing)
- [Demo credentials](#demo-credentials)
- [Production deployment](#production-deployment)
- [Security considerations](#security-considerations)

## Architecture

```
src/
  app/
    (site)/            customer-facing pages (menu, cart, checkout, orders, account)
    admin/              admin dashboard (orders, menu, kitchen display, settings)
    api/                Next.js Route Handlers — see full list below
  components/           shared UI (header, footer, providers)
  features/              feature-scoped UI (menu cards, etc.)
  lib/
    auth/               password hashing, JWT + refresh tokens, session/RBAC
    services/           business logic: pricing, orders, promos, delivery,
                        opening hours, Stripe, Cloudinary, notifications
    validation/         Zod schemas, shared by API routes and forms
    prisma.js           Prisma client singleton
    env.js              startup environment validation (Zod)
    api-error.js        consistent API error envelope
  stores/                Zustand client state (cart only — never prices)
  emails/                HTML email templates
prisma/
  schema.prisma          full relational schema
  seed.js                 realistic seed data (KES pricing)
tests/                    Vitest unit tests for business logic
```

**The pricing engine (`src/lib/services/pricing.js`) is the single source of
truth for money.** The client cart only ever sends item ids, quantities, and
chosen modifier ids — it never sends prices. Every price, modifier
validation, delivery fee, promo discount, and tax calculation is re-derived
from the database on the server, both when previewing the cart
(`POST /api/cart`) and again when the order is actually created
(`POST /api/orders`). The client-computed total shown in the UI is a courtesy
preview; nothing from it is trusted.

### API routes

```
/api/auth/register            /api/auth/login          /api/auth/refresh
/api/auth/logout              /api/auth/forgot-password /api/auth/reset-password
/api/auth/verify-email

/api/categories                /api/menu               /api/menu/[slug]
/api/cart                      (server-side pricing preview)

/api/orders                    /api/orders/[id]         /api/orders/[id]/status
/api/orders/[id]/reorder

/api/payments/create           /api/payments/webhook    /api/payments/refund

/api/delivery/zones            /api/delivery/calculate
/api/promos/validate
/api/reviews
/api/restaurant-info
/api/account/profile           /api/account/addresses   /api/account/addresses/[id]

/api/admin/users               /api/admin/menu           /api/admin/menu/[id]
/api/admin/menu/upload-image   /api/admin/categories     /api/admin/categories/[id]
/api/admin/modifier-groups     /api/admin/modifier-groups/[id]
/api/admin/orders              /api/admin/analytics
/api/admin/delivery-zones      /api/admin/delivery-zones/[id]
/api/admin/promos              /api/admin/promos/[id]
/api/admin/reviews             /api/admin/reviews/[id]
/api/admin/settings
```

## Tech stack

React 19 · Next.js (App Router) · JavaScript · Tailwind CSS · Framer Motion ·
Lucide React · React Hook Form · Zod · TanStack Query · Zustand · Prisma ·
PostgreSQL · Stripe · Cloudinary · Resend · Twilio (optional) · WhatsApp
Cloud API (optional) · Argon2 · JWT

## Getting started

```bash
git clone <this-repo>
cd new-haven-restaurant
npm install
cp .env.example .env   # then fill in real values — see below
npx prisma migrate dev --name init
npm run prisma:seed
npm run dev
```

Visit `http://localhost:3000`. Admin dashboard is at `/admin` (sign in with
the seeded admin account first).

## Environment variables

See `.env.example` for the full list with comments. All server variables are
validated with Zod at startup (`src/lib/env.js`) — the app will refuse to
boot in production with a missing/invalid required variable, and will log a
clear error in development.

Required to run anything: `DATABASE_URL`, `JWT_ACCESS_SECRET`,
`JWT_REFRESH_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`.

Optional (features degrade gracefully if omitted): Cloudinary (image
uploads), Resend (emails), Google Maps (address autocomplete/geocoding),
Twilio and WhatsApp (SMS/WhatsApp notifications — the `NotificationService`
abstraction in `src/lib/services/notifications/` silently skips any channel
that isn't configured, recording why in the `Notification` table).

## Database setup

1. Provision a PostgreSQL database (local Postgres, [Supabase](https://supabase.com),
   or [Neon](https://neon.tech) all work).
2. Set `DATABASE_URL` in `.env`.
3. Run migrations:
   ```bash
   npx prisma migrate dev --name init
   ```
4. Seed realistic sample data:
   ```bash
   npm run prisma:seed
   ```

## Stripe setup

1. Create a Stripe account, grab your test-mode secret/publishable keys from
   the Dashboard, and set `STRIPE_SECRET_KEY` / `STRIPE_PUBLISHABLE_KEY` /
   `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`.
2. **Webhook (required for orders to ever mark as paid):**
   ```bash
   stripe listen --forward-to localhost:3000/api/payments/webhook
   ```
   Copy the `whsec_...` value it prints into `STRIPE_WEBHOOK_SECRET`.
3. In production, add a webhook endpoint in the Stripe Dashboard pointing to
   `https://yourdomain.com/api/payments/webhook`, subscribed to at least:
   `checkout.session.completed`, `checkout.session.expired`,
   `payment_intent.payment_failed`, `charge.refunded`.
4. The webhook handler (`src/app/api/payments/webhook/route.js`) is
   idempotent: every processed Stripe event id is recorded in
   `StripeWebhookEvent`, so retried or duplicate deliveries are safely
   ignored rather than double-applying an order/payment state change.

## Cloudinary setup

Create a free Cloudinary account, then set `CLOUDINARY_CLOUD_NAME`,
`CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`. Menu image uploads go through
`POST /api/admin/menu/upload-image` (manager/admin only), which validates
file type and size before uploading.

## Resend setup

Create a Resend account and a verified sending domain, then set
`RESEND_API_KEY` and `RESEND_FROM_EMAIL`. Without these, the app still runs —
emails are recorded as `FAILED` with reason "EMAIL channel is not configured"
in the `Notification` table instead of throwing.

## Google Maps setup

Set `GOOGLE_MAPS_API_KEY` (enable the Places and Geocoding APIs) to power
address autocomplete at checkout. Delivery-zone matching
(`POST /api/delivery/calculate`) currently uses straight-line (Haversine)
distance from the restaurant's configured coordinates to pick a zone; this is
a deliberate, working baseline that can be swapped for the Distance Matrix
API for road-distance/ETA accuracy without changing the endpoint's contract.

## Twilio / WhatsApp setup (optional)

Set `TWILIO_ACCOUNT_SID` / `TWILIO_AUTH_TOKEN` / `TWILIO_PHONE_NUMBER` for
SMS, and `WHATSAPP_ACCESS_TOKEN` / `WHATSAPP_PHONE_NUMBER_ID` for WhatsApp
(via the WhatsApp Cloud API). Both are entirely optional — order status
updates are always emailed, and SMS/WhatsApp are attempted best-effort
alongside email whenever configured.

## Local development

```bash
npm run dev              # start the dev server
npx prisma studio         # browse/edit the database visually
npm run lint
```

## Testing

```bash
npm test
```

The suite (Vitest) covers the business logic that matters most to get right:
password hashing, JWT/refresh-token handling, the pricing engine (modifier
validation, delivery minimums, tax), promo code rules (expiry, usage limits,
discount caps), the order status state machine, and checkout validation
schemas. Prisma is mocked in these tests so they run without a live
database. For endpoint-level and payment-flow tests (Stripe test-mode
payments, webhook delivery, duplicate-order prevention), run against a real
seeded database and the Stripe CLI's `stripe trigger` commands.

## Demo credentials

Created by `npm run prisma:seed` — **development only, change or remove
before deploying to production**:

| Role     | Email                                | Password       |
|----------|---------------------------------------|----------------|
| Admin    | admin@newhavenrestaurant.co.ke        | Admin@12345    |
| Staff    | staff@newhavenrestaurant.co.ke        | Staff@12345    |
| Customer | customer@example.com                  | Customer@123   |

## Production deployment

**App — Vercel:**
1. Import the repo, set every variable from `.env.example` in the Vercel
   project settings (Production + Preview as appropriate).
2. `npx prisma migrate deploy` runs your pending migrations against the
   production database — run this manually or as part of your CI/CD step
   before/during deploy (Vercel build steps don't run long-lived migration
   commands well; a dedicated deploy step or GitHub Action is recommended).
3. Set the Stripe webhook endpoint to your production domain (see above).
4. No `localhost` URLs are hard-coded anywhere — `NEXT_PUBLIC_APP_URL` drives
   every absolute link (emails, Stripe redirect URLs).

**Database:** Supabase or Neon Postgres both work out of the box with the
connection string format in `.env.example`.

**Do not commit `.env`** — it's already in `.gitignore`. Rotate the demo
seed credentials (or delete those seeded users) before going live.

## Security considerations

- Passwords hashed with Argon2id; never logged or returned by any endpoint.
- Access tokens are short-lived JWTs (15 min); refresh tokens are opaque
  random strings, stored only as a SHA-256 hash, with rotation on every
  refresh and automatic full-session revocation if a revoked/expired token
  is replayed (a signal of token theft).
- Every admin/staff API route calls `requireRole(...)` server-side —
  client-side route guards in `/admin` are UX only, never the actual
  authorization boundary.
- All money math (prices, modifiers, delivery fees, tax, discounts, totals)
  is computed server-side from the database on every request that matters;
  client-supplied prices/totals are never trusted.
- Stripe webhook signature is verified on every request
  (`stripe.webhooks.constructEvent`); processing is idempotent via the
  `StripeWebhookEvent` table.
- Order creation is idempotent via a client-generated `idempotencyKey`,
  preventing duplicate orders from a refreshed checkout page.
- Zod validates every API input; error responses never leak stack traces,
  database error text, or other internals — see `src/lib/api-error.js`.
- Basic in-memory rate limiting is applied to auth and order-creation
  endpoints (`src/lib/rate-limit.js`). This is process-local memory, which
  is fine for a single long-running server; for a multi-instance serverless
  deployment (e.g. Vercel), back it with a shared store (Upstash Redis, etc.)
  before relying on it against real credential-stuffing traffic.
- Menu image uploads validate MIME type and size before ever reaching
  Cloudinary.
