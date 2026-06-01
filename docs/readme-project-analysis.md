# PrintByFalcon — Project Analysis for README

> **Purpose:** This document captures a comprehensive analysis of the entire PrintByFalcon project.
> It serves as a **session checkpoint** — if the context fills up, a new session can read this file
> to resume work on the README without re-analyzing the entire codebase.
>
> **Last updated:** 2026-06-01

---

## 1. Project Identity

| Field | Value |
|---|---|
| **Name** | Print By Falcon |
| **Domain** | `printbyfalcon.com` |
| **Type** | Full-stack e-commerce platform |
| **Market** | Egyptian market (B2C + B2B) |
| **Niche** | Printers, toner cartridges, ink cartridges, and printing supplies |
| **Languages** | Arabic (default, RTL) + English (LTR) |
| **Currency** | Egyptian Pound (EGP) |
| **Owner** | Omar — `support@printbyfalcon.com` |
| **Solo developer** | Yes — built entirely by one person |
| **Status** | Live in production on Hostinger VPS |

---

## 2. Tech Stack (Complete)

### Core Framework
- **Next.js 15** (App Router, TypeScript) — full-stack framework
- **React 19** (RC) — UI library
- **TypeScript 5.6** — type safety

### Database & ORM
- **PostgreSQL 16** — primary database (data + queue + sessions + audit log)
- **Prisma 5.22** — ORM with full-text search preview feature
- **42 KB schema** — 30+ models, comprehensive data model

### Styling & UI
- **Tailwind CSS 3.4** — utility-first CSS
- **shadcn/ui** (Radix UI primitives) — component system
- **Lucide React** — icon library
- **IBM Plex Sans Arabic** — Arabic font
- **Inter** — English font (via next/font)

### Authentication & Security
- **Custom auth system** (no Auth.js — ADR-021)
- **bcrypt** (cost 12) — password hashing
- **DB-backed sessions** — 30-day rolling, SHA-256 hashed tokens
- **WhatsApp OTP** (6-digit, SHA-256 hashed, 5-min expiry)
- **Rate limiting** — DB sliding-window counter
- **Zod** — validation schemas (shared client/server)

### Payments
- **Paymob** — card payments (Visa/Mastercard/Meeza) + Fawry pay-at-outlet
- **Cash on Delivery (COD)** — with configurable fees per zone
- **Submit for Review** — B2B payment method

### Notifications & Communication
- **Whats360** — WhatsApp automation (OTP, order status, notifications)
- **Nodemailer** — transactional emails via Hostinger SMTP
- **pg-boss** — background job queue for notifications

### Internationalization
- **next-intl** — i18n framework
- **Bilingual** — Arabic (AR, default) + English (EN)
- **Full RTL support** — Tailwind logical properties

### Background Processing
- **pg-boss 10.1** — PostgreSQL-based job queue
- **Worker process** — separate Node.js container
- Jobs: send-whatsapp, send-email, heartbeat

### Infrastructure & Deployment
- **Docker** — containerized (app + worker + postgres + nginx)
- **Nginx** — reverse proxy + SSL + static assets
- **Cloudflare Free** — CDN, DNS, DDoS protection, WAF
- **Hostinger KVM2 VPS** — 2 vCPU / 8 GB RAM / 100 GB NVMe
- **Let's Encrypt (Certbot)** — SSL certificates
- **GitHub Actions** — CI/CD (lint, typecheck, test, build, deploy)

### Monitoring & Error Tracking
- **GlitchTip** (self-hosted) — Sentry-compatible error tracking
- **Netdata** — server metrics
- **UptimeRobot** — uptime monitoring
- **Pino** — structured JSON logging

### Image Processing
- **Sharp** — generates 3 WebP sizes (thumb 200px, medium 800px, original 1600px)

### PDF Generation
- **@react-pdf/renderer** — invoice PDF generation

### Testing
- **Vitest** — unit tests (200+ tests)
- **Playwright** — E2E tests (10 test files)
- **ESLint** + **Prettier** + **Husky** — code quality

### SEO
- Dynamic sitemap (`/sitemap.xml`)
- Robots.txt
- Schema.org JSON-LD on product pages
- OpenGraph + Twitter meta tags
- Canonical URLs + hreflang alternates

---

## 3. Architecture Overview

