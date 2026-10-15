# PRODUCT REQUIREMENTS: DT Trado

> **This is the file that changes for each product.** Keep the same section headings.

---

## 1. Vision

**DT Trado** is a cloud sales, stock and collection system for distributors and wholesalers in Bangladesh, with an offline-first mobile app for field salesmen. It answers the questions every distributor owner asks and cannot answer today:

- Who owes me money, and how old is it?
- Which salesman collected cash, and has it reached the cashier?
- Which customer bought what, and what did they stop buying?
- Which products are moving, and which are sitting dead?
- How much stock do I have, where is it, and what is about to expire?

The flow:

```text
Purchase -> Warehouse -> Salesman -> Customer -> Invoice -> Collection -> Outstanding
```

Salesman app: **Customer -> Order -> Collection -> Receipt**.
Owner: **Dashboard -> Sales -> Outstanding -> Stock**.

**Company:** Datatech BD Ltd, Dhaka.
**Target customers:** FMCG distributors, electronics, cosmetics, pharmaceuticals, restaurant suppliers, stationery, and construction material wholesalers. The first release is built around the FMCG and general wholesale pattern (many small shops, daily routes, credit sales, cash collection) because it is the largest group and the pain is sharpest. Other verticals are served by settings (batch and expiry, serial numbers, units) and later vertical packs.
**Languages:** English and Bangla (UI, documents, SMS).
**Salesman app:** an installable web app (PWA) for Android, offline-first, part of this project. Native iOS and Android apps are a separate future project that can use the same `/api/v1`.

---

## 2. Business Model & Deployment _(decision)_

**Multi-tenant SaaS.** One application and one shared database serve many distributors (tenants). Each tenant has one or more branches and warehouses. Isolation, export and security rules are in `architecture.md` sections 11 and 12.

> Change this only if you decide to sell dedicated installations for large distributors; that is a different deployment and pricing model.

### Pricing and plans _(suggested, editable in the platform console)_

| Plan    | Price per month (BDT) | Limits and modules                                                                                                                               |
| ------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Starter | about 2,000           | 1 warehouse, up to 3 salesmen, 3 office users; sales, stock, collection, outstanding, basic reports                                              |
| Growth  | about 6,000           | Up to 3 warehouses, 15 salesmen, 10 office users; routes and visit plans, targets, schemes, credit control, van stock, returns, batch and expiry |
| Pro     | about 15,000          | Unlimited warehouses, 50+ salesmen, supervisors, delivery management, advanced reports, API access, priority support                             |

Salesmen are the natural pricing unit for field-sales software; add an extra price per salesman if you prefer. SMS is sold as prepaid credits. These are business decisions; the code only needs plan limits and module toggles (`PlanGate`) to be configurable per tenant.

### Sales and onboarding flow

Your field salesperson visits distributors and sells a trial:

1. **Platform console** (Datatech only): create a tenant in minutes, set plan, trial dates and limits, add SMS credits.
2. **Setup wizard** for the tenant: company profile and logo, warehouses, units, categories, document number formats, credit policy.
3. **Import wizard:** products with units and prices, customers with **opening balances**, suppliers, opening stock (with batches and expiry), and price lists, from Excel.
4. **Install the salesman app** on each salesman's phone (QR link, login, register device).
5. **Trial:** 30 days with a visible banner; then subscribe. After the trial, 7-day read-only grace, then suspension; data kept for 90 days and exportable.

---

## 3. Roles

| Role               | Scope                            | Main needs                                                             |
| ------------------ | -------------------------------- | ---------------------------------------------------------------------- |
| `owner`            | All branches                     | Dashboard, outstanding, stock, profit, approvals, settings             |
| `manager`          | Assigned branch                  | Sales, purchase, stock, customers, approvals, reports                  |
| `supervisor`       | Own team of salesmen             | Team targets, approvals, visit monitoring, conflicts                   |
| `salesman`         | Own customers, routes, van stock | The mobile app: visits, orders, collections, returns                   |
| `sales-officer`    | Branch                           | Back-office order and invoice entry, customer service                  |
| `warehouse-keeper` | Assigned warehouse               | Receive stock, pick, van issue and return, stock counts                |
| `delivery-man`     | Assigned deliveries              | Delivery list, proof of delivery, collection on delivery               |
| `accountant`       | Finance, all branches            | Collections, settlement, ledger, payables, expenses, financial reports |

Platform role: `super-admin` (Datatech staff), not tied to a tenant.
**Customers do not log in** in v1; they receive documents by SMS link or PDF. A customer self-service portal and ordering app is a later module.
Cost prices and margins are visible only with the `products.view-cost` permission (not salesmen or delivery staff).

