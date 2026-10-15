# ARCHITECTURE & CODING RULES: DT Trado

> Shared engineering rules for Datatech products, adapted for DT Trado (multi-tenant SaaS for distributors and wholesalers with an offline-first field-sales app). Sections 11 to 16 are specific to this product.
> If this file conflicts with the product's `prd.md`, **the PRD wins** for business rules and roles. This file wins for code structure.

---

## 1. Tech Stack

| Area                    | Choice                                                                                                                                                           |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Framework               | Laravel 13.x (latest stable), PHP 8.3+. Check with `composer show laravel/framework`.                                                                            |
| Database                | MySQL 8                                                                                                                                                          |
| Cache / Session / Queue | Redis (`CACHE_STORE`, `SESSION_DRIVER`, `QUEUE_CONNECTION` = `redis`)                                                                                            |
| Frontend                | Inertia.js v2, Vue 3 (`<script setup>`), Tailwind CSS v4, shadcn-vue (Radix Vue)                                                                                 |
| Toasts                  | `vue-sonner`                                                                                                                                                     |
| Auth                    | Staff and salesmen: username / email / mobile login with password; salesmen are bound to a registered device. Spatie Permission with **teams** (tenant as team). |
| Real-time               | Not required in v1. Use normal page refresh and polling to keep hosting cheap.                                                                                   |
| Audit                   | `spatie/laravel-activitylog`                                                                                                                                     |
| Backup                  | `spatie/laravel-backup` (daily, via scheduler)                                                                                                                   |
| Localization            | `en` and `bn` (`lang/en`, `lang/bn`)                                                                                                                             |
| SMS                     | BdBulkSMS via `SmsService` (section 5)                                                                                                                           |
| Testing                 | Pest                                                                                                                                                             |
| Code quality            | Laravel Pint, Larastan (level 5 minimum)                                                                                                                         |
| Containers              | Docker Compose: nginx, php-fpm (with Chromium), mysql, redis, queue worker, scheduler                                                                            |
| Icons                   | Font Awesome Free via `@fortawesome/vue-fontawesome` (tree-shaken SVG), wrapped in `AppIcon`                                                                     |
| Charts                  | `chart.js` + `vue-chartjs` (one library only)                                                                                                                    |
| Multi-tenancy           | One app, shared MySQL database, `tenant_id` on every tenant-owned table (section 11)                                                                             |
| PDF                     | `spatie/laravel-pdf` with the Browsershot (headless Chromium) driver                                                                                             |
| Barcode / QR            | Camera scanning in the browser (`BarcodeDetector` with a `@zxing/browser` fallback) for products and documents; `endroid/qr-code` for invoice verification codes |
| Offline / PWA           | `vite-plugin-pwa` (Workbox) + IndexedDB via Dexie (salesman app, delivery and warehouse screens)                                                                 |
| Maps                    | Leaflet with a tile provider from `.env` (customer locations, visit maps)                                                                                        |
| Excel import / export   | `maatwebsite/excel`                                                                                                                                              |
| API docs                | `dedoc/scramble` (OpenAPI at `/api-docs`)                                                                                                                        |
| Bot protection          | Cloudflare Turnstile on public booking and OTP forms                                                                                                             |

**Rendering**

- Public pages (`Web/` controllers): the Datatech product page and trial sign-up only, with Inertia SSR and dynamic meta tags.
- The admin panel (`/admin`) and the **salesman app (`/app`, installable PWA, offline-first)** are client-side only.
- Admin panel (`/admin`): client-side SPA, **SSR disabled**. No Filament.

---

## 2. Folder Structure

```text
app/
├── Actions/            # One class = one business operation (CreateOrderAction)
├── Services/           # External integrations + orchestration (SmsService, BkashService)
├── DataTransferObjects/# Only when an Action has 4+ inputs
├── Enums/              # OrderStatus, PaymentMethod ...
├── Http/
│   ├── Controllers/
│   │   ├── Admin/      # Inertia controllers for the SPA (POS included)
│   │   ├── Api/V1/     # JSON API (see api.md)
│   │   └── Web/        # Public SSR pages
│   ├── Requests/       # Form Requests (all validation lives here)
│   └── Resources/      # API Resources
└── Models/             # Relationships, scopes, casts, LogsActivity. No business logic.
```

**No Repository layer and no interfaces in v1.** Eloquent is the data layer. SMS has one provider and there are no payment gateways, so both are plain service classes. Add an interface only when there are two or more real implementations. Mobile money collections (bKash, Nagad) are recorded by the salesman with a TrxID and verified by the accountant; gateway integration is out of scope for v1.