### Single-VPS Architecture ("Small and boring")
```
Internet → Cloudflare (CDN/DNS/TLS/DDoS) → Nginx → Next.js App
                                                  → pg-boss Worker
                                                  → PostgreSQL 16
                                                  → GlitchTip
                                                  → Netdata
```

### Three Surfaces
1. **B2C Storefront** — server-rendered for SEO, Egyptian individual buyers
2. **B2B Portal** — server-rendered with client-side cart/bulk-order tools, Egyptian companies
3. **Admin Panel** (`/admin/*`) — owner, ops team, sales reps

### Environments
| Env | Domain | Purpose |
|---|---|---|
| Development | `localhost:3000` | Local dev with Docker Postgres |
| Staging | `staging.printbyfalcon.com` | Pre-production testing |
| Production | `printbyfalcon.com` | Live site |

---

## 4. Database Schema (30+ Models)

### Identity & Access (7 models)
- `User` — B2C/B2B/ADMIN with role-based access
- `Session` — DB-backed, 30-day rolling
- `WhatsAppOtp` — SHA-256 hashed, rate-limited
- `PasswordReset` — token-based, 1-hour expiry
- `AdminInvite` — invite-based admin onboarding
- `RateLimit` — sliding window counter
- `Setting` — flexible KV store (JSON-valued)

### Catalog (6 models)
- `Brand` — HP, Canon, Epson, etc. (bilingual)
- `Category` — unlimited nesting tree (bilingual)
- `Product` — bilingual, FTS vector, specs JSON, pricing
- `ProductImage` — 3 WebP variants per image
- `PrinterModel` — printer compatibility reference
- `ProductCompatibility` — product ↔ printer many-to-many

### Commerce (10+ models)
- `Cart` / `CartItem` — session-based for guests, user-based for logged-in
- `Inventory` / `InventoryReservation` / `InventoryMovement` — full stock management
- `Order` / `OrderItem` / `OrderStatusEvent` / `OrderDailySequence` — complete order lifecycle
- `Invoice` / `InvoiceAnnualSequence` — gapless annual numbering

### Shipping & Geography (5 models)
- `Address` — 27 Egyptian governorates enum
- `ShippingZone` — admin-configurable zones with rates
- `GovernorateZone` / `GovernorateConfig` — governorate-to-zone mapping
- `Courier` — delivery partner management

### B2B (4 models)
- `Company` — approved B2B companies with pricing tiers
- `B2BApplication` — application pipeline (pending → approved/rejected)
- `PricingTier` — A (10% off), B (15% off), C (per-SKU)
- `CompanyPriceOverride` — per-SKU negotiated prices

### Notifications & Audit (4 models)
- `Notification` — WhatsApp + email send log
- `NotificationOptOut` — customer opt-out (STOP keyword)
- `AuditLog` — comprehensive append-only audit trail
- `Return` / `ReturnItem` — returns management

### Payments & Promotions
- `PaymentMethodConfig` — admin-toggled payment methods
- `PromoCode` — percentage or fixed discount codes

---

## 5. Key Features

### Storefront (B2C)
- [x] Bilingual homepage with hero, value props, categories, featured products, brands
- [x] Full-text search (Postgres FTS + trigram fallback)
- [x] Search by printer model (compatibility cross-reference)
- [x] Product catalog with filters (brand, category, price, authenticity, stock)
- [x] Product detail pages with gallery, specs, compatibility list
- [x] Shopping cart with 15-min stock soft holds
- [x] Guest checkout + signed-in checkout
- [x] Address management (5 max, default selection)
- [x] Order tracking with status timeline
- [x] WhatsApp OTP authentication (B2C)
- [x] Cookie consent banner
- [x] Privacy policy + Terms of service + Cookie policy

### B2B Portal
- [x] Company registration with CR number + tax card
- [x] Admin approval pipeline (pending → approved/rejected)
- [x] Tiered pricing (A/B/C tiers + per-SKU overrides)
- [x] "Submit for Review" checkout (sales-rep mediated)
- [x] Company-wide order history
- [x] Credit terms (None/Net-15/Net-30/Custom)
- [x] Checkout policy per company (Both/SFR-only/Pay-now-only)