---

## 4. Modules

### A. Tenants, Plans & Platform Admin (Datatech)

- Tenant CRUD, plan, trial dates, limits (salesmen, warehouses, users), module toggles, suspend and resume.
- Subscription records, SMS credit wallet, usage overview, trials ending soon, impersonation with a visible banner and audit log, tenant data export.

### B. Products, Units & Price Lists

- Products: code, barcode, name (English and Bangla), category, brand, supplier, **base unit and unit conversions** (carton, dozen, piece), MRP, trade price, purchase price, VAT setting, reorder level, optional **batch and expiry tracking**, optional serial tracking (later), image.
- **Price lists:** a default list plus lists by customer group, route or channel (retail, wholesale, institution), each with effective dates. Customer-specific price overrides are allowed with permission.
- Product import and barcode label printing.

### C. Suppliers, Purchase & Goods Receipt

- Suppliers with opening balances and contact details.
- Purchase orders, **goods receipt** (with batch, expiry, cost and quantity differences), purchase invoices, purchase returns.
- Landed cost entries (freight, unloading) allocated to receipts.
- **Supplier payables:** payments, allocation to purchase invoices, supplier ledger and aging, cheque and bank payments.

### D. Warehouses & Stock

- Multiple warehouses and depots; a **van warehouse per salesman**.
- Immutable stock movements (receipt, sale, return, transfer, van issue and return, adjustment, damage, expiry write-off, count difference); balances always equal the sum of movements.
- Transfers between warehouses with a document and receive confirmation; **van stock issue and return** at the start and end of the day with reconciliation.
- Stock count (cycle or full) with variance review and approval; adjustments need reason and approval above a limit.
- **Batch and expiry:** first-expiry-first-out allocation, near-expiry alerts, expired stock blocked from sale, write-off flow.
- Weighted average costing; stock valuation; stock ledger per product; reorder alerts and suggested purchase quantities.

### E. Customers, Routes & Visit Plans

- Customers: code, name, shop name, owner, mobile, address, **location pin**, photo, type or channel, customer group, price list, assigned salesman and route, **credit limit and credit days**, **opening balance**, status. Duplicate detection on name and mobile.
- Routes (a named list of customers) and visit plans: weekly schedule per salesman, visit day per customer and visit order; ad hoc visits allowed.
- Customer approval flow for customers created by salesmen (optional).
- Customer 360: profile, ledger, purchase history (what they buy and when they stopped), open invoices, visits, notes.

### F. Sales: Orders, Invoices, Deliveries & Returns

- Order sources: salesman app, back-office entry, phone. Statuses: `draft -> submitted -> approved -> (picked) -> delivered or invoiced`.
- Two modes per tenant (or per salesman): **order booking** (salesman books an order, delivery and invoice happen later from the warehouse) and **van sales** (salesman sells from van stock and invoices on the spot). Both can coexist.
- Invoices with discounts, **schemes**, VAT (if enabled), delivery charge; challan or delivery note; credit note for returns.
- **Schemes:** quantity-based free goods (buy 10 get 1), percentage or amount discounts on quantity or value, product or category based, valid between dates, for selected customer groups or routes. One `ApplySchemesAction` handles all of them and shows the benefit on the document.
- **Returns:** good returns go back to stock, damaged returns go to a damage stock location with a reason and photo; approval above a limit; credit note adjusts the customer ledger.
- **Delivery management** (Pro): delivery trips with a list of invoices, delivery man, proof of delivery (signature or OTP), failed delivery reasons, and cash collected on delivery.

### G. Credit Control