---

## 3. Core Rules

1. **Thin controllers.** Validate (Form Request) → call an Action → return a response.
2. **Admin controllers and API controllers call the same Actions.** Never duplicate business logic.
3. **Strict types.** `declare(strict_types=1);`, typed parameters, return types and properties everywhere.
4. **No over-engineering.** No factories, abstractions or patterns without a concrete need today.
5. **Money:** store as `decimal(12,2)` in BDT. Never use floats. API returns money as strings (`"1250.00"`).
6. **Stock movements, invoices, receipts, allocations, settlements and sync batches run inside `DB::transaction()`.** Use `lockForUpdate()` on stock balances, document number counters and customer balances.
7. **Tenant and branch scoping:** every tenant-owned model uses `BelongsToTenant`, and branch-owned models also use `BelongsToBranch` (section 11). A user must never read another tenant's data, and non-owner users must never read another branch's data. Test both.
8. **All user-facing text uses translation keys** (`__('orders.created')`), never hard-coded strings.
9. **Secrets only in `.env`**, accessed via `config()`. Never call `env()` outside config files.
10. **No emojis anywhere** (UI, translations, SMS, receipts, seeders, comments, commits). Icons come from Font Awesome through `AppIcon`; see `design.md` section 23.
11. **Eager load relations** to avoid N+1. Add DB indexes on foreign keys and common filters (customer, product SKU and barcode, invoice date, salesman, route, document numbers).
12. **Field and business data rules apply everywhere** (section 12): no customer personal data in logs, URLs or error trackers, and location data is collected only at defined work events.
13. **Quantities** use `decimal(14,3)` in the product's base unit, with unit conversions applied at the edges (UI and documents). Prices and amounts follow rule 5.

---

## 4. Roles & Permissions

Spatie Permission with **teams enabled** (`teams => true`, team key `tenant_id`), so every role and assignment belongs to one tenant. Roles are defined in `prd.md` and seeded per tenant when the tenant is created. `super-admin` is a platform role (Datatech staff) that is not tied to a tenant.

Use permissions (`orders.approve`, `credit.override`, `discount.override`, `stock.adjust`, `collections.settle`, `products.view-cost`), not role names, in policies and `@can` checks. The root platform account comes from `.env` values (never hard-code a password).

---

## 5. Authentication, OTP & SMS

### Login (username, email or mobile)

```php
public function getCredentials(): array
{
    $login = trim($this->string('login')->toString());

    $field = match (true) {
        filter_var($login, FILTER_VALIDATE_EMAIL) !== false => 'email',
        preg_match('/^(\+?88)?01[3-9]\d{8}$/', $login) === 1 => 'mobile',
        default => 'username',
    };

    if ($field === 'mobile') {
        $login = normalize_bd_mobile($login); // -> 8801XXXXXXXXX, store mobiles in this format
    }

    return [$field => $login, 'password' => $this->input('password')];
}
```

Throttle login with `RateLimiter` (5 attempts per minute per login + IP).

### SmsService (transport only)

```php
declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class SmsService
{
    public function send(string $mobile, string $message): bool
    {
        $response = Http::asForm()
            ->timeout(10)
            ->post(config('services.bdbulksms.url'), [
                'to' => $mobile,
                'message' => $message,
                'token' => config('services.bdbulksms.token'),
            ]);

        // Confirm the exact success format in the BdBulkSMS API docs.
        $sent = $response->successful() && str_starts_with($response->body(), 'Ok');

        $sent
            ? Log::info('SMS sent.', ['mobile' => $mobile])
            : Log::error('SMS failed.', ['mobile' => $mobile, 'status' => $response->status()]);

        return $sent;
    }
}
```

- **Never disable SSL verification** (`'verify' => false` is forbidden).
- Config in `config/services.php`: `bdbulksms.url`, `bdbulksms.token`.
- Send SMS through a queued Job so a slow gateway never blocks a request.

### OtpService (rules)

- Generate with `random_int(100000, 999999)`.
- Store **hashed** (`Hash::make`) in Redis cache, key `otp:{mobile}`, **TTL 5 minutes**.
- Send limit: `RateLimiter` keyed by **IP + mobile** (max 5 per hour). Not session-based; clearing cookies must not bypass it.
- Verify limit: max 5 wrong attempts per code, then invalidate the code.
- Delete the code after successful verification.

---

## 6. Logging

Use `Log` with PSR-3 levels and context arrays. Never log passwords, OTPs, tokens, card or wallet numbers, or customer personal data (section 12).

