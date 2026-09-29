# ShopSphere — Full-stack E-commerce Platform

A production-oriented e-commerce application: a React storefront and admin panel on top of an Express/MongoDB REST API, with server-side pricing, atomic inventory reservation, pluggable payments (Razorpay, Stripe, Cash on Delivery), pluggable image storage (local, S3, Cloudinary), email notifications and Docker deployment.

---

## Contents

1. [Features](#features)
2. [Architecture](#architecture)
3. [Tech stack](#tech-stack)
4. [Folder structure](#folder-structure)
5. [Getting started (local)](#getting-started-local)
6. [Environment variables](#environment-variables)
7. [Database & seed data](#database--seed-data)
8. [Running with Docker](#running-with-docker)
9. [Payments](#payments)
10. [API reference](#api-reference)
11. [Testing](#testing)
12. [Security](#security)
13. [Performance](#performance)
14. [Deployment](#deployment)
15. [Known limitations](#known-limitations)

---

## Features

**Customers**

- Browse, search (full-text with prefix fallback and debounced suggestions), filter (category, brand, price, rating, availability, discount, attributes and variant options) and sort (newest, price, rating, popularity, discount) with URL-driven, shareable state and pagination
- Product pages with variant selection (e.g. *256GB / Blue*), per-variant price, stock and images, specifications, reviews and related products
- Cart for guests (stored locally, priced by the server) and signed-in users (persistent); the guest cart merges into the account at sign-in
- Wishlist, with move-to-cart
- Coupons (percentage/fixed, minimum order, maximum discount, date window, global and per-user limits, category- or product-specific)
- Four-step checkout: address → delivery → payment → review, with the summary always computed by the server
- Online payment through Razorpay or Stripe Checkout, or cash on delivery; payment results are verified server-side and by signed webhooks
- Order history, order tracking timeline, cancellation, returns within 7 days and reviews of delivered purchases
- Registration, login, logout, email verification, forgot/reset password, change password, profile and avatar

**Administrators** (`/admin`)

- Dashboard: total/today/monthly revenue, order counts, customers, low stock, plus charts for revenue, orders, sales by category, top products and customer growth (each chart has a table view)
- Products: CRUD, bulk delete, activate/deactivate, multiple images, variants, specifications, filterable attributes, SEO fields, search/filter/sort/pagination
- Categories (two levels) and brands with images and SEO; inventory (available/reserved/sold, low and out of stock, stock adjustments)
- Orders with a validated status workflow, tracking numbers, cancellation and refunds; customers and roles; coupons; review moderation; homepage banners; store settings (tax, shipping, homepage sections, review approval)

---

## Architecture

```
Browser ── React SPA (Vite) ──► /api ──► Express app
            RTK Query + Axios            routes → validators (Zod) → controllers → services → models (Mongoose)
            access token in memory       │                                        │
            refresh token: HTTP-only     ├── Redis (cache, rate limits; optional) ├── MongoDB
            cookie                       ├── Storage provider: LOCAL | S3 | CLOUDINARY
                                         ├── Payment providers: Razorpay | Stripe | COD
                                         └── Nodemailer (SMTP, or logged in development)
```

**Key design decisions**

- **Thin controllers, fat services.** Controllers only translate HTTP to service calls; all business rules live in `server/src/services`.
- **The server is the only source of prices.** Carts store product/variant IDs and quantities. `pricingService` resolves live prices, `couponService` evaluates discounts, and totals (tax, shipping, discount) are recomputed at order time. Client-supplied amounts are ignored, because unknown fields are stripped by validation.
- **Inventory without transactions.** Stock moves between `stock → reserved → sold` buckets with atomic conditional updates (`$inc` guarded by `stock >= qty`), so concurrent checkouts cannot oversell. It works on a standalone MongoDB and needs no replica set. Online orders reserve stock; payment confirmation commits it; cancellation or payment expiry (a background job, default 30 minutes) releases it. COD orders commit immediately.
- **Idempotent payment confirmation.** Both the client verify call and the gateway webhook go through the same compare-and-set transition (`PENDING → CONFIRMED` only once), so duplicate or concurrent notifications never double-commit stock. A payment that arrives after an order expired is refunded automatically.
- **Pluggable providers.** Payments (`services/payment/*Provider.js`) and storage (`services/storage/*`) implement small interfaces and are selected with environment variables. New gateways register through `paymentService.registerProvider()`.
- **Refresh-token rotation with reuse detection.** Refresh tokens are stored hashed, rotate on every use, and replaying an old token revokes the whole token family.

---

## Tech stack

| Layer | Technologies |
|---|---|
| Frontend | React 19, Vite, React Router 7, Redux Toolkit + RTK Query (Axios base query), Tailwind CSS 4, React Hook Form + Zod, Recharts, Sonner, Lucide icons |
| Backend | Node.js 22, Express 4, MongoDB + Mongoose 9, Zod, JWT, bcrypt (via `bcryptjs`), Helmet, CORS, express-rate-limit (+ Redis store), Multer, Sharp, Nodemailer, Pino, ioredis |
| Payments | Razorpay SDK, Stripe SDK (Checkout Sessions) |
| Storage | Local disk, AWS S3 (`@aws-sdk/client-s3`), Cloudinary |
| Testing | Jest + Supertest + mongodb-memory-server (API); Vitest + React Testing Library (client) |
| Ops | Docker, docker compose, nginx |

> `bcryptjs` is a pure-JavaScript implementation of the same bcrypt algorithm (cost 12), chosen to avoid native build toolchains in Docker and CI.

---

## Folder structure

```
ecommerce/
├── client/
│   ├── src/
│   │   ├── assets/             static assets
│   │   ├── components/         ui/ (design system), layout/, product/, cart/, order/, admin/, forms/, common/
│   │   ├── constants/          labels, status maps, sort options
│   │   ├── features/           RTK slices + API endpoints per domain (auth, catalog, cart, wishlist, account, checkout, admin)
│   │   ├── hooks/              useCart, useWishlist, usePayment, useProductFilters, useVariantSelection, useDebounce
│   │   ├── layouts/            Main, Account, Admin layouts
│   │   ├── pages/              storefront, auth/, account/, admin/
│   │   ├── routes/             lazy route table + auth/admin guards
│   │   ├── services/           Axios client (auth + refresh interceptors), RTK Query base API
│   │   ├── store/              store setup, persistence
│   │   ├── test/               test setup and helpers
│   │   ├── utils/              formatting and helpers
│   │   ├── App.jsx, main.jsx
│   ├── Dockerfile, nginx.conf, vite.config.js, .env.example
├── server/
│   ├── src/
│   │   ├── config/             env (validated), db, redis, logger
│   │   ├── constants/          roles, statuses, transitions, sorts
│   │   ├── controllers/        HTTP adapters
│   │   ├── jobs/               payment-expiry job
│   │   ├── middleware/         auth, validation, sanitisation, rate limits, uploads, errors
│   │   ├── models/             Mongoose schemas + indexes
│   │   ├── routes/             routers per domain
│   │   ├── seed/               seed script + demo catalogue
│   │   ├── services/           business logic (pricing, orders, inventory, payment/, storage/, email/, …)
│   │   ├── uploads/            local storage root (served at /uploads)
│   │   ├── utils/              ApiError, helpers
│   │   ├── validators/         Zod schemas
│   │   ├── app.js, server.js
│   ├── scripts/memoryDb.js     throwaway MongoDB for local development
│   ├── tests/                  Jest + Supertest suites
│   ├── Dockerfile, .env.example
├── docker-compose.yml, .env.example, .gitignore, README.md
```

---

## Getting started (local)

**Prerequisites:** Node.js 20+ (22 recommended) and MongoDB 6+. Redis is optional. No MongoDB installed? Use `npm run db:memory` (see below).

```bash
# 1. API
cd server
npm install
cp .env.example .env          # set JWT_ACCESS_SECRET / JWT_REFRESH_SECRET (any value works in development)
npm run seed                  # demo data (wipes the database)
npm run dev                   # http://localhost:5000  (health: /api/health)

# 2. Client (new terminal)
cd client
npm install
cp .env.example .env
npm run dev                   # http://localhost:5173  (proxies /api and /uploads to :5000)
```

**Without a local MongoDB:** in a separate terminal run `cd server && npm run db:memory`. This starts an in-memory MongoDB on port 27017; data is lost when it stops. Then run `npm run seed` and `npm run dev` as above.

**Demo accounts** (created by the seed):

| Role | Email | Password |
|---|---|---|
| Super admin | `superadmin@shopsphere.dev` | `Admin@12345` |
| Admin | `admin@shopsphere.dev` | `Admin@12345` |
| Customer | `aarav@example.com` (also `priya@`, `rohan@`, `sara@`) | `User@12345` |

**Demo coupons:** `WELCOME10` (10%, max ₹1,000), `FLAT500` (₹500 off over ₹4,999), `TECH15` (15% off electronics), `FIT20` (yoga mats only), `SUMMER25` (expired).

---

## Environment variables

All server variables are validated at startup (`server/src/config/env.js`). In production the server **refuses to start** with missing or weak JWT secrets. Full annotated list: [`server/.env.example`](server/.env.example).

| Variable | Purpose |
|---|---|
| `PORT`, `NODE_ENV`, `LOG_LEVEL`, `APP_TIMEZONE` | Runtime basics (`APP_TIMEZONE` drives "today"/"this month" analytics) |
| `MONGO_URI` | MongoDB connection string |
| `REDIS_URL` | Optional: shared cache and rate-limit store (falls back to in-process) |
| `CLIENT_URL` | Allowed CORS origin(s), comma-separated; the first is used in email links |
| `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `JWT_ACCESS_EXPIRES`, `JWT_REFRESH_EXPIRES` | Token signing (use 48+ random bytes each) |
| `COOKIE_SECURE`, `TRUST_PROXY` | Cookie `Secure` flag (defaults to on in production); number of proxies in front |
| `CURRENCY`, `PAYMENT_PROVIDERS`, `PAYMENT_PENDING_TIMEOUT_MINUTES` | Currency, enabled payment methods, unpaid-order expiry |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET` | Razorpay |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` | Stripe |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASSWORD`, `EMAIL_FROM` | Email. With no `SMTP_HOST`, emails are rendered and logged instead of sent |
| `STORAGE_PROVIDER`, `MAX_UPLOAD_SIZE_MB` | `LOCAL`, `S3` or `CLOUDINARY` |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_BUCKET_NAME`, `AWS_PUBLIC_URL` | S3 (the bucket or CDN must allow public reads of uploaded objects) |
| `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` | Cloudinary |

Client: `VITE_API_URL` (default `/api`) and `VITE_SITE_URL` (canonical/Open Graph URLs). **No secrets belong in the client.** Only the Razorpay key ID, which is public by design, reaches the browser, and it comes from `GET /api/payments/config`.

---

## Database & seed data

- Indexes are declared on the models: product slug, SKU, variant SKU, a weighted full-text index (name, SKU, brand, category, tags, description), and compound indexes for listing filters and sorts; orders by user, number, status and date; reviews by product+user (unique) and status. In development, Mongoose builds them automatically; in production (`autoIndex` off), run the seed once or call `syncIndexes()` during deployment.
- `npm run seed` **wipes the database** and creates 6 users, 19 categories (5 top-level plus subcategories), 10 brands, 35 products (most with variants), 15 historical orders, 20 verified-purchase reviews, 5 coupons and 3 banners. Product images are generated locally, so no network is needed.
- The seed refuses to run when `NODE_ENV=production` unless you pass `--force`.
- After reseeding, **restart the API** if it runs without Redis, because its in-process cache may still hold old data. With Redis, the seed flushes the cache itself.

---

## Running with Docker

```bash
cp .env.example .env     # set JWT_ACCESS_SECRET and JWT_REFRESH_SECRET (compose stops with a message if missing)
docker compose up --build
docker compose exec backend npm run seed -- --force     # optional demo data
```

- Store: <http://localhost:8080> (nginx serves the SPA and proxies `/api` and `/uploads` to the backend, so everything is same-origin)
- Services: `frontend` (nginx), `backend` (Node), `mongodb` (Mongo 7), `redis` (Redis 7), with named volumes for data and uploads
- Serving plain HTTP on a host other than `localhost`? Set `COOKIE_SECURE=false` in `.env`, or login cookies will be dropped by the browser.

---

## Payments

`PAYMENT_PROVIDERS` lists the enabled methods. A gateway is only offered when its credentials are set. `GET /api/payments/config` tells the client what is available.

**Flow:** `POST /api/orders` (stock reserved) → `POST /api/payments/create` (gateway order/session) → customer pays → `POST /api/payments/verify` **and/or** webhook → order confirmed, stock committed, emails sent. Unpaid orders are cancelled after `PAYMENT_PENDING_TIMEOUT_MINUTES`, and their stock is released.

| Provider | Client experience | Verification | Webhook URL | Events |
|---|---|---|---|---|
| Razorpay | Razorpay Checkout modal (script loaded on demand) | HMAC-SHA256 of `order_id\|payment_id` with the key secret | `POST /api/payments/webhook/razorpay` | `payment.captured`, `order.paid`, `payment.failed` |
| Stripe | Redirect to Stripe Checkout, back to `/checkout/result` | Session retrieved server-side; `payment_status === 'paid'` | `POST /api/payments/webhook/stripe` | `checkout.session.completed`, `…async_payment_succeeded`, `…async_payment_failed`, `checkout.session.expired` |
| COD | No online step; order confirmed immediately | — | — | Marked paid when delivered |

Refunds are issued through the gateway automatically when a paid order is cancelled, and can also be issued by an admin (status `REFUNDED`). Enable **automatic capture** in the Razorpay dashboard. Local webhook testing: `stripe listen --forward-to localhost:5000/api/payments/webhook/stripe`, or an ngrok tunnel for Razorpay.

---

## API reference

Base URL `/api`. Every response uses the same envelope:

```json
{ "success": true,  "message": "Request successful", "data": {}, "pagination": { "page": 1, "limit": 20, "total": 100, "totalPages": 5 } }
{ "success": false, "message": "Validation failed", "error": [{ "field": "email", "message": "Invalid email address", "location": "body" }] }
```

Status codes: 400 (bad request), 401 (unauthenticated), 403 (forbidden), 404, 409 (conflict: duplicates, stock, concurrency), 413 (payload too large), 422 (validation), 429 (rate limited), 500.
Auth: `Authorization: Bearer <accessToken>`; the refresh token is an HTTP-only cookie scoped to `/api/auth`. 🔒 = signed in, 🛡 = ADMIN or SUPER_ADMIN.

| Method & path | Description |
|---|---|
| **Auth** | |
| `POST /auth/register` · `POST /auth/login` | Create account / sign in → `{ user, accessToken }` and refresh cookie |
| `POST /auth/refresh` · `POST /auth/logout` | Rotate tokens / revoke session |
| `POST /auth/forgot-password` · `POST /auth/reset-password` | Email reset link (same response whether or not the account exists) / set new password |
| `POST /auth/verify-email` · `POST /auth/resend-verification` 🔒 | Verify email token / resend link |
| `GET /auth/me` 🔒 · `PUT /auth/change-password` 🔒 | Current user / change password (signs out other devices) |
| `PATCH /users/me` 🔒 · `POST /users/me/avatar` 🔒 | Update profile / upload avatar (multipart `avatar`) |
| **Catalogue** | |
| `GET /products` | List with `page, limit, q, category, subCategory, brand` (comma-separated slugs or ids), `minPrice, maxPrice, rating, inStock, minDiscount, featured, attr_<name>=a,b, sort` (`newest, price_asc, price_desc, rating, popular, discount, relevance`) |
| `GET /products/suggestions?q=` · `GET /products/facets` | Search suggestions / filter facets (brands, price range, attributes) |
| `GET /products/:idOrSlug` · `GET /products/:id/related` | Product details / related products |
| `POST /products` 🛡 · `PUT /products/:id` 🛡 · `DELETE /products/:id` 🛡 | Create / update / delete |
| `PATCH /products/:id/status` 🛡 · `POST /products/bulk-delete` 🛡 | Activate or deactivate / bulk delete |
| `GET /categories` · `GET /categories/tree` · `GET /categories/:slug` | Active categories (flat, tree, single) |
| `POST /categories` 🛡 · `PUT /categories/:id` 🛡 · `DELETE /categories/:id` 🛡 | Manage categories (same shape for `/brands`) |
| `GET /brands` · `GET /brands/:slug` | Active brands |
| `GET /home` · `GET /banners` · `GET /settings/public` · `POST /newsletter` | Homepage payload, banners, public settings, newsletter signup |
| `POST /uploads/products` 🛡 · `POST /uploads/catalog` 🛡 · `POST /uploads/reviews` 🔒 | Upload images (multipart `images`) → stored image descriptors |
| **Reviews** | |
| `GET /products/:id/reviews` | Approved reviews with `page, limit, sort, rating` plus a rating breakdown |
| `GET /products/:id/reviews/eligibility` 🔒 · `POST /products/:id/reviews` 🔒 | Whether the user may review / create a review (delivered purchase required) |
| `PUT /reviews/:id` 🔒 · `DELETE /reviews/:id` 🔒 | Edit or delete your own review |
| **Cart, wishlist, addresses** | |
| `GET /cart` 🔒 · `GET /cart/count` 🔒 | Priced cart (`couponCode`, `shippingMethod` query) / item count |
| `POST /cart` 🔒 · `PUT /cart/:itemId` 🔒 · `DELETE /cart/:itemId` 🔒 · `DELETE /cart` 🔒 | Add / update quantity / remove / clear |
| `POST /cart/merge` 🔒 · `POST /cart/preview` | Merge guest cart / price a guest cart |
| `GET /wishlist` 🔒 · `GET /wishlist/ids` 🔒 · `POST /wishlist` 🔒 · `DELETE /wishlist/:productId` 🔒 · `POST /wishlist/:productId/move-to-cart` 🔒 | Wishlist |
| `GET/POST /addresses` 🔒 · `PUT/DELETE /addresses/:id` 🔒 · `PATCH /addresses/:id/default` 🔒 | Address book |
| `POST /coupons/validate` 🔒 | Validate a coupon against your cart → discount and recomputed totals |
| **Orders & payments** | |
| `POST /orders` 🔒 | Place an order from the cart: `{ addressId, billingAddressId?, shippingMethod, paymentMethod, couponCode?, notes? }` |
| `GET /orders` 🔒 · `GET /orders/:id` 🔒 | Your orders |
| `PUT /orders/:id/cancel` 🔒 · `POST /orders/:id/return` 🔒 | Cancel (pending, confirmed or processing) / request a return (within 7 days of delivery) |
| `GET /payments/config` · `POST /payments/create` 🔒 · `POST /payments/verify` 🔒 | Enabled methods and public keys / start payment / verify payment |
| `POST /payments/webhook/:provider` | Gateway webhooks (raw body, signature verified) |
| **Admin** 🛡 | |
| `GET /admin/dashboard` · `GET /admin/analytics?days=30` | KPIs / time series and breakdowns |
| `GET /admin/orders` · `GET /admin/orders/:id` · `PATCH /admin/orders/:id/status` | Orders; status updates follow the allowed transition map (`{ status, note, trackingNumber, carrier }`) |
| `GET /admin/products` · `GET /admin/products/:id` | Products, including inactive ones (`status`, `stockStatus` filters) |
| `GET /admin/categories` · `GET /admin/brands` | Including inactive entries |
| `GET /admin/inventory` · `PATCH /admin/inventory/:id` | Stock overview (`status=low\|out`) / adjust stock |
| `GET /admin/users` · `PATCH /admin/users/:id` | Customers; role and active status (admin roles are managed by SUPER_ADMIN only) |
| `GET/POST /admin/coupons` · `PUT/DELETE /admin/coupons/:id` | Coupons |
| `GET /admin/reviews` · `PATCH /admin/reviews/:id` · `DELETE /admin/reviews/:id` | Moderation |
| `GET/POST /admin/banners` · `PUT/DELETE /admin/banners/:id` | Banners |
| `GET/PUT /admin/settings` | Store settings |

---

## Testing

```bash
cd server && npm test        # 51 API tests (Jest + Supertest, in-memory MongoDB; downloads a MongoDB binary on first run)
cd client && npm test        # 12 client tests (Vitest + React Testing Library)
```

The API suites cover authentication (hashing, refresh rotation and reuse detection, password reset, roles, operator injection), products (filters, search, facets, admin CRUD and validation), cart and wishlist, coupons (every rule, plus per-user limits across real orders), orders (server-side totals, overselling under concurrent checkout, cancellation and restock, the status workflow, returns, payment expiry), payments (signature verification, idempotent webhooks, automatic refunds; the Razorpay provider is tested against real HMAC logic) and reviews (purchase requirement, rating aggregation, moderation).

---

## Security

- Helmet security headers (a strict CSP for the JSON API), CORS allow-list, `x-powered-by` disabled, request size limits (1 MB JSON, 5 MB per image)
- Rate limits: global API, authentication (20 per 15 minutes), sensitive actions such as password reset (5 per hour); stored in Redis when available
- Zod validation on every input; unknown fields are stripped (no mass assignment); keys starting with `$` or containing `.` are removed (NoSQL injection); HTML tags are stripped from text input; React escapes output; JSON-LD is escaped
- bcrypt (cost 12); JWT access tokens (15 minutes) held in memory; rotating refresh tokens in HTTP-only, `SameSite=Lax`, `Secure` cookies; reset and verification tokens stored as SHA-256 hashes with expiry
- Role-based authorization on every admin route; users can only reach their own orders, addresses and reviews
- Uploads: MIME allow-list, size and count limits, content verified by decoding with Sharp, then re-encoded to WebP (which strips metadata); storage keys are path-traversal safe; review images are restricted to the reviews folder
- Payments are never trusted from the client: signatures and sessions are verified server-side, webhooks are signature-checked over the raw body, and gateway secrets never leave the server
- Structured Pino logs for auth, payment, order and admin events (plus a persistent `AuditLog` collection for admin actions); authorization headers, cookies, passwords and tokens are redacted

---

## Performance

- Route-level code splitting (every page is lazy-loaded); charts are isolated in their own chunk, loaded only by the admin dashboard
- Lazy-loaded, sized images; uploads are compressed to WebP in two sizes (1600px and 480px thumbnails)
- Lean list projections, compound indexes for every listing filter and sort, and pagination everywhere (maximum 100 items per page)
- Caching (Redis, or in-process fallback): homepage rails (2 min), categories and brands (10 min), banners and settings (5 min), with invalidation on admin changes
- gzip compression, immutable caching for hashed assets and uploads (nginx and Express), memoized product cards, optimistic wishlist updates

---

## Deployment

1. **Database:** MongoDB Atlas (or a self-hosted replica set). Set `MONGO_URI`. Create indexes once (seed with `--force` on a fresh database, or run `syncIndexes()`).
2. **Redis:** a managed Redis instance; set `REDIS_URL`. Recommended whenever you run more than one API instance, so rate limits and cache invalidation are shared.
3. **Secrets:** generate strong JWT secrets and store every key in your platform's secret manager, never in the image.
4. **Storage:** use `S3` or `CLOUDINARY` in production. Local disk does not work across multiple instances unless it is a shared volume.
5. **API:** build `server/Dockerfile` and run behind HTTPS (load balancer or ingress). Set `TRUST_PROXY` to the number of proxies, `CLIENT_URL` to your domain and `NODE_ENV=production`. Health check: `GET /api/health`.
6. **Client:** build `client/Dockerfile` (nginx), or deploy `client/dist` to any static host or CDN. If the API is on a different origin, set `VITE_API_URL=https://api.example.com/api`, add the storefront origin to `CLIENT_URL`, and serve both over HTTPS. For cross-site cookies, the refresh cookie needs `SameSite=None; Secure`, so a same-origin reverse proxy (as in the provided nginx config) is simpler.
7. **Payments:** set live keys and register the webhook URLs listed in [Payments](#payments).
8. **Email:** configure SMTP (SES, SendGrid, Postmark, …) and `EMAIL_FROM` on a verified domain.

---

## Known limitations

- Background jobs (payment expiry) run in-process on an interval. They are idempotent and safe across replicas, but a dedicated queue (e.g. BullMQ) would be a natural next step at scale.
- Search uses MongoDB text indexes with a prefix fallback. For typo tolerance and relevance tuning, consider Atlas Search or Meilisearch.
- Prices are single-currency (`CURRENCY`); tax is one configurable rate applied to the discounted subtotal.
- Uploaded images that are never attached to a product, review or banner (for example an abandoned form) are not garbage-collected.
- The storefront is a client-rendered SPA: SEO tags and structured data are rendered by React at runtime. Server-side rendering or prerendering would further improve crawler coverage.