### Admin Panel
- [x] Dashboard with sales metrics and charts
- [x] Full catalog CRUD (products, brands, categories, printer models)
- [x] Image management (upload, reorder, alt text)
- [x] Inventory management (receive stock, adjust, bulk operations)
- [x] Order management with status transitions
- [x] Courier handoff modal (courier selection, waybill, delivery date)
- [x] Cancellation queue with approve/deny
- [x] Returns log with refund decisions
- [x] B2B application review queue
- [x] Company management (pricing tiers, credit terms, overrides)
- [x] Shipping zones + governorate configuration
- [x] COD policy management
- [x] Payment methods configuration
- [x] Promo code management
- [x] User management (invite admins, roles: Owner/Ops/Sales Rep)
- [x] Store info settings
- [x] WhatsApp device management
- [x] VAT rate configuration
- [x] Notification opt-out management
- [x] Invoice generation and management
- [x] Bulk operations (archive products, mark orders)

### Notifications
- [x] WhatsApp OTP (B2C auth)
- [x] Order confirmation (WhatsApp + email)
- [x] Order status changes
- [x] B2B pending review notification
- [x] Payment failure notification
- [x] Customer opt-out support (STOP keyword)
- [x] Rate-limited (5/phone/hour)

### Payments
- [x] Paymob card (Visa/Mastercard/Meeza) — hosted iframe
- [x] Paymob Fawry pay-at-outlet
- [x] Cash on Delivery (COD) — configurable fees per zone
- [x] B2B Submit for Review
- [x] HMAC webhook verification
- [x] Hourly reconciliation job for stale payments

---

## 6. Project Structure

```
D:\PrintByFalcon\
├── app/                        # Next.js App Router
│   ├── [locale]/               # /ar/* and /en/* routes
│   │   ├── account/            # User account pages
│   │   ├── admin/              # Admin panel (15 sections)
│   │   ├── b2b/                # B2B portal
│   │   ├── cart/               # Shopping cart
│   │   ├── categories/         # Category browse
│   │   ├── checkout/           # Checkout flow
│   │   ├── cookies/            # Cookie policy
│   │   ├── login/              # B2B login
│   │   ├── order/              # Order confirmation
│   │   ├── payments/           # Payment pages
│   │   ├── privacy/            # Privacy policy
│   │   ├── products/           # Product catalog
│   │   ├── search/             # Search page
│   │   ├── sign-in/            # B2C sign-in (OTP)
│   │   └── terms/              # Terms of service
│   ├── actions/                # 25 Server Action files
│   ├── api/                    # API routes (health, webhooks, search)
│   ├── invoices/               # Invoice PDF routes
│   └── storage/                # Dev-mode file serving
├── components/                 # React components
│   ├── admin/                  # 52 admin components
│   ├── account/                # Account components
│   ├── b2b/                    # B2B components
│   ├── cart/                   # Cart components
│   ├── catalog/                # Catalog components
│   ├── checkout/               # Checkout components
│   ├── order/                  # Order components
│   ├── ui/                     # 8 UI primitives (button, card, dialog, etc.)
│   └── (site-level)            # Header, footer, nav, search, language switcher
├── lib/                        # Business logic (20 directories, 19 files)
│   ├── auth/                   # Authentication helpers
│   ├── b2b/                    # B2B business logic
│   ├── cart/                   # Cart logic
│   ├── catalog/                # Catalog queries, search, FTS
│   ├── email/                  # Email templates
│   ├── i18n/                   # i18n configuration
│   ├── inventory/              # Inventory management
│   ├── invoices/               # Invoice generation
│   ├── notifications/          # Notification system
│   ├── order/                  # Order number generation
│   ├── orders/                 # Order queries
│   ├── payments/               # Paymob integration
│   ├── pricing/                # B2B pricing resolution
│   ├── promo/                  # Promo code logic
│   ├── returns/                # Returns processing
│   ├── settings/               # Settings management
│   ├── shipping/               # Shipping zone resolution
│   ├── storage/                # File storage helpers
│   └── validation/             # Zod schemas
├── worker/                     # pg-boss background worker
│   ├── index.ts                # Worker entry point
│   └── jobs/                   # Job handlers (heartbeat, email, whatsapp)
├── prisma/                     # Database schema + seed
├── messages/                   # i18n catalogs (ar.json, en.json)
├── docker/                     # Docker configs
│   ├── Dockerfile.app          # Next.js app container
│   ├── Dockerfile.worker       # Worker container
│   ├── docker-compose.prod.yml # Production stack
│   ├── docker-compose.staging.yml
│   ├── nginx/                  # Nginx configuration
│   └── postgres/               # Postgres configuration
├── scripts/                    # Utility scripts
│   ├── deploy-production.sh    # Production deploy
│   ├── deploy-staging.sh       # Staging deploy
│   ├── backup.sh               # Database backup
│   ├── restore.sh              # Database restore
│   ├── post-push.ts            # Post-deploy setup (FTS, indexes, seeds)
│   ├── seed-catalog.ts         # CSV catalog importer
│   ├── seed-orders.ts          # Demo order data
│   └── seed-b2b.ts             # B2B demo data
├── tests/e2e/                  # 10 Playwright E2E test files
├── .github/workflows/          # CI + deploy workflows
├── docs/                       # 25 documentation files
├── public/                     # Static assets
├── storage/                    # Product images + invoices
└── fixtures/                   # CSV test data
```