```php
Log::info('Order created.', ['order_id' => $order->id, 'branch_id' => $order->branch_id]);
Log::warning('Stock below threshold.', ['ingredient_id' => $id]);
Log::error('Payment failed.', ['order_id' => $id, 'error' => $e->getMessage()]);
```

Do not log the stack trace in every message; the exception handler already does it. Log with `'exception' => $e` only when needed.

---

## 7. Activity Log & Backup

`LogsActivity` is a **trait**, not an interface:

```php
use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Activitylog\LogOptions;
use Spatie\Activitylog\Traits\LogsActivity;

class User extends Authenticatable
{
    use LogsActivity;

    public function getActivitylogOptions(): LogOptions
    {
        return LogOptions::defaults()
            ->logOnly(['name', 'email', 'mobile', 'username'])
            ->logOnlyDirty();
    }
}
```

- Use `logOnly([...])`, never `logAll()` (it can log password hashes).
- Explicitly log sensitive actions with `activity()->causedBy($user)->performedOn($model)->log('order.voided')`.
- Backup: `backup:run --only-db` daily via scheduler to S3 or a mounted volume, `backup:clean` weekly, and notify on failure.

---

## 8. Localization

- Files: `lang/en/*.php` and `lang/bn/*.php`. Every key must exist in both.
- A middleware sets the locale from user preference → session → default `en`.
- Vue receives translations through Inertia shared props.
- Dates and numbers: format through one helper so Bangla digits can be toggled.

---

## 9. Frontend Rules

- `<script setup>` only. One component = one responsibility.
- Show feedback on every save/delete/error with `toast.success()` / `toast.error()`.
- Handle loading, empty and error states in every list and form.
- Inertia forms use `useForm`. Show server validation errors beside fields.
- Visual and interaction rules are in `design.md`. Dialogs, confirmations, validation messages, dashboards, charts, reports and tables **must** follow sections 12 to 22 there.
- Build the shared UI kit (`resources/js/components/ui-kit/`) first and reuse it. Never use `window.alert()`, `confirm()` or `prompt()`, and never build a one-off dialog, table or chart inside a page.
- Validation lives in Form Requests. Display errors with the `FormField` component, focus the first invalid field, and never clear the form on error.

---

## 10. Testing & Definition of Done

A feature is **done** only when all of these are true:

- [ ] Pest feature tests cover the happy path and main failure cases
- [ ] Authorization tested (allowed role passes, other role gets 403, other branch gets 404, **other tenant gets 404**)
- [ ] No customer personal data in logs, URLs or error trackers (section 12)
- [ ] Reconciliation tests pass: stock balances equal the sum of movements, customer balances equal ledger entries, and collections equal allocations plus advances
- [ ] Where the feature is used in the salesman app: offline and sync tests pass (idempotent, no duplicates, conflicts recorded)
- [ ] `./vendor/bin/pint` is clean and `./vendor/bin/phpstan` passes
- [ ] Both `en` and `bn` translations added
- [ ] Sensitive actions write an activity log entry
- [ ] `NoEmojiTest` passes (no emojis or symbol icons in code, lang files, views or docs)
- [ ] The UI Definition of Done checklist in `design.md` (section 22) passes
- [ ] Migrations run fresh (`migrate:fresh --seed`) without errors
- [ ] The acceptance criteria for the feature in `roadmap.md` are met

---

## 11. Multi-Tenancy (SaaS)

- **Model:** one application and one shared database. Every tenant-owned table has a non-null, indexed `tenant_id`, and unique keys include it (for example `unique(tenant_id, patient_code)`).
- **Tenant resolution:** a `ResolveTenant` middleware reads the host (`{slug}.dtmedix.example` or a custom domain). API requests take the tenant from the authenticated token. Never trust a `tenant_id` sent by the client.
- **Traits:** `BelongsToTenant` adds a global scope and fills `tenant_id` on create. `BelongsToBranch` adds branch scoping for non-owner users. An architecture test fails if a model with a `tenant_id` column lacks the trait.
- **Queues and scheduler:** every queued job, notification and listener carries the tenant ID and restores the tenant context before running. Per-tenant scheduled tasks loop over active tenants.
- **Isolation by prefix:** cache keys, Redis keys and storage paths (`tenants/{id}/...` on a private disk) all include the tenant ID.
- **Platform area (`/platform`)** for Datatech only: create tenants, plans, trial dates, subscription payments, SMS credits, limits, suspend or resume, usage, and support impersonation. Impersonation shows a permanent banner and is written to the audit log.
- **Plan limits** are enforced in one place, a `PlanGate` service (salesmen, warehouses, users, modules). It returns `PLAN_LIMIT_REACHED` or `MODULE_DISABLED`. Never check plan limits in scattered controllers.
- **Tests:** every model and every endpoint has a cross-tenant test (tenant A cannot read or write tenant B, and gets 404).
- **Export and exit:** a command exports all of one tenant's data (CSV or JSON plus files). Tenant deletion is scheduled and only runs after the retention period.

