# LayerCraft 3D — Full-Stack E-Commerce Platform

A complete, production-ready website and admin system for **LayerCraft 3D** — a custom 3D printing and personalised gifting business.

- **Customer storefront** (single-page, premium light theme): Home, Shop, Products, How It Works, Gifting, FAQ, Contact
- **Product customisation** directly from the product card/modal (size, finish, material, colour, quantity, custom text, design instructions, reference-photo upload)
- **Cart** (persistent, quote-aware) → **Checkout** → **WhatsApp ordering** (structured message, order saved to DB first)
- **Customer chat assistant** (DB-driven knowledge base → optional AI escalation → WhatsApp handoff)
- **Secure admin dashboard** at `/admin` (JWT auth, activity log): Dashboard, Orders, Products, Catalog (categories/materials/colours), FAQ, Settings, Import/Export
- **SEO**: metadata, Open Graph, Twitter cards, sitemap, robots, JSON-LD (Organization + Product ItemList)

---

## 1. Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16 (App Router), React 19, TypeScript 5 |
| Styling | Tailwind CSS 4 + shadcn/ui (New York), lucide icons |
| Database | Prisma ORM — SQLite for local sandbox, **PostgreSQL/Supabase for production** (`supabase/schema.sql`) |
| Auth | Custom JWT sessions (jose) in httpOnly cookies + bcrypt password hashing, middleware + per-route guards |
| Validation | Zod (shared server-side schemas) |
| State | Zustand (cart, persisted to localStorage) |
| AI chat | z-ai-web-dev-sdk (server-side only) behind a swappable provider abstraction |
| Uploads | Local `public/uploads/` in sandbox with magic-byte validation; **Supabase Storage in production** |

> **Note on Supabase:** the sandbox environment runs SQLite. The data model, APIs and UI are database-agnostic. `supabase/schema.sql` + `supabase/seed.sql` contain the full PostgreSQL/Supabase deployment (tables, indexes, RLS policies, storage buckets guidance, seed data) — the app points at Supabase simply by changing `DATABASE_URL`.

---

## 2. Quick Start (Local Development)

```bash
# 1. Install dependencies
bun install        # or npm install / pnpm install

# 2. Configure environment
cp .env.example .env.local
#   set AUTH_SECRET (openssl rand -hex 32), ADMIN_EMAIL, ADMIN_PASSWORD

# 3. Create the database + seed the LayerCraft 3D catalog
bun run db:push        # creates tables from prisma/schema.prisma
bun prisma/seed.ts     # seeds categories, products, variants, materials, colours, FAQs, settings, admin

# 4. Start
bun run dev            # http://localhost:3000
```

- Storefront: `http://localhost:3000`
- Admin: `http://localhost:3000/admin` → sign in with `ADMIN_EMAIL` / `ADMIN_PASSWORD`

**Lint / type check:** `bun run lint` and `bunx tsc --noEmit`

---

## 3. Admin Setup

1. Set `ADMIN_EMAIL` and `ADMIN_PASSWORD` in `.env.local` **before** running the seed (defaults: `admin@layercraft3d.com` / `LayerCraft@3D` — **change these**).
2. Run `bun prisma/seed.ts`. The password is hashed with bcrypt (cost 12) and stored in `admin_users`.
3. Sign in at `/admin`. Sessions are signed JWTs in an httpOnly, SameSite cookie valid 7 days.
4. Every admin mutation is recorded in the **Data & Activity** tab (who / what / when).

Additional admins: add a row to `admin_users` (hash with bcrypt), or re-use the same flow. To rotate the admin password, update `password_hash` or re-seed with a new `ADMIN_PASSWORD`.

### Security model
- `/api/admin/*` is protected twice: middleware (JWT verify + same-origin check on mutations) **and** every route handler re-verifies the session server-side.
- No admin routes, links or UI ship to the customer storefront (only a subtle `/admin` link in the footer leads to the protected login).
- All prices for orders are resolved **server-side** from the database — client-sent prices are never trusted.
- Uploads validate MIME type, extension, size (≤5 MB) **and magic bytes**; arbitrary files are rejected.
- Rate limiting (in-memory sliding window): login 5/min, uploads 10/min, orders 6/min, chat 20/min per IP.
- Raw database errors are never returned to customers — friendly messages only.

---

## 4. Supabase Setup (Production Database)