---

## 7. Sprint History & Timeline

| Sprint | Date | Focus |
|---|---|---|
| Sprint 1 | 2026-04-19 | Foundation — auth, sessions, rate limiting, admin invites, VPS setup |
| Sprint 2 | 2026-04-18 | Catalog — brands, categories, products, images, printer models |
| Sprint 3 | 2026-04-19 | Smart Search — FTS, filters, printer-model search, 200-SKU fixture |
| Sprint 4 | 2026-04-19 | B2C Accounts + Cart + Checkout + Paymob → M0 milestone |
| Sprint 5 | 2026-04-20 | Order Tracking + Notifications + Admin Order Management |
| Sprint 6 | ~2026-04-21 | Inventory polish, invoice metadata, store settings |
| Sprint 7 | ~2026-04-22 | B2B — Company, application pipeline, pricing tiers |
| Sprint 8 | ~2026-04-22 | B2B checkout (Submit for Review + Pay Now) |
| Sprint 9 | ~2026-04-23 | Shipping zones, COD policy, promo codes, Fawry via Paymob |
| Sprint 10 | ~2026-04-23 | Returns workflow, return policy, admin guide |
| Sprint 11 | ~2026-04-23 | Security hardening, M1 readiness, 200 tests, E2E coverage |
| Sprint 11.5 | 2026-05-06 | Admin-controlled config (governorates, zones, payment methods, WhatsApp) |

**Milestones:**
- **M0** — Internal demo (reached Sprint 4)
- **M1** — Soft launch to 5 B2C testers + 3 B2B companies (target: end of Sprint 12)

---

## 8. External Integrations

| Service | Purpose | Integration Type |
|---|---|---|
| **Paymob** | Card + Fawry payments | Hosted iframe + webhook |
| **Whats360** | WhatsApp automation | REST API + inbound webhook |
| **Hostinger SMTP** | Transactional emails | SMTP via Nodemailer |
| **Cloudflare Free** | CDN, DNS, DDoS, WAF | Proxy mode |
| **GlitchTip** | Error tracking (Sentry-compatible) | Self-hosted on same VPS |
| **UptimeRobot** | Uptime monitoring | External HTTP checks |
| **Netdata** | Server metrics | Self-hosted on same VPS |

---

## 9. Security Measures

- HTTPS enforced (Nginx 301 redirect)
- HTTP security headers (CSP, HSTS, X-Frame-Options, CORP, COOP)
- Passwords: bcrypt cost 12
- Sessions: crypto-random tokens, HttpOnly+Secure+SameSite=Lax cookies
- OTPs: SHA-256 hashed, 5-min expiry, 3 attempts max
- Rate limiting: DB-backed sliding window
- CSRF: Next.js Server Actions origin check
- SQL injection: Prisma parameterized queries
- XSS: React escaping + no dangerouslySetInnerHTML
- File uploads: MIME-sniffed, max 5 MB, re-encoded by Sharp
- Webhook signatures: HMAC verification
- Cloudflare WAF + DDoS protection
- Origin lockdown: UFW accepts only Cloudflare IPs
- Production env guard: fails boot if dev flags enabled
- OWASP Top 10 checklist — all green

---

## 10. Documentation Inventory