---

## 12. Business, Customer & Field Data Rules

Distributors hold sensitive commercial data (prices, margins, customer lists, outstanding) and track staff in the field. Collect little, expose little, and log access.

- **Customer and supplier data** (names, mobiles, addresses, shop photos, locations) is personal data. Never put it in logs, exception messages, error trackers, URLs or query strings. Use IDs only.
- **Files** (shop photos, cheque images, signatures, PDFs) live on a private disk and are served only through authorized controllers or short-lived signed URLs. Never use public storage.
- **Least access:** salesmen see only their own customers, routes, prices and van stock. Supervisors see their team. Cost prices and margins are hidden from salesmen and delivery staff (`products.view-cost`). Customer lists and price lists exports need a permission and are logged.
- **Location data (salesman monitoring):** record latitude, longitude, accuracy and time only at defined events (day start and end, check-in and check-out, order, collection, new customer). **No continuous background tracking in v1.** The app shows a clear notice of what is collected. Keep raw location data for 12 months by default, never track outside working hours, and show raw data only to permitted roles.
- **Devices:** a salesman token is tied to a registered device (strict mode optional). The app locks with a PIN after inactivity. When a salesman is deactivated or a token is revoked, the app clears its local data the next time it connects.
- **No hard deletes** of transactions or ledgers. Master data (products, customers, suppliers, salesmen) is deactivated, not deleted.
- **Sessions:** idle timeout (default 30 minutes in the admin), strong password policy, optional TOTP for owners.
- **Backups** are encrypted and a restore drill is documented.

---

## 13. Documents, Imports, Messaging & Scheduled Jobs

- **PDF:** use `spatie/laravel-pdf` with the Browsershot (headless Chromium) driver, because simple PDF libraries mis-render Bangla conjunct letters. The Docker image includes Chromium and the fonts (Inter, Noto Sans Bengali). Invoices, challans, money receipts, customer statements and reports are HTML with CSS `@page`. Batches (for example month-end statements for 300 customers) run in a queue and are delivered as one merged PDF or as a zip.
- **Receipt sharing from the phone:** the salesman app shares receipts as a PDF or image through the Web Share API (for example to WhatsApp) and sends an SMS receipt. Direct Bluetooth printing from a browser is unreliable on most low-cost portable printers; treat printing as optional.
- **Import:** `maatwebsite/excel`, run in a queue with chunking, a dry-run validation step, a downloadable row-level error report, and a safe re-run (match on an external key such as SKU or customer code). Imports cover products, units, customers with opening balances, suppliers, opening stock (with batches and expiry) and price lists. Never leave a half-imported dataset without a summary.
- **SMS:** a plain `SmsService` sends through BdBulkSMS in a queued job using per-tenant templates. Bangla SMS is Unicode (70 characters per segment); show the segment count and credit cost in the template editor and before every bulk send.
- **SMS credits:** every tenant has a prepaid credit wallet. Sending reserves and deducts credits in a transaction; insufficient credits fail with a clear message and an owner alert.
- **Scheduled jobs** (overdue reminders, low-stock and expiry alerts, unsettled-cash alerts, daily owner summary, visit plan generation, trial expiry, backups) must be **idempotent** and tenant-aware: running twice never duplicates anything, and one failing tenant never stops the others.

---

## 14. Stock & Inventory Integrity