1. Create a project at [supabase.com](https://supabase.com).
2. SQL Editor → run `supabase/schema.sql` (tables, indexes, RLS, triggers).
3. SQL Editor → run `supabase/seed.sql` (catalog + settings seed).
4. Storage → create two **public-read** buckets: `products` and `references`.
   - Policy: public `SELECT`; `INSERT/DELETE` only for authenticated admins.
5. Project Settings → Database → copy the **connection string** (use the "Connection pooling" / port 6543 string for serverless).
6. Set `DATABASE_URL` to that connection string and deploy the app (see §6).
7. Seed the admin there too: run `bun prisma/seed.ts` locally with `DATABASE_URL` pointing at Supabase, **or** create the user via Supabase Auth and insert the matching `admin_users` row (see the note at the bottom of `seed.sql`).

RLS summary: anon can read **active** catalog / **published** FAQs / settings only. Orders, chats, logs and all writes require an authenticated admin (or the server's trusted connection). Direct anonymous inserts are blocked.

---

## 5. GitHub Setup

```bash
git init
git add .
git commit -m "LayerCraft 3D — initial production build"
git branch -M main
git remote add origin git@github.com:<you>/layercraft3d.git
git push -u origin main
```

`.gitignore` already excludes `.env*`, `node_modules`, `.next`, logs and the SQLite db file. **Never commit `.env.local`.**

---

## 6. Vercel Deployment

1. Push to GitHub (§5) → Vercel → **New Project** → import the repo.
2. Environment Variables (Project → Settings):
   - `DATABASE_URL` — your Supabase pooled connection string
   - `AUTH_SECRET` — long random string
   - `NEXT_PUBLIC_SITE_URL` — `https://yourdomain.com`
   - `ADMIN_EMAIL`, `ADMIN_PASSWORD` (only needed for seeding)
   - `AI_CHAT_ENABLED=true` (optional)
3. Deploy. Before the first deploy, ensure the Supabase schema + seed ran (§4).
4. Add your custom domain in Vercel → Domains.

> **Image storage note:** on serverless hosting the local `public/uploads` folder is ephemeral. For production, upload product photos through the admin (they land in the DB as URLs) after switching storage to Supabase Storage, or commit your product images to `/public/images/products` like the seeded ones.

---

## 7. Business Owner's Guide

### How to add a product
Admin → **Products** → **+ New Product**:
1. Fill **Name** (slug auto-generates), category, short + full description.
2. **Pricing & visibility**: base price (single price), max price (for ranges like lamps), or tick **Quote only** for "Custom Quote" products.
3. **Variants** → add rows like `10 cm — Plain · ₹1,500` (size, finish, price).
4. Tick the **materials** (PLA / PLA+ / PETG) and **colours** the product supports — the storefront only ever shows these options.
5. **Images** → upload photos or add a URL → pick the **primary** image.
6. Toggle **Featured / Active**, **Save**. The storefront updates instantly — no code changes needed.

### How to upload images
In the product form → **Images** → *Upload files* (JPG/PNG/WEBP, ≤5 MB each). The first/primary image is shown on the product card. Customer reference photos upload automatically in the Customise panel and attach to their order (visible in the admin order detail).

### How to change prices
Admin → **Products** → ✏️ **Edit** → update the variant price or base/max price → **Save changes**. Prices, ranges and "Custom Quote" badges everywhere on the site come from the database and update immediately. Publish toggle (👁 switch) unpublishes a product without deleting it.

### How to manage orders
Admin → **Orders** → search by order number/customer/phone/city, filter by status:
- Open an order → see customer details, every item's variant/material/colour/quantity, customisation text and reference images.
- Update **Order status** (New → Confirmed → Design Review → Production → Quality Check → Ready → Shipped → Delivered / Cancelled), **Payment status** (Pending / Paid / Refunded / Not Applicable) and **WhatsApp status**.
- **Open WhatsApp** button messages the customer prefilled with their order number.
- **Internal notes** stay admin-only.

### How to change the WhatsApp number
Admin → **Settings** → **Business** → *WhatsApp number* (digits with country code, e.g. `919821434440`) and *WhatsApp display number* (e.g. `+91 98214 34440`) → **Save settings**. Every WhatsApp button on the site (order messages, chat handoff, contact) updates from this single setting — nothing is hardcoded.

### Chatbot configuration
- **Welcome message**: Admin → **Settings** → **Chat** → *Chatbot welcome message*.
- **Knowledge**: the assistant answers from your live database — products, prices, materials, colours, FAQs and business info. Editing FAQs/products/settings immediately improves its answers.
- **AI escalation**: unknown questions go to the server-side AI provider (never exposes keys to the browser). Disable with `AI_CHAT_ENABLED=false` — the bot then falls back to structured answers plus a **"Talk to an agent on WhatsApp"** handoff button (prefilled message).
- To swap AI providers later, edit only `src/lib/chat/service.ts` (`aiReply`) — the UI never changes.

### Product Import / Export
Admin → **Data & Activity** → **Export catalog JSON** downloads the full catalog. **Import** validates every product (slug, prices, references) and **rejects the whole batch if anything is malformed** — malformed data can never break the storefront. Products are matched by slug and upserted.

---

## 8. Project Structure

```
src/
├─ app/
│  ├─ page.tsx               # storefront (SSR data + client islands)
│  ├─ layout.tsx             # fonts, SEO metadata, toaster
│  ├─ sitemap.ts / robots.ts # SEO routes
│  ├─ admin/page.tsx         # admin auth gate + shell
│  └─ api/                   # public + admin REST endpoints
│     ├─ products, categories, faq, settings, upload, orders, chat
│     ├─ auth/login|logout|me
│     └─ admin/dashboard|products|orders|categories|materials|colors|faq|settings|upload|export|activity
├─ components/
│  ├─ site/                  # navbar, hero, catalog, modals, sections, footer
│  ├─ cart/                  # zustand store + drawer + sticky bar
│  ├─ checkout/              # checkout dialog (WhatsApp ordering)
│  ├─ chat/                  # floating assistant widget
│  ├─ admin/                 # login, dashboard, tabs (orders/products/catalog/faq/settings/data)
│  ├─ shared/                # price tag, image fallback
│  └─ ui/                    # shadcn/ui
├─ services/                 # catalog, orders (server price resolution), uploads, admin products
├─ lib/                      # db, auth, validation (zod), settings, whatsapp builders, rate-limit, chat engine
├─ types/                    # shared DTOs
└─ middleware.ts             # admin API guard + same-origin check
prisma/                      # schema + seed.ts
supabase/                    # schema.sql + seed.sql (PostgreSQL/RLS for production)
public/uploads/              # runtime uploads (products, references)
```

---

## 9. Final Checklist

- ✅ **No customer-facing admin tools** — the storefront exposes zero admin functionality (only a subtle link to the protected `/admin` login).
- ✅ **Admin is authentication protected** — bcrypt credentials, JWT httpOnly cookie, middleware + per-route verification, same-origin mutation checks, login rate limiting, activity log.
- ✅ **Products come from the database** — the storefront reads the catalog dynamically via SSR services/APIs; admin edits appear instantly; no hardcoded catalog in frontend code.
- ✅ **Images use storage** — product images and customer reference uploads are stored and served from the uploads directory (Supabase Storage documented for production) with strict validation.
- ✅ **Cart works** — add/remove/quantity/merge logic, persists across visits (localStorage), quote-aware totals.
- ✅ **Checkout works** — full validation (10-digit phone, 6-digit PIN), server-side price resolution, order persisted **before** WhatsApp redirect, friendly errors.
- ✅ **WhatsApp order works** — structured message (customer, order number, per-item variant/material/colour/qty/customisation/pricing, totals, confirmation line) opened via `wa.me`; opening WhatsApp is explicitly **not** payment confirmation.
- ✅ **Chatbot works** — DB-driven knowledge answers (materials, prices, customisation, ordering, delivery policy), optional server-side AI escalation for unknown questions, WhatsApp handoff with prefilled message.
- ✅ **Mobile responsive** — sticky cart bar, sheet nav, stacked layouts, 44px touch targets, chat widget offset, safe-area padding.
- ✅ **No secret keys exposed** — all credentials server-side (`.env.local` / host env), AI SDK used only in backend, `.env.example` documents every variable, `.gitignore` excludes env files.

---

## 10. Future-Ready Hooks

The architecture leaves clean seams for: Razorpay/Stripe/UPI (add a payment provider + verify before `paymentStatus = "Paid"` — no fake payments exist now), coupons (orders already carry `discount`), shipping providers (settings carry `shipping.flatRate`), GST invoices, email/SMS/WhatsApp Cloud API notifications, customer accounts/wishlists/reviews and stock management (new tables + admin tabs following the existing patterns).
