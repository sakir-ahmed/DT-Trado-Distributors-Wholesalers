# DT Trado

**Sales, stock and collection system for distributors and wholesalers in Bangladesh, with an offline-first salesman app.**
Developed by **Datatech BD Ltd**, Dhaka.

DT Trado gives the owner clear answers to: who owes me money, which salesman collected cash, which customer bought what, which products are moving, and how much stock sits where.

```text
Purchase -> Warehouse -> Salesman -> Customer -> Invoice -> Collection -> Outstanding
```

Salesman app: **Customer -> Order -> Collection -> Receipt**. Owner: **Dashboard -> Sales -> Outstanding -> Stock**.

Built for FMCG, electronics, cosmetics, pharmaceutical, restaurant supply, stationery and construction material distributors. Bilingual (English and Bangla). Multi-tenant SaaS: many distributors on one platform.

> This project is built phase by phase from the specification in `docs/`. Start with `docs/roadmap.md`.

---

## Features

- **Products and pricing:** units and conversions (carton, dozen, piece), barcodes, price lists, schemes (buy 10 get 1), batch and expiry
- **Purchase and stock:** goods receipt, weighted average cost, multiple warehouses, van stock, transfers, stock counts, expiry control
- **Customers and routes:** credit limit and days, opening balances, routes and visit plans, location pins, Customer 360
- **Sales:** orders, invoices, challans, returns, credit notes, credit control (block, warn or approval)
- **Collection:** money receipts, cheques, mobile money (TrxID), cash custody and cashier settlement
- **Outstanding:** customer ledger, aging, statements, SMS reminders
- **Salesman app (PWA, offline-first):** day start and end, visits with check-in, order with barcode scan, collection and receipt sharing, returns, targets
- **Monitoring and incentives:** targets, leaderboard, day timeline, incentive statements (event-based location, no continuous tracking)
- **Delivery management:** trips, proof of delivery, collection on delivery
- **Owner dashboard and reports:** sales, collection, outstanding, stock, margin, salesman performance
- **Platform console (Datatech):** tenants, plans, trials, SMS credits, import monitor
- **Mobile-ready API:** documented, versioned `/api/v1` with sync endpoints for future native apps

Full scope and business rules: `docs/prd.md`.

---

## Tech Stack

| Area                  | Technology                                                               |
| --------------------- | ------------------------------------------------------------------------ |
| Backend               | Laravel 13.x, PHP 8.3+                                                   |
| Database              | MySQL 8 (shared database, `tenant_id` isolation)                         |
| Cache, session, queue | Redis                                                                    |
| Frontend              | Inertia.js v2, Vue 3, Tailwind CSS v4, shadcn-vue                        |
| Salesman app          | Installable PWA (Workbox, IndexedDB with Dexie), camera barcode scanning |
| Icons, charts, maps   | Font Awesome Free, Chart.js, Leaflet                                     |
| Auth and roles        | Laravel Sanctum (device-bound tokens), Spatie Permission (teams)         |
| Audit and backup      | Spatie Activitylog, Spatie Backup                                        |
| PDF, import           | spatie/laravel-pdf (Browsershot), Laravel Excel                          |
| SMS                   | BdBulkSMS with prepaid credit wallet                                     |
| Testing and quality   | Pest, Playwright (offline tests), Laravel Pint, Larastan                 |
| Containers            | Docker and Docker Compose                                                |

---

## Documentation

AI coding agents read `CLAUDE.md` first.

| File                   | Purpose                                                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `docs/prd.md`          | Product requirements: roles, modules, business rules, plans, scope                                                       |
| `docs/architecture.md` | Stack, structure, coding rules, multi-tenancy, stock, credit and collection integrity, offline rules, Definition of Done |
| `docs/api.md`          | API conventions, endpoints, salesman app endpoints, offline sync                                                         |
| `docs/design.md`       | Design system, sales desk, salesman app, documents, icons, sizing, tables                                                |
| `docs/roadmap.md`      | Build phases with acceptance criteria                                                                                    |
| `docs/ui-preview.html` | Static visual mockup of the intended UI (open in a browser)                                                              |

---

## Getting Started

### Requirements

- Docker and Docker Compose, Git
- Node.js 20+ and npm (only if you run the frontend outside Docker)

### Setup

```sh
git clone <repository-url> dt-trado
cd dt-trado

cp .env.example .env
# Edit .env: database, Redis, mail, BdBulkSMS, map tile provider

docker compose up -d --build
docker compose exec app composer install
docker compose exec app php artisan key:generate
docker compose exec app php artisan migrate --seed
docker compose exec app npm install
docker compose exec app npm run dev
```

The service name `app` follows the Docker Compose file created in Phase 0; adjust if yours differs.

The platform administrator is created by the seeder from `.env` (never commit real values):

```text
PLATFORM_ADMIN_NAME=
PLATFORM_ADMIN_EMAIL=
PLATFORM_ADMIN_PASSWORD=
```

For local development the seeder also creates two demo distributors so tenant isolation can be tested. The salesman app is served at `/app`; open it in Android Chrome and use "Add to Home screen".

### Background processes

| Process                                                              | Command                     |
| -------------------------------------------------------------------- | --------------------------- |
| Queue worker                                                         | `php artisan queue:work`    |
| Scheduler (alerts, reminders, reconciliation, trial expiry, backups) | `php artisan schedule:work` |

---

## Development Workflow

```sh
php artisan test            # or ./vendor/bin/pest
./vendor/bin/pint           # code style
./vendor/bin/phpstan        # static analysis
php artisan migrate:fresh --seed
```

Rules:

- Build one roadmap phase at a time; finish its acceptance criteria and the Definition of Done in `docs/architecture.md`.
- Every feature has Pest tests, including authorization, branch scope, and **cross-tenant isolation**.
- Stock changes only through stock movements; customer balances only through ledger entries; reconciliation tests must pass.
- Anything the salesman app uses must work offline and sync idempotently.
- Never put customer personal data in logs or URLs.
- No emojis anywhere. Icons come from Font Awesome through `AppIcon`.
- Every text key exists in both `lang/en` and `lang/bn`.

---

## Roles

Tenant roles: `owner`, `manager`, `supervisor`, `salesman`, `sales-officer`, `warehouse-keeper`, `delivery-man`, `accountant`. Platform role: `super-admin`. Customers do not log in in v1.

---

## Deployment Notes

- HTTPS only (required for the PWA, camera and location); `APP_ENV=production`, `APP_DEBUG=false`.
- Run the queue worker and scheduler under a process supervisor.
- The PHP image must include Chromium and the Inter and Noto Sans Bengali fonts for PDFs.
- Schedule encrypted daily backups and test a restore before onboarding real distributors.
- Run the nightly reconciliation (stock and ledger) and alert on any difference.
- Confirm VAT and invoice format (Mushak), location data notice for salesmen, and SMS sender ID rules with the client's accountant and a legal advisor.

---

## Support

Datatech BD Ltd, Dhaka, Bangladesh.

## License

Proprietary. All rights reserved by Datatech BD Ltd. Not for redistribution.
