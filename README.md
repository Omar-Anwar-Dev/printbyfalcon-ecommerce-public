# 🦅 Print By Falcon

[![Next.js](https://img.shields.io/badge/Next.js-15.0-000000?style=flat-square&logo=next.js&logoColor=white)](https://nextjs.org/)
[![React](https://img.shields.io/badge/React-19.0-61DAFB?style=flat-square&logo=react&logoColor=black)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.6-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-3.4-38B2AC?style=flat-square&logo=tailwind-css&logoColor=white)](https://tailwindcss.com/)
[![Prisma ORM](https://img.shields.io/badge/Prisma-5.22-2D3748?style=flat-square&logo=prisma&logoColor=white)](https://www.prisma.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?style=flat-square&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-Enabled-2496ED?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com/)
[![Playwright](https://img.shields.io/badge/Playwright-E2E_Tests-2E8B57?style=flat-square&logo=playwright&logoColor=white)](https://playwright.dev/)
[![Vitest](https://img.shields.io/badge/Vitest-Unit_Tests-76E254?style=flat-square&logo=vitest&logoColor=black)](https://vitest.dev/)

A production-ready, highly-optimized, bilingual (Arabic & English) **B2C + B2B e-commerce platform** specializing in printers, toner/ink cartridges, and printing supplies, tailored specifically for the **Egyptian market**. 

Developed entirely by a solo developer, the project is fully dockerized and deployed in a high-availability, resource-optimized configuration on a **Hostinger VPS** behind a secure Cloudflare edge.

---

## 📖 Table of Contents

1. [System Architecture](#-system-architecture)
2. [Key Product Dimensions](#-key-product-dimensions)
3. [Technology Stack](#-technology-stack)
4. [Comprehensive Feature Set](#-comprehensive-feature-set)
5. [Database Architecture & Key Entities](#-database-architecture--key-entities)
6. [Local Development & Quick Start](#-local-development--quick-start)
7. [Scripts Reference Cheat Sheet](#-scripts-reference-cheat-sheet)
8. [Environment Variables Guide](#-environment-variables-guide)
9. [Project Directory Layout](#-project-directory-layout)
10. [API & Server Actions Reference](#-api--server-actions-reference)
11. [Security Hardening](#-security-hardening)
12. [Testing & Quality Assurance](#-testing--quality-assurance)
13. [Deployment & Operations](#-deployment--operations)
14. [Documentation Library Index](#-documentation-library-index)
15. [Contributing & Maintenance](#-contributing--maintenance)
16. [Contact & Support](#-contact--support)

---

## 🏛 System Architecture

Print By Falcon uses a **"Small & Boring" Single-VPS architecture** designed for high efficiency, reliability, and low operational overhead. 

### 1. Host Topology
The entire application stack, background worker queue, database, and monitoring run inside Docker containers on a single **Hostinger KVM2 VPS (2 vCPUs, 8 GB RAM, 100 GB NVMe)**.

```mermaid
graph TD
    Client[Client Browser / Mobile] -->|HTTPS / WAF| CF[Cloudflare CDN & DNS]
    CF -->|UFW Firewall Rule| Nginx[Nginx Reverse Proxy Container]
    
    subgraph "Hostinger KVM2 VPS"
        Nginx -->|Proxy Pass| NextApp[Next.js App Container]
        NextApp -->|Reads/Writes| Postgres[PostgreSQL 16 Container]
        NextApp -->|Uploads| LocalStorage[(Disk Storage: /storage)]
        
        Worker[pg-boss Worker Container] -->|Polls / Processes| Postgres
        Worker -->|Sends Notifications| SMS[Whats360 WhatsApp API]
        Worker -->|Sends Email| SMTP[Hostinger SMTP Service]
        
        %% Self-hosted monitoring
        GlitchTip[GlitchTip Error Tracker Container] <-->|Sentry DSN| NextApp
        Netdata[Netdata Performance Monitor Container] -->|Telemetry| Host[Host Metrics]
    end
```

### 2. Async Notification & Job Processing Flow
Instead of slowing down HTTP request cycles with third-party network calls, critical tasks (WhatsApp OTPs, order notifications, invoice creation) are offloaded to **pg-boss** (a PostgreSQL-backed job queue) running in a separate worker container.

```mermaid
sequenceDiagram
    participant C as Customer / Client
    participant A as Next.js Server Action
    participant DB as PostgreSQL DB (pg-boss queue)
    participant W as background worker
    participant WA as Whats360 WhatsApp API
    
    C->>A: Request OTP / Checkout Order
    activate A
    A->>DB: Write business entities
    A->>DB: Enqueue Job ('send-whatsapp', payload)
    A->>C: Respond immediately (success status)
    deactivate A
    
    Note over W, DB: Worker polls DB queue every 1s
    W->>DB: Fetch pending job
    activate W
    W->>WA: POST WhatsApp message payload
    WA-->>W: 200 OK (Delivered)
    W->>DB: Mark job as complete
    deactivate W
```

---

## 🌐 Key Product Dimensions

*   **Egyptian Market Optimization:** Deeply integrated with Egyptian geography (preloaded with all **27 Egyptian Governorates**), shipping logistics, localized currencies (EGP), and regional payment integrations.
*   **True Bilingual i18n & RTL:** Full integration of `next-intl` providing natural translation for both Arabic (default, RTL layout) and English (LTR layout). Layouts dynamically adapt via Tailwind logical properties.
*   **Dual-Audience Capabilities:** Runs both a standard public storefront for **B2C retail customers** and a protected, approval-based portal for **B2B corporate clients** complete with custom bulk features and personalized pricing tiers.
*   **Printer-Specific Search Intelligence:** A custom compatibility search engine allowing customers to select their printer model and immediately find all compatible toner and ink cartridges.

---

## 💻 Technology Stack

| Component | Technology | Version | Purpose & Rationale |
| :--- | :--- | :--- | :--- |
| **Framework** | **Next.js** (App Router) | `15.0` RC | Server-side rendering (SSR) for storefront SEO + Server Actions for security. |
| **Frontend UI** | **React** | `19.0` RC | Modern React primitives, optimized server rendering. |
| **Language** | **TypeScript** | `5.6` | Type-safety, cleaner code refactoring, self-documenting parameters. |
| **Styling** | **Tailwind CSS** | `3.4` | Rapid styling, custom theme variables, logical RTL directions (`start`/`end`). |
| **Components** | **shadcn/ui** | Latest | Base accessible primitives (Radix UI) custom-themed to the project. |
| **Database** | **PostgreSQL** | `16.0` | Relational structure, ACID compliance, pg-boss backing, Full-Text Search. |
| **ORM** | **Prisma** | `5.22` | Intuitive type-safe client, migration runner, full-text search preview. |
| **Queue Manager**| **pg-boss** | `10.1` | PostgreSQL-backed job processor. Avoids the memory and cost overhead of Redis. |
| **Payments** | **Paymob** | API v2 | Localized payment processor supporting credit cards, Meeza, and Fawry outlets. |
| **WhatsApp API**| **Whats360** | API v3 | Bypasses standard Meta Business API template approval bottlenecks. |
| **Error Logging**| **GlitchTip** | Latest | Self-hosted, lightweight Sentry-compatible error logging. |
| **Testing** | **Vitest / Playwright**| Latest | Fast unit testing (Vitest) + bulletproof E2E checkout verification (Playwright). |

---

## ✨ Comprehensive Feature Set

### 1. Storefront (B2C Customers)
*   **Bilingual Landing Page:** Visual hero banners, core value propositions, category navigation, featured items, and brands.
*   **Consumables Compatibility Search:** Select Brand $\rightarrow$ Select Printer Family $\rightarrow$ Select Printer Model $\rightarrow$ Instant list of compatible toner, ink, and accessories.
*   **Full-Text Search:** Custom Postgres FTS queries using `simple` parser configuration (works for bilingual terms without stemming conflicts).
*   **Robust Catalog Filters:** Filter products by brand, price range, stock availability, category, and authentication status (Original vs. Compatible).
*   **Shopping Cart Stock Hold:** Employs a **15-minute soft reservation lock** on inventory when products are added to the cart, preventing overselling.
*   **B2C Passwordless Authentication:** Fully passwordless authentication using mobile phone numbers and verification codes sent via WhatsApp OTP.

### 2. B2B Corporate Portal
*   **Application Pipeline:** Egyptian companies apply with their Commercial Registry (CR) numbers and Tax Card details.
*   **Admin Review Queue:** Applications transition through states (`Pending` $\rightarrow$ `Approved` or `Rejected`) with full audit logs.
*   **B2B Client Tiers:** Assign accounts to standard discount tiers: **Tier A (10% off)**, **Tier B (15% off)**, or **Tier C (Negotiated custom pricing per SKU)**.
*   **Flexible Checkout Terms:** Enable different policies per company, including standard credit terms (Net-15, Net-30) and payment method permissions:
    *   **SFR (Submit for Review):** Submit order without checkout payment for manual validation and sales rep routing.
    *   **SFR & Pay Now:** Let companies choose between card payment or submitting on credit terms.
    *   **Pay Now Only:** Force upfront payment.
*   **Corporate Dashboards:** Centralized views for company order history, billing invoices, and current credit terms.

### 3. Back-Office Admin Panel (`/admin`)
*   **Sales Metrics Dashboard:** Visual data charts showing sales, top-selling items, order volume, and pending tasks.
*   **Catalog & Brand Management:** Interactive dashboard for CRUD operations on brands, categories, products, and printer compatibility mapping.
*   **Advanced Inventory Logs:** Log stock movements, record adjustments, view inventory reservations, and perform bulk price updates.
*   **Order Operations Center:** Monitor order statuses, handle shipping allocations, and print invoices.
*   **Courier Allocation Interface:** Assign couriers to orders, configure custom delivery parameters, print waybills, and record tracking tokens.
*   **Admin User Management:** Set roles for staff members: `Owner`, `Ops Manager`, and `Sales Representative` with tailored system access limits.
*   **System Controls:** Toggles for configuring active shipping zones, base COD fees, VAT rates, and payment methods.

---

## 🗄 Database Architecture & Key Entities

The PostgreSQL database is organized into logical functional modules containing over 30 tables:

```
                  ┌───────────────────────┐
                  │         User          │
                  │ (Role: Admin/B2B/B2C) │
                  └──────────┬────────────┘
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
   ┌─────────────────┐               ┌─────────────────┐
   │     Company     │               │     Address     │
   │ (B2B Accounts)  │               │ (Egyptian Govs) │
   └────────┬────────┘               └────────┬────────┘
            │                                 │
            │     ┌─────────────────────┐     │
            ├────►│        Order        │◄────┘
            │     │(B2C/B2B Transaction)│
            │     └──────────┬──────────┘
            ▼                │
   ┌─────────────────┐       │       ┌─────────────────┐
   │   PricingTier   │       ├──────►│    OrderItem    │
   │ (Tier overrides)│       │       └────────┬────────┘
   └─────────────────┘       ▼                ▼
                  ┌─────────────────┐┌─────────────────┐
                  │     Invoice     ││     Product     │
                  │ (Gapless annual)││ (Bilingual FTS)│
                  └─────────────────┘└─────────────────┘
```

### Module Summary
1.  **Identity & Permissions (`User`, `Session`, `WhatsAppOtp`, `AdminInvite`, `RateLimit`):** Tracks custom session keys, one-time-passcode expiry data, and API rate-limiting thresholds.
2.  **Product Catalog (`Product`, `ProductImage`, `Brand`, `Category`, `PrinterModel`, `ProductCompatibility`):** Translates name and description strings dynamically. Tracks category hierarchies and many-to-many printer compatibility indices.
3.  **Commerce Transactions (`Order`, `OrderItem`, `OrderStatusEvent`, `Invoice`, `Cart`, `CartItem`):** Tracks order timelines, records line-item snapshotted prices, and generates gapless invoice serial numbers.
4.  **Inventory (`Inventory`, `InventoryReservation`, `InventoryMovement`):** Monitors current warehouse quantities, active holds (15-min soft holds), and historical stock audit trails.
5.  **B2B Pipelines (`Company`, `B2BApplication`, `CompanyPriceOverride`):** Links approved companies to specific catalog prices and tracks company registry details.
6.  **Shipping & Geography (`Address`, `ShippingZone`, `GovernorateZone`, `Courier`):** Manages shipping rates by regional zone and maps the 27 Egyptian governorates to appropriate carrier rates.

---

## 🚀 Local Development & Quick Start

Set up a fully functional development environment on your local machine in under 30 minutes.

### 📋 Prerequisites
*   **Node.js 22 LTS** (minimum `20.10.x`)
*   **Docker Desktop** (or a local instance of PostgreSQL 16)
*   **Git**

---

### 🛠 Installation & Execution Steps

#### 1. Clone the Repository & Install Dependencies
```bash
git clone https://github.com/Omar-Anwar-Dev/print-by-falcon.git
cd print-by-falcon
npm ci
```

#### 2. Configure Environment Variables
Copy the template variables file:
```bash
cp .env.example .env.local
```
Open `.env.local` and configure your local keys:
*   Set a strong 32-byte string for `AUTH_SECRET` (generate using `openssl rand -base64 32`).
*   Set a password for bootstrapping in `OWNER_TEMP_PASSWORD` (used on first admin setup).

#### 3. Spin Up the Database Container
Run a local PostgreSQL instance via Docker:
```bash
docker run -d \
  --name pbf-pg-local \
  -e POSTGRES_USER=pbf \
  -e POSTGRES_PASSWORD=pbf \
  -e POSTGRES_DB=pbf_dev \
  -p 5432:5432 \
  postgres:16-alpine
```

#### 4. Run Migrations and Seed Database
Run Prisma migrations to initialize the database structure and load the seeds:
```bash
npx prisma migrate dev --name init
npx prisma db seed
```
> [!NOTE]
> The database seed creates the owner account (`support@printbyfalcon.com`) using `OWNER_TEMP_PASSWORD`. On your first login at `/admin/login`, the system will require you to change this password.

#### 5. Start Development Servers
Start the main Next.js development server:
```bash
npm run dev
```

In a separate terminal window, start the pg-boss background worker:
```bash
npm run worker
```

---

### 🌐 Key Local Access Points

*   **Storefront (Arabic):** [http://localhost:3000/ar](http://localhost:3000/ar)
*   **Storefront (English):** [http://localhost:3000/en](http://localhost:3000/en)
*   **B2C OTP Login:** [http://localhost:3000/ar/sign-in](http://localhost:3000/ar/sign-in) *(Dev mode writes the OTP code to the server terminal)*
*   **B2B Company Login:** [http://localhost:3000/ar/login](http://localhost:3000/ar/login)
*   **Admin Portal Login:** [http://localhost:3000/ar/admin/login](http://localhost:3000/ar/admin/login)
*   **Liveness Check Endpoint:** [http://localhost:3000/api/health](http://localhost:3000/api/health)

---

## 📋 Scripts Reference Cheat Sheet

| Command | Action | Environment |
| :--- | :--- | :--- |
| `npm run dev` | Starts the Next.js development server. | Local Development |
| `npm run worker` | Starts the background worker locally. | Local Development |
| `npm run build` | Builds the optimized Next.js app bundle. | Production Build |
| `npm run start` | Serves the built production bundle locally. | Production / Staging |
| `npm test` | Runs the full Vitest unit test suite. | Local / CI |
| `npm run test:e2e` | Runs Playwright integration tests. | Local / CI |
| `npm run lint` | Checks codebase code styles via ESLint. | Quality Check / CI |
| `npm run typecheck`| Runs the TypeScript compiler check. | Quality Check / CI |
| `npm run format` | Standardizes codebase spacing via Prettier. | Quality Check |
| `npm run prisma:studio`| Opens a GUI browser to review database rows. | Local Development |
| `npm run prisma:migrate`| Applies schema migrations and updates the client.| Local Development |
| `npm run prisma:seed` | Runs the custom database seeding operations. | Local / Setup |

---

## 🔑 Environment Variables Guide

| Variable | Default Value (Dev) | Description & Security Guidelines |
| :--- | :--- | :--- |
| `DATABASE_URL` | `postgresql://pbf:pbf@localhost:5432/pbf_dev` | Connection URI. Keep hidden in production. |
| `AUTH_SECRET` | *(Must generate 32-byte)* | Secret key used to sign browser session cookies. |
| `OTP_DEV_MODE` | `true` | When `true`, WhatsApp API calls are bypassed and logged to the terminal. |
| `WHATSAPP_CLOUD_API_TOKEN`| `""` | Bearer token for the Whats360 WhatsApp Gateway API. |
| `SENTRY_DSN` | `""` | GlitchTip instance DSN endpoint for tracking errors. |
| `OWNER_EMAIL` | `support@printbyfalcon.com` | Email address used for the main administrator account. |
| `OWNER_TEMP_PASSWORD` | `AdminTemporaryPass123!` | Initial password used to bootstrap the seed administrator account. |
| `NEXT_PUBLIC_APP_URL` | `http://localhost:3000` | Root URL. Used for generating canonical SEO tags and webhooks. |
| `PAYMOB_API_KEY` | `""` | Secret integration key provided in your Paymob dashboard. |

---

## 📂 Project Directory Layout

```
D:\PrintByFalcon\
├── app/                        # Next.js App Router Structure
│   ├── [locale]/               # Bilingual base routes (/ar and /en)
│   │   ├── account/            # Customer profile information pages
│   │   ├── admin/              # Admin back-office panel dashboard views
│   │   ├── b2b/                # B2B portal registrations and orders
│   │   ├── cart/               # Shopping cart details layout
│   │   └── checkout/           # Dynamic checkout forms (B2C & B2B)
│   ├── actions/                # 25 secure Server Action entry points
│   ├── api/                    # System webhooks, health checks, & API routers
│   └── invoices/               # react-pdf dynamic invoice layouts
├── components/                 # React UI Components
│   ├── admin/                  # 52 bespoke components for managing the store
│   ├── ui/                     # 8 atomic primitives configured with Tailwind
│   └── (site-level)            # Global site header, footer, & navigation bar
├── lib/                        # Business Logic & Infrastructure Bindings
│   ├── auth/                   # Custom session verify & auth guards
│   ├── payments/               # Paymob API payloads & verification logic
│   └── validation/             # Shared client/server validation models (Zod)
├── worker/                     # pg-boss Background Process Module
│   ├── index.ts                # Main script listening to database queues
│   └── jobs/                   # Individual background handlers
├── prisma/                     # Database Schema Models & Migration Files
├── messages/                   # i18n Translation Dictionaries (ar.json & en.json)
├── docker/                     # Compose files, Nginx rules, & Dockerfiles
├── scripts/                    # Backups, deployments, & catalog importer scripts
├── tests/                      # Testing Configuration
│   └── e2e/                    # Playwright end-to-end user journeys
└── docs/                       # Comprehensive documentation library
```

---

## 🔗 API & Server Actions Reference

To ensure top performance, Print By Falcon replaces REST endpoints with **Next.js Server Actions** for core state updates, while keeping standard API routes for external webhooks.

### 1. Key API Webhook Handlers
*   `POST /api/webhooks/paymob` — Processes payment status updates from Paymob. It verifies the payload integrity using HMAC signatures before marking orders as paid.
*   `POST /api/webhooks/whats360` — Captures inbound customer messages (such as `STOP` keywords) to manage notification opt-outs.
*   `GET /api/health` — Returns system status codes, serving as a health check endpoint for monitoring tools.

### 2. Primary Server Actions (`app/actions/`)
*   `b2cAuth.ts` $\rightarrow$ Enqueues verification OTP numbers, validates codes, and initializes client session cookies.
*   `order.ts` $\rightarrow$ Manages the checkout flow: it validates quantities, applies promo codes, calculates shipping fees, reserves inventory, and registers new orders.
*   `inventory.ts` $\rightarrow$ Admin tool for recording stock movements, adjusting quantities, and managing reserves.
*   `b2b.ts` $\rightarrow$ Manages B2B account applications, updates company tier discount levels, and configures credit limits.

---

## 🛡 Security Hardening

The application is built on a "defense-in-depth" design philosophy:

*   **Custom Session Layer (ADR-021):** Replaced standard Auth.js with a custom, database-backed session table. Only the secure SHA-256 hash of the session token is stored in the database, protecting against token leaks. Cookies use `HttpOnly`, `Secure`, and `SameSite=Lax` flags.
*   **OTP Security Protection:** OTPs are stored in the database as SHA-256 hashes, set to expire in 5 minutes, limit logins to a maximum of 3 attempts, and rate-limited to 3 requests per phone number every 30 minutes.
*   **Origin Enforcement:** Next.js Server Actions enforce origin verification by default. Webhook handlers verify signature keys using HMAC before processing payload data.
*   **Production Safeguards:** The app includes startup checks that halt boot operations in production if test modes (such as `OTP_DEV_MODE`) are left enabled.
*   **Infrastructure Isolation:** The VPS uses UFW firewall rules set up to block all direct public ports, routing external traffic exclusively through Cloudflare IP addresses.

---

## 🧪 Testing & Quality Assurance

Print By Falcon uses a test suite to prevent logic errors across pricing configurations, inventory holds, and checkout states.

### 1. Unit Testing (Vitest)
Contains over 200 unit tests focused on core business calculations:
```bash
# Run unit tests
npm test
```

### 2. Integration Testing (Playwright)
Validates complete user journeys (e.g., checkout flows, B2B registrations, admin actions) using headless browser environments:
```bash
# Execute Playwright E2E suites
npm run test:e2e
```

---

## 🏗 Deployment & Operations

The application is deployed using a Docker multi-container setup managed via Docker Compose on the VPS.

### 1. Build and Run via Compose (Production)
```bash
docker compose -f docker/docker-compose.prod.yml up -d --build
```
This spins up four isolated services:
1.  `pbf-app`: Next.js web application server.
2.  `pbf-worker`: pg-boss background worker listener.
3.  `pbf-postgres`: Production PostgreSQL database.
4.  `pbf-nginx`: Secure reverse proxy handling SSL certificates and static asset caching.

### 2. Operational Backups
A database backup script is configured to run daily as a cron job, saving encrypted database dumps to backup storage:
```bash
# Run database backup script
bash scripts/backup.sh
```

---

## 📚 Documentation Library Index

For detailed guidelines and technical notes on specific components, refer to the following documentation files:

### Core Framework & Architecture
*   [Product Requirements (PRD.md)](docs/PRD.md) — Product specifications, scope boundaries, and development roadmap.
*   [Architecture Details (architecture.md)](docs/architecture.md) — Network topology, container setups, and performance strategy.
*   [Decision Records (decisions.md)](docs/decisions.md) — Log of 33+ Architectural Decision Records (ADRs).
*   [Progress History (progress.md)](docs/progress.md) — Sprint achievements, timelines, and milestones.

### System Runbooks
*   [Server Operations (runbook.md)](docs/runbook.md) — VPS setups, Docker deployment instructions, and database restore steps.
*   [Admin Dashboard Guide (admin-guide.md)](docs/admin-guide.md) — Product listings, inventory updates, and order workflows.
*   [B2B Customer Guide (b2b-user-guide.md)](docs/b2b-user-guide.md) — Partner applications, billing terms, and credit guidelines.
*   [Inventory Operations (inventory-ops-guide.md)](docs/inventory-ops-guide.md) — Warehouse stock audits and reservation settings.
*   [Returns Workflow (returns-workflow.md)](docs/returns-workflow.md) — Credit notes, refunds, and damaged stock returns.

### Configuration & Settings
*   [Settings Reference (settings-reference.md)](docs/settings-reference.md) — Database keys and schema defaults.
*   [Key Rotations (paymob-key-rotation.md)](docs/paymob-key-rotation.md) — Safe rotation of Paymob API keys.
*   [WhatsApp Gateway keys (whats360-key-rotation.md)](docs/whats360-key-rotation.md) — Safe rotation of Whats360 API tokens.
*   [Audit Logs Reference (audit-log-queries.md)](docs/audit-log-queries.md) — PostgreSQL queries for tracking admin events.

---

## 🤝 Contributing & Maintenance

*   **Main Branch Workflow:** Code merges are made directly to the `main` branch through Pull Requests.
*   **Mandatory Checks:** All PRs must successfully pass ESLint styling checks, TypeScript builds, and Vitest/Playwright tests before merging.
*   **Updating Documentation:** When implementing changes that affect database structures or API endpoints, update the corresponding documentation files in the `docs/` folder and log new system decisions as ADR entries in `docs/decisions.md`.

---

## ✉️ Contact & Support

*   **System Owner & Developer:** Omar
*   **Email Support:** `support@printbyfalcon.com`
*   **Production Site URL:** [printbyfalcon.com](https://printbyfalcon.com)
*   **Monitoring Portal:** [errors.printbyfalcon.com](https://errors.printbyfalcon.com) *(Self-hosted GlitchTip dashboard)*