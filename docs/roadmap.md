# ROADMAP: DT Trado

> The AI builds **one phase at a time**, runs tests, and stops. Each phase must satisfy the Definition of Done in `architecture.md`.
> **First sellable release = Phases 0 to 8** (back office plus the salesman app with offline sync, credit control, collection and settlement). Phases 9 to 13 are upgrades for Growth and Pro plans.

Mark finished phases with `[x]`.

---

## Phase 0: Foundation, Tenancy & UI Kit

- [ ] Docker Compose (nginx, php-fpm with Chromium, mysql, redis, queue, scheduler), `.env.example`
- [ ] Multi-tenancy: `Tenant`, `ResolveTenant`, `BelongsToTenant`, `BelongsToBranch`, tenant-aware jobs, prefixed cache and storage
- [ ] Architecture test that fails when a tenant-owned model lacks the trait
- [ ] Inertia + Vue 3 + Tailwind v4 tokens (`design.md`), Sonner, dark mode, SSR on public pages only
- [ ] Localization middleware, `en` / `bn` files, language switcher
- [ ] Staff auth (username / email / mobile), OTP service, rate limits, idle timeout, device registration for salesmen
- [ ] Spatie Permission with teams, per-tenant role seeding, platform `super-admin`
- [ ] Activity log, access log for exports, encrypted backups, tenant export command
- [ ] Minimal `/platform` console: create tenant in under 5 minutes, plan, limits, trial dates, SMS credits
- [ ] `PlanGate` skeleton (limits, modules, trial and suspension rules)
- [ ] **Shared UI kit** from `design.md` section 22 with `AppIcon` and `NoEmojiTest` in CI
- [ ] OpenAPI docs generator at `/api-docs`
- [ ] PWA shell for `/app` (manifest, service worker, Dexie setup, status pill) with a placeholder screen that works offline
- [ ] A hidden `/admin/ui-kit` showcase page that renders every component in light, dark, `en` and `bn`

**Acceptance:** `migrate:fresh --seed` works; two seeded tenants cannot see each other's data (tested); each role sees only its menu; the UI-kit page renders; the `/app` shell installs on Android Chrome and opens offline.

## Phase 1: Company Setup, Products, Units, Price Lists & Import

- [ ] Setup wizard, branches, categories, brands, document number series
- [ ] Products with base unit and conversions, barcodes, images, VAT setting, reorder levels, batch and expiry flags
- [ ] Price lists (default and by customer group), effective dates, permission-protected cost price visibility
- [ ] Product, unit and price list import (dry run, row-level errors, safe re-run)
- [ ] Barcode label printing

**Acceptance:** unit conversions are exact (tested with carton, dozen and piece); a user without `products.view-cost` never receives cost data from any route or API (tested); importing the same file twice creates no duplicates.

## Phase 2: Warehouses, Stock Engine, Suppliers & Purchase

- [ ] Warehouses and depots, van warehouse per salesman
- [ ] `PostStockMovementAction`, stock balances under lock, stock ledger
- [ ] Suppliers, purchase orders, goods receipt with batch, expiry and cost, purchase returns, landed cost
- [ ] Weighted average costing, stock valuation
- [ ] Transfers, adjustments with reasons and approval, stock count with variance review, opening stock import
- [ ] Reorder alerts and suggested purchase

**Acceptance:** for every product and warehouse, balance equals the sum of movements (reconciliation test, including concurrency tests); weighted cost matches hand-calculated examples; negative stock is blocked unless allowed; FEFO allocation is correct.

## Phase 3: Customers, Routes, Salesmen & Visit Plans

- [ ] Customers (credit limit, days, opening balance, price list, location pin, photo), duplicate detection, customer approval flow
- [ ] Salesmen, supervisors, routes, weekly visit plans, targets setup
- [ ] Customer import with opening balances; customer ledger foundation (opening entries)
- [ ] Map view of customers (Leaflet) and Customer 360 page

**Acceptance:** a salesman can only access their own customers (tested); opening balances create ledger entries and match the imported total; duplicate warnings work.

## Phase 4: Sales Back Office (orders, invoices, challan, returns)

- [ ] Order, invoice and challan entry with the line grid from `design.md` section 6, price list application and snapshots
- [ ] `ApplySchemesAction` (free goods, discounts) and scheme management
- [ ] `CheckCreditAction` with `block`, `warn` and `approval` modes, approvals inbox
- [ ] Customer ledger postings, document numbering under lock, immutable documents with credit notes
- [ ] Sales returns (good and damaged), approval and stock effects
- [ ] Invoice, challan and credit note PDFs (A4, A5), in-words amounts, QR verification

**Acceptance:** concurrent invoices never duplicate numbers or oversell stock (tested); customer balance equals the sum of ledger entries (tested); scheme examples match hand calculations; a posted invoice cannot be edited.

## Phase 5: Collections, Cheques, Settlement & Outstanding

- [ ] Money receipts (cash, mobile money with TrxID, bank, cheque), allocation and advances, receipt reversal with approval
- [ ] Cheque register and status flow, bounce handling
- [ ] Cash custody per salesman, cashier **settlement** with shortage handling and approval
- [ ] Outstanding report with aging buckets, customer statements (single and bulk PDF), follow-up notes and promises
- [ ] Collection desk screen from `design.md` section 6