- **Stock changes only through stock movements.** Never update a quantity column directly. A single `PostStockMovementAction` writes an immutable `stock_movements` row (warehouse, product, batch, signed quantity in base unit, unit cost, reference document, user, time) and updates the `stock_balances` row in the same transaction.
- **Locking:** lock the balance row (`lockForUpdate()`) before checking and changing it. Negative stock is blocked by default and can be allowed per warehouse by setting.
- **Locations:** main warehouses, branch depots and a **van warehouse per salesman** are all warehouses. Issuing stock to a van and returning it are transfers with their own documents.
- **Movement types:** purchase receipt, sales issue, sales return (good or damaged), transfer out and in, van issue and return, adjustment, damage, expiry write-off, stock count difference, opening balance.
- **Costing:** weighted average cost is recalculated on each receipt in the same transaction; each sale line stores the cost at the time of sale. Returns use the original sale cost.
- **Batches and expiry** (per product setting): receipts capture batch and expiry; issues allocate by **first expiry first out**; expired batches cannot be sold; near-expiry alerts use a configurable window (default 90 days). Serial-number tracking is optional per product (later phase).
- **Units:** each product has a base unit and conversions (for example 1 carton = 12 dozen = 144 pieces). Documents show the unit chosen by the user and store the quantity in the base unit.
- **Reconciliation test:** for every product and warehouse, `balance = sum of movements`. Run it in CI and as a nightly check that alerts the platform team.
- Adjustments, write-offs and stock count differences need a reason, and approval above a configured value.

---

## 15. Sales, Credit & Collection Integrity

- **Documents:** order, delivery or challan, invoice, credit note (return), money receipt, settlement, cheque. Posted invoices and receipts are **immutable**; mistakes are corrected by a credit note or a receipt reversal with a reason and approval.
- **Customer ledger:** every invoice (debit), receipt (credit), credit note (credit) and adjustment writes an immutable `customer_ledger_entries` row. The balance is the sum of entries and is never typed. Reconcile in tests.
- **Allocation:** a receipt is allocated to invoices (oldest first by default, or the invoices the salesman selected). Allocations are stored rows. An unallocated amount becomes an explicit **advance** on the customer's account.
- **Credit control** runs in one place (`CheckCreditAction`) for orders and invoices: limit amount, credit days and overdue rules, with the tenant's mode (`block`, `warn` or `approval`). Overrides need a permission, a reason and an audit entry.
- **Pricing:** price comes from the customer's price list at the time of the order and is stored as a snapshot on the line. Schemes are applied by a single `ApplySchemesAction`. Discounts and price overrides are limited by role.
- **Collections stay "with the salesman"** (cash in hand) until a cashier **settles** them. Settlement compares expected and counted cash; a shortage becomes a recoverable record with approval, never silently ignored. Cheques follow their own status flow (received, deposited, cleared, bounced); a bounced cheque reverses the receipt.
- **Document numbers** are generated under a lock per series (branch, salesman or device). Salesman devices have their own series so offline documents never collide. Gaps are not allowed; voided numbers stay visible as voided.
- **Day close:** a salesman's day is closed only after stock return and cash settlement are reconciled or flagged.
- Totals on dashboards must equal report totals; add reconciliation tests for sales, collections and outstanding.

---

## 16. Offline-First Field App Rules

- The salesman app (`/app`) is an installable PWA for Android Chrome. The service worker caches the app shell with versioned caches; on a new deploy show "Update available" and never reload in the middle of an order or collection.
- **Local data:** after login the app downloads only what that salesman needs: own customers and routes, price lists, products and schemes, van stock, open invoices of own customers, and settings. It is stored in IndexedDB (Dexie) and updated through `/api/v1/sync/changes`.
- **Outbox:** every write made in the app (visit, order, collection, new customer, return request, van sale, day start and end) is stored locally first with a `client_id` (ULID), type, payload and device timestamp, then pushed to `/api/v1/sync/push`. Retries must be safe because the server is idempotent by `client_id`.
- **The UI writes to the local database first** and shows the result immediately; sync is a background concern with a visible status pill and pending counter.
- **Documents created offline** (receipts, van invoices) use the salesman's own number series and show as "pending sync" until accepted.
- **Conflicts** (van stock would go negative, credit limit exceeded since the last sync, price changed, customer deactivated) are recorded and surfaced to the supervisor in a Sync Conflicts screen. They are never silently dropped or overwritten; money collected is always accepted and recorded.
- Never queue anything that needs the server to decide in real time except where the rules above allow (SMS sending, approvals that need a supervisor).
- Do not store passwords or secrets on the device; lock with a PIN; clear local data on logout, revoke or deactivation.
- Test offline behavior with a Playwright test that goes offline, takes orders and collections, reconnects, and asserts no duplicates and no losses.

---

## 17. Working Method (for the AI)

1. Read `docs/prd.md`, `docs/api.md`, `docs/design.md`, `docs/roadmap.md`.
2. Build **one phase at a time** from `roadmap.md`. Do not start the next phase.
3. For each feature: migration → model → Form Request → Action → controller → Vue page → tests.
4. If something in the docs is unclear or contradictory, **ask before assuming**.
5. Run tests and fix failures before reporting a feature as finished.