- Per customer: credit limit, credit days, and a **policy mode** per tenant: `block`, `warn`, or `approval` (a supervisor must approve).
- Checks on order and invoice: limit exceeded, overdue invoices above the allowed age, and customer on hold.
- Overrides need permission and a reason and appear in an approvals inbox (works from the supervisor's phone). Every override is logged.

### H. Collection, Receipts & Cash Settlement

- Salesman and office collection: cash, bKash or Nagad (with TrxID, verified later by the accountant), bank transfer, and **cheque** (number, bank, date, image). Allocation to invoices (oldest first or selected) and **advance** for unallocated amounts.
- **Money receipts** numbered per salesman or device series; shared from the phone as PDF or image and sent by SMS.
- **Cash with salesman:** collections stay in the salesman's custody until a cashier **settles**. Settlement shows expected versus counted cash by method and handles shortages with approval and a recoverable record.
- Cheque register: received, deposited, cleared, bounced (a bounce reverses the receipt and charges the customer ledger); post-dated cheque reminders.
- Receipt reversal with reason and approval; day-wise collection reports by salesman, route and method.

### I. Outstanding & Customer Ledger

- Immutable ledger per customer (invoices, receipts, credit notes, adjustments, opening balance); the balance is the sum of entries.
- **Outstanding report** by customer, salesman, route and group with **aging buckets** (current, 1-30, 31-60, 61-90, over 90), overdue amount, and promised payment date notes.
- Customer statements (PDF) individually or in bulk; overdue reminders by SMS with a credit estimate; follow-up notes and collection promises.

### J. Salesman Field App (PWA, offline-first)

- Login with device registration and PIN; downloads own customers, routes, price lists, schemes, products, van stock and open invoices.
- **Day:** start day (location, van stock), visit list with route order and map, check in and check out with location and outcome, end day with reconciliation.
- **Customer:** card with outstanding, overdue, available credit, last orders and top products; add a new customer with a location pin and photo.
- **Order and van sale:** product search and **barcode scan**, units, schemes and credit feedback, drafts, submit.
- **Collection:** record and share a money receipt.
- **Returns** with reason and photo; **targets and incentives** progress; notices from the office.
- Works fully offline; syncs in the background with a visible pending counter; conflicts are reported to the supervisor (rules in `architecture.md` section 16).
- Location is captured only at defined events; there is no continuous tracking in v1.

### K. Targets, Incentives & Salesman Monitoring

- Monthly targets per salesman, supervisor and optionally per product group: sales value, collection value, new customers, visits, productive visit rate.
- Target versus achieved views, daily and monthly; leaderboard.
- **Incentive rules:** a percentage of collection or sales above a target, slab based, with a monthly incentive statement (computed from posted data, approved and locked).
- **Monitoring** (event based): day start and end times, check-in map, visit duration, orders and collections per visit, visits outside planned routes, skipped customers, and unsettled cash. No continuous tracking in v1.

### L. Owner Dashboard

- Sales, collection, outstanding, overdue, stock value, gross margin, new and lost customers, top and slow products, salesman leaderboard, collection rate, unsettled cash, near-expiry stock, stock-out risks, pending approvals.
- Filters by branch, salesman, route, category, and period; every KPI drills into a report that matches exactly.

### M. Reports

Sales (by product, customer, category, salesman, route, period), purchase, stock position, stock ledger, stock valuation, fast and slow movers, expiry, outstanding and aging, collection, salesman performance, target versus achieved, customer purchase trend, price list, scheme usage, returns and damage, supplier payables, profit and margin (with permission), expenses, VAT summary. Export to PDF and Excel.

### N. Expenses & Basic Accounts

- Expense entry by category and branch (fuel, salary, rent, van running cost, other) with attachments; salesman expense claims from the app (optional).
- Cash book and bank book, payables and receivables summaries, and a simple profit and loss summary.
- Full double-entry accounting is out of scope for v1 (planned as a later module).

### O. Notifications & SMS

- SMS (BdBulkSMS) with prepaid credits: order confirmation, delivery notice, receipt, overdue reminder, owner daily summary; bilingual templates with preview and segment counter; delivery log; per-event switches.
- In-app notifications and approval requests; optional push notifications from the app later.

### P. Data Import, Onboarding & Mobile-Ready API

- Import wizard: products, units, price lists, customers with opening balances, suppliers, opening stock with batches; dry run, row-level errors, safe re-run, summary.
- Everything the app needs is available through `/api/v1` (see `api.md`) with auto-generated OpenAPI documentation, stable versioning, and sync endpoints for offline clients.

---

## 5. Business Rules

| Topic                      | Rule                                                                                                                                                                                                           |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Stock**                  | Stock changes only through immutable movements. Balance equals the sum of movements. Negative stock is blocked by default and configurable per warehouse.                                                      |
| **Units**                  | Quantities are stored in the product's base unit; documents show the chosen unit. Conversion factors are fixed per product and cannot change once movements exist (create a new unit instead).                 |
| **Costing**                | Weighted average cost, recalculated at each receipt. A sale line stores the cost at the time of sale. Returns use the original sale cost.                                                                      |
| **Batch and expiry**       | When enabled, issues use first expiry first out; expired batches cannot be sold; near-expiry window is configurable (default 90 days).                                                                         |
| **Pricing**                | The customer's price list applies at order time and is stored as a snapshot on the line. Price changes never alter existing documents. Overrides are limited by role and logged.                               |
| **Schemes**                | Applied automatically by validity, customer group and route; free goods are issued from stock as zero-price lines linked to the scheme.                                                                        |
| **Credit**                 | `block`, `warn` or `approval` mode. The check uses the limit, the current balance including unposted orders, credit days and overdue age. Overrides need permission and a reason.                              |
| **Documents**              | Posted invoices, receipts and credit notes are immutable. Corrections use credit notes or reversals with a reason and approval. Voided numbers remain visible.                                                 |
| **Numbering**              | Sequential per series (branch, salesman or device) without gaps; salesman devices have their own series so offline documents never collide. Confirm legal invoice requirements with the client's accountant.   |
| **Allocation**             | Receipts are allocated to invoices (oldest first by default) as stored rows. Unallocated money becomes an advance. The balance is calculated, never typed.                                                     |
| **Cash custody**           | A collection belongs to the salesman until a cashier settles it. Settlement handles shortage and excess with approval; the system never hides a difference.                                                    |
| **Cheques**                | Status flow: received, deposited, cleared or bounced. A bounced cheque reverses the receipt and records a charge.                                                                                              |
| **Returns**                | Good items return to stock; damaged items go to a damage location. A return needs a reason, a reference to the original invoice (or approval if none), and approval above the limit.                           |
| **Targets and incentives** | Computed from posted documents only. An approved incentive statement is locked.                                                                                                                                |
| **Offline sync**           | The server is the source of truth. Every queued item has a client-generated ID, so a retry never duplicates. Collections are always accepted; stock, credit and price conflicts are recorded for a supervisor. |
| **Location**               | Recorded at work events only, shown to permitted roles, kept for 12 months by default, with a visible notice in the app.                                                                                       |
| **Cost visibility**        | Cost prices and margins are hidden unless `products.view-cost` is granted.                                                                                                                                     |
| **VAT**                    | Off by default; configurable per tenant and product. Confirm VAT and Mushak invoice requirements with the client's accountant.                                                                                 |
| **Records**                | No hard deletes of transactions. Master data is deactivated, not deleted.                                                                                                                                      |
| **Tenant isolation**       | No query may return another tenant's data. Tested for every model and endpoint.                                                                                                                                |
| **Plan limits**            | Salesmen, warehouses, users and modules are enforced centrally with a clear message and an upgrade path.                                                                                                       |

---

## 6. Hardware

- Android phones (mid-range) for salesmen; desktop or laptop for the back office.
- Optional: A4 laser printer for invoices, 80mm thermal printer for counter receipts, barcode scanner (USB keyboard wedge) in the warehouse, portable Bluetooth printer for van sales (optional, support depends on the device).
- The app also scans barcodes with the phone camera.

---

## 7. Non-Functional Requirements

- **Localization:** UI, documents and SMS in `en` and `bn`.
- **Offline first:** the salesman app opens and works with no connection for a full day; sync resumes automatically; no data is lost if the browser is killed mid-order.
- **Performance:** salesman app opens in under 3 seconds from cache on a mid-range phone; product search under 200 ms offline with 20,000 products; invoice save under 1 second; outstanding report under 3 seconds for 20,000 customers; sync of a full day (about 60 documents) under 30 seconds on 3G.
- **Scale targets:** a tenant with 50,000 products, 20,000 customers and 100 salesmen, with 5 years of data, remains responsive using indexes and aggregated tables.
- **Security:** HTTPS only, device-bound tokens, PIN lock, private file storage, access logs for exports, rate limits, idle timeout, encrypted backups, tenant isolation.
- **Audit:** activity log for price and discount overrides, credit overrides, stock adjustments, receipt reversals, cheque status changes, settlement differences, and role changes.
- **Availability:** target 99.5% monthly; planned maintenance outside working hours.
- **Browsers and devices:** latest Chrome on Android for the salesman app; latest Chrome and Edge on desktop.

---

## 8. Screen List (per role)

| Role                  | Screens                                                                                                                         |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| owner                 | Dashboard, outstanding and aging, stock position, sales analysis, salesman performance, profit, approvals, settings, access log |
| manager               | Orders, invoices, deliveries, purchase, stock, customers and routes, targets, approvals, reports                                |
| supervisor (mobile)   | Team dashboard, visit monitoring, approvals inbox, sync conflicts, target progress                                              |
| salesman (mobile)     | Home, visits, customer card, order, collection, returns, van stock, targets, end of day                                         |
| sales-officer         | Order and invoice entry, customer search, collection desk, customer statements                                                  |
| warehouse-keeper      | Goods receipt, pick lists, van issue and return, transfers, stock count, expiry list                                            |
| delivery-man (mobile) | Delivery list, proof of delivery, collection on delivery                                                                        |
| accountant            | Settlement, collections, cheque register, supplier payables, ledger, expenses, financial reports                                |
| platform (Datatech)   | Tenants, plans and trials, usage, SMS credits, impersonation, import monitor                                                    |

---

## 9. Out of Scope

- Customer self-service ordering app and B2B portal (later module).
- Native iOS and Android apps (separate project).
- Continuous background GPS tracking and route replay.
- Direct mobile money gateway integration for collections (TrxID entry with verification is included).
- Full double-entry accounting, payroll, and NBR e-filing.
- Manufacturing, retail POS, multi-currency, and franchise group reporting.
- Third-party delivery or courier API integrations, route optimization, and fleet telematics.
- Serial-number tracking and warranty management (planned for the electronics vertical pack).
- Pharmaceutical regulatory features such as drug license checks (a later vertical pack; batch and expiry are included).

---

## 10. Key User Stories & Acceptance Criteria

1. **Day start.** _As a salesman, I start my day._
   - Done when: the app loads my route, customers, prices and van stock; start-day records time and location; it works with no internet after the first sync.
2. **Order in a shop.** _As a salesman, I book an order._
   - Done when: a 10-line order is taken in under 2 minutes, barcode scan adds the product, schemes and credit warnings appear before submit, and it saves offline.
3. **Credit control.** _As an owner, I stop bad credit._
   - Done when: an order over the limit is blocked, warned or sent for approval according to the tenant mode, the supervisor can approve from the phone, and the override is logged.
4. **Collection.** _As a salesman, I collect money._
   - Done when: a collection is allocated to the oldest invoices (or the selected ones), a receipt is shared as PDF and SMS, it works offline with the salesman's own number series, and the customer outstanding updates.
5. **Cash settlement.** _As a cashier, I settle a salesman._
   - Done when: expected versus counted cash is shown by method, a shortage needs approval and becomes a recoverable record, and the cash with salesman balance returns to zero.
6. **Outstanding.** _As an owner, I see who owes me._
   - Done when: the outstanding report shows aging buckets by customer, salesman and route, totals equal the customer ledger, and bulk reminders show segment and credit cost first.
7. **Invoice and delivery.** _As an officer, I invoice a booked order._
   - Done when: stock is issued by FEFO (when batches are on), the invoice and challan print, the customer ledger posts, and the stock balance equals the sum of movements.
8. **Van stock.** _As a warehouse keeper, I issue and receive back van stock._
   - Done when: the issue and return documents are scanned, differences are flagged, and the day cannot be closed with an unexplained difference.
9. **Returns.** _As a salesman, I record a return._
   - Done when: good returns add stock, damaged returns go to the damage location, a credit note adjusts the ledger, and approval is required above the limit.
10. **Purchase.** _As a manager, I receive goods._
    - Done when: a goods receipt with batch and expiry updates stock and the weighted average cost, differences against the purchase order are shown, and the supplier payable posts.
11. **Expiry.** _As a manager, I avoid expiry losses._
    - Done when: batches near expiry appear on the dashboard and in alerts, expired batches cannot be sold, and a write-off needs a reason and approval.
12. **Targets and incentives.** _As a supervisor, I see my team._
    - Done when: target versus achieved updates after sync, the leaderboard matches posted data, and the monthly incentive statement can be approved and locked.
13. **Monitoring.** _As an owner, I see what my salesmen did._
    - Done when: the day timeline shows check-ins on a map with visit duration, outcomes, orders and collections, with no tracking outside work events.
14. **Import.** _As Datatech support, I digitize a distributor in one day._
    - Done when: a dry run reports row errors without saving, the real import is repeatable without duplicates, and opening balances and opening stock appear in the ledger and stock reports.
15. **Offline reliability.** _As a salesman, the app never loses my work._
    - Done when: orders and collections taken offline sync without duplicates after reconnecting, killing the browser loses nothing, and conflicts appear on the supervisor's screen.
16. **Reconciliation.** _As an owner, my numbers agree._
    - Done when: sales, collection and outstanding on the dashboard equal the reports and the ledger, and nightly reconciliation shows no stock or ledger differences.
17. **Subscription and trial.** _As Datatech, I run a trial and bill the tenant._
    - Done when: a banner shows days left, limits (salesmen, warehouses) are enforced, expiry goes to read-only for 7 days and then suspension, and data can be exported.
18. **Tenant isolation.** _As a distributor, my data is never visible to another distributor._
    - Done when: automated tests prove that every model and endpoint returns 404 across tenants.

---

## 11. Phasing

See `roadmap.md`. Build phase by phase; do not implement later phases early. The first sellable release is Phases 0 to 8.