| File | Size | Purpose |
|---|---|---|
| `PRD.md` | 39 KB | Product Requirements Document |
| `architecture.md` | 35 KB | System architecture |
| `decisions.md` | 130 KB | Architectural Decision Records (ADRs) |
| `implementation-plan.md` | 86 KB | Sprint-by-sprint implementation plan |
| `progress.md` | 225 KB | Running sprint progress log |
| `design-system.md` | 21 KB | Design tokens and component catalog |
| `runbook.md` | 21 KB | Operations runbook |
| `admin-guide.md` | 8 KB | Admin panel user guide |
| `b2b-user-guide.md` | 5 KB | B2B user guide |
| `catalog-data-guide.md` | 10 KB | Catalog data entry guide |
| `order-ops-guide.md` | 9 KB | Order operations guide |
| `inventory-ops-guide.md` | 5 KB | Inventory operations guide |
| `returns-workflow.md` | 6 KB | Returns workflow documentation |
| `sales-rep-guide.md` | 11 KB | Sales representative guide |
| `security-audit.md` | 9 KB | Security audit results |
| `a11y-audit.md` | 5 KB | Accessibility audit |
| `e2e-coverage-matrix.md` | 6 KB | E2E test coverage matrix |
| `faq.md` | 8 KB | Bilingual FAQ |
| `m1-readiness.md` | 8 KB | M1 launch checklist |
| `sprint1-external-tasks.md` | 32 KB | Sprint 1 VPS/DNS/SSL setup runbook |
| `settings-panel-reference.md` | 7 KB | Settings panel reference |
| `settings-reference.md` | 6 KB | Settings keys reference |
| `paymob-key-rotation.md` | 8 KB | Paymob key rotation procedure |
| `whats360-key-rotation.md` | 7 KB | Whats360 key rotation procedure |
| `audit-log-queries.md` | 6 KB | Audit log SQL queries |

---

## 11. Key Design Decisions (Notable ADRs)

| ADR | Decision |
|---|---|
| ADR-005 | B2C uses WhatsApp OTP; B2B uses email+password |
| ADR-007 | One shared login per B2B company (MVP) |
| ADR-010 | No Redis — PostgreSQL for everything |
| ADR-015 | Staging + production on same VPS (cost optimization) |
| ADR-019 | Order number format: `ORD-YY-DDMM-NNNNN` |
| ADR-020 | Invoice number format: `INV-YY-NNNNNN` (gapless annual) |
| ADR-021 | Custom auth (no Auth.js) — Server Actions + Session table |
| ADR-022 | Direct Fawry integration descoped |
| ADR-024 | Cloudflare Free as CDN/DNS/TLS edge |
| ADR-025 | Fawry pay-at-outlet via Paymob sub-integration |
| ADR-027 | Unlimited category nesting |
| ADR-029 | Bilingual FTS uses `simple` config (no Arabic stemmer) |
| ADR-030 | COD pulled into Sprint 4 (from Sprint 9) |
| ADR-031 | Design direction: "Apple-Store restraint for a Cairo printer-supplies shop" |
| ADR-033 | Whats360 replaces Meta Cloud API (no template approval bottleneck) |

---

## 12. Numbers & Metrics

| Metric | Value |
|---|---|
| Database models | 30+ |
| Server Action files | 25 |
| Admin components | 52 |
| UI primitives | 8 |
| Unit tests | 200+ (Vitest) |
| E2E test files | 10 (Playwright) |
| i18n keys (AR) | ~13 KB JSON |
| i18n keys (EN) | ~10 KB JSON |
| Prisma schema | 1,142 lines |
| Documentation files | 25 |
| Total docs size | ~750 KB |
| GitHub workflows | 3 (CI, deploy-staging, deploy-production) |
| Docker containers | 4+ (app, worker, postgres, nginx) |
| Egyptian governorates | 27 (enum) |
| Expected daily visitors | 100–500 |

---

## 13. What Makes This Project Notable

1. **Solo-built production e-commerce** — entire platform built by one developer
2. **Egypt-first** — Egyptian governorates, EGP currency, Arabic RTL, Egyptian payment methods
3. **Dual-audience** — B2C (individual buyers) + B2B (companies with tiered pricing)
4. **Printer-specific** — unique compatibility search (search by printer model → find consumables)
5. **Full bilingual** — true Arabic/English with RTL support, not just translation
6. **Complete business logic** — orders, inventory, invoicing, returns, promo codes, shipping zones
7. **Self-hosted monitoring** — GlitchTip + Netdata on same VPS
8. **Extensive documentation** — 25 docs files, 750+ KB of documentation
9. **Security-first** — OWASP Top 10, CSP, HSTS, rate limiting, webhook HMAC
10. **WhatsApp-native** — OTP via WhatsApp (not SMS), order notifications via WhatsApp