**Acceptance:** allocations plus advances equal collections exactly (tested); settlement differences are recorded, never hidden; aging totals equal ledger balances; a bounced cheque reverses its receipt correctly.

## Phase 6: Salesman App Core (online first)

- [ ] Login with device registration and PIN lock
- [ ] Home, start and end day, visit list, check in and out with location and outcome
- [ ] Customer card, order entry with search, **barcode scan**, units, schemes and credit feedback
- [ ] Collection with receipt sharing (PDF or image through the share sheet) and SMS receipt
- [ ] Returns with reason and photo, new customer form, targets progress

**Acceptance:** a 10-line order is taken in under 2 minutes on a mid-range phone; credit and scheme feedback matches the back-office results exactly; a salesman can never read another salesman's data (tested).

## Phase 7: Offline Sync

- [ ] Bootstrap and delta sync endpoints, scoped to each salesman
- [ ] Dexie database, outbox with `client_id`, background sync with retry, status pill and pending counter
- [ ] Per-device document number series for offline receipts and van invoices
- [ ] Conflict detection and recording, Sync Conflicts screen for supervisors
- [ ] Revoke and deactivate handling: the app clears local data on the next connection
- [ ] Playwright offline tests (offline day, kill the browser, reconnect)

**Acceptance:** a full day offline (orders, collections, new customers, returns) syncs without duplicates or loss; resending a batch changes nothing; conflicts are listed with clear reasons; collected money is always accepted.

## Phase 8: SMS, Dashboard, Reports & Targets

- [ ] SMS service with credit wallet, templates (English and Bangla), order, receipt and overdue messages, bulk reminders with cost estimate
- [ ] Owner dashboard: sales, collection, outstanding, overdue, stock value, margin, leaderboard, alerts
- [ ] Reports with PDF and Excel export following `design.md` section 17
- [ ] Targets, target versus achieved, leaderboard, unsettled cash alerts
- [ ] Salesman monitoring views (day timeline and map, visit outcomes)

**Acceptance:** dashboard totals equal report totals and ledger totals exactly (tested); bulk reminders show segments and credits first and fail safely at zero credits; the dashboard loads in under 3 seconds on a seeded tenant with 20,000 customers and 2 years of data. **This completes the first sellable release.**

## Phase 9: Van Sales, Van Stock & Warehouse Operations

- [ ] Van issue and return documents with scanning and differences, end-of-day reconciliation lock
- [ ] Van sale mode (immediate invoice from van stock), offline-capable
- [ ] Pick lists, loading lists, goods receipt discrepancies, mobile stock count with scanning
- [ ] Batch and expiry operations: near-expiry alerts, write-off flow, expiry report

**Acceptance:** a salesman's day cannot close with unexplained van stock or cash differences; van sales offline sync without double issuing; expired batches cannot be sold.

## Phase 10: Delivery Management

- [ ] Delivery trips, assignment to delivery men, mobile delivery screens
- [ ] Proof of delivery (signature or OTP), failed delivery reasons, partial delivery
- [ ] Cash collected on delivery and its settlement
- [ ] Delivery performance report

**Acceptance:** a delivery man sees only assigned deliveries (tested); partial delivery adjusts invoices through credit notes and stock correctly.

## Phase 11: Incentives, Expenses & Basic Accounts

- [ ] Incentive rules (slab, percentage), monthly statements, approval and lock
- [ ] Expenses and salesman expense claims, cash book and bank book
- [ ] Supplier payables aging, payments and allocation, payables report
- [ ] Profit and loss summary (permission protected)

**Acceptance:** incentive amounts match hand-calculated examples for every rule type; an approved statement cannot be edited; supplier ledger equals purchase invoices minus payments.

## Phase 12: Subscription, Billing & Onboarding Tools

- [ ] Platform console: plans, module toggles, limits, suspend or resume, usage, impersonation with audit
- [ ] Trial banner, 7-day read-only grace, suspension, scheduled retention
- [ ] Subscription records, SMS credit top-ups, per-salesman pricing option
- [ ] Onboarding checklist, import monitor, demo data, trials-ending list

**Acceptance:** limits for salesmen and warehouses are enforced everywhere (tested); expiry moves to read-only then suspension on schedule; importing a full dataset can be monitored and repeated safely.

## Phase 13: Polish, API & Hardening

- [ ] API review for future native apps, OpenAPI examples, contract tests that keep `v1` stable
- [ ] Security review (access logs, device binding, location data retention, encryption, backup restore drill), accessibility pass
- [ ] Performance check with 50,000 products, 20,000 customers and 100 salesmen
- [ ] User guide (English and Bangla), a salesman quick-start card, onboarding guide for Datatech staff, deployment guide

**Acceptance:** full test suite green, Pint and Larastan clean, a restore drill passes, nightly reconciliation shows zero differences on the performance dataset, and a new distributor can be set up and have its first order taken in the app within one day.

---

## Later Modules (not in this project scope)

Customer self-service ordering portal, native iOS and Android apps, continuous GPS tracking, mobile money gateway integration, full double-entry accounting, serial and warranty tracking (electronics pack), pharmaceutical compliance pack, route optimization.
