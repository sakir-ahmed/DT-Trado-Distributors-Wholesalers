# API SPECIFICATION: DT Trado

> Sections 1 and 2 are shared conventions. Sections 3 onward are specific to DT Trado.

---

## 1. Purpose & Conventions

**Who uses the API:** the salesman PWA (`/app`) for sync and field operations, future native apps, and integrations. **The admin web app uses Inertia controllers (session auth)** that call the _same Actions_ as the API. Do not duplicate business logic.

| Topic         | Rule                                                                                                                                                                                           |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Base path     | `/api/v1/...` in `routes/api.php`                                                                                                                                                              |
| Tenant        | Taken from the authenticated token. **Never accepted from the client body.**                                                                                                                   |
| Auth          | Laravel Sanctum bearer tokens. Salesman and delivery tokens are bound to a registered device (`X-Device-Id`). Token abilities: `staff`, `salesman`, `delivery`.                                |
| Docs          | OpenAPI generated automatically (Scramble) at `/api-docs`; every endpoint has request and response examples.                                                                                   |
| Versioning    | No breaking changes inside `v1`: add fields freely, never rename or remove. Clients send `X-App-Version`, `X-Platform` and `X-Device-Id`.                                                      |
| Format        | JSON. Success: `{"success": true, "data": {...}, "message": "Localized message"}`. Error: `{"success": false, "error": {"code": "CREDIT_LIMIT_EXCEEDED", "message": "...", "details": {...}}}` |
| Language      | `Accept-Language: en` or `bn`                                                                                                                                                                  |
| Money         | Decimal **string** in BDT, e.g. `"15250.00"`                                                                                                                                                   |
| Quantity      | Decimal **string** in the product's base unit (3 decimals), with an optional `unit` and `unit_quantity` for display                                                                            |
| Dates         | ISO 8601 with timezone (`Asia/Dhaka`)                                                                                                                                                          |
| Pagination    | `?page=1&per_page=25`; response includes `meta` (`current_page`, `last_page`, `total`)                                                                                                         |
| Filtering     | `?filter[status]=open&filter[route]=12&sort=-created_at`                                                                                                                                       |
| Idempotency   | `POST` of orders, invoices, collections, settlements, returns, stock documents, imports and bulk SMS requires an `Idempotency-Key` header. Sync items use `client_id` (section 5).             |
| Rate limits   | Login and OTP: strict. Others: 120 per minute per token.                                                                                                                                       |
| Scope         | Salesmen see only their own customers, routes and van stock. Supervisors see their team. Every branch or warehouse route checks access, else **404**.                                          |
| Personal data | No customer data in URLs or query strings (use IDs).                                                                                                                                           |

### Error codes

| HTTP | Code                                      | When                                                                                     |
| ---- | ----------------------------------------- | ---------------------------------------------------------------------------------------- |
| 401  | `UNAUTHENTICATED`                         | Missing or invalid token                                                                 |
| 401  | `DEVICE_NOT_REGISTERED` / `TOKEN_REVOKED` | Device unknown or access revoked (the app clears local data)                             |
| 403  | `FORBIDDEN`                               | Role lacks permission                                                                    |
| 404  | `NOT_FOUND`                               | Missing, or belongs to another scope, user or tenant                                     |
| 422  | `VALIDATION_FAILED`                       | Invalid input, `details` has field errors                                                |
| 409  | `DUPLICATE_CUSTOMER_SUSPECTED`            | Name and mobile match an existing customer                                               |
| 409  | `INSUFFICIENT_STOCK`                      | Not enough stock in the warehouse or batch (`details` has available)                     |
| 409  | `BATCH_EXPIRED`                           | Selling an expired batch                                                                 |
| 409  | `CREDIT_LIMIT_EXCEEDED`                   | Over limit; `details` has mode (`block`, `warn`, `approval`), limit, balance and overdue |
| 409  | `CUSTOMER_ON_HOLD`                        | Customer blocked for credit reasons                                                      |
| 202  | `APPROVAL_REQUIRED`                       | Request created and waiting for a supervisor (`details` has approval id)                 |
| 422  | `PRICE_OVERRIDE_NOT_ALLOWED`              | Price or discount outside the role's limit                                               |
| 409  | `DOCUMENT_ALREADY_POSTED`                 | Editing a posted invoice or receipt (use a credit note or reversal)                      |
| 409  | `RECEIPT_ALREADY_REVERSED`                | Reversing twice                                                                          |
| 422  | `ALLOCATION_EXCEEDS_DUE`                  | Allocation larger than the invoice due                                                   |
| 409  | `DAY_NOT_CLOSEABLE`                       | Unreconciled van stock or cash                                                           |
| 409  | `SETTLEMENT_DIFFERENCE_UNAPPROVED`        | Shortage or excess needs approval                                                        |
| 409  | `UNIT_CONVERSION_LOCKED`                  | Changing conversions after movements exist                                               |
| 403  | `COST_NOT_ALLOWED`                        | Reading cost data without `products.view-cost`                                           |
| 403  | `PLAN_LIMIT_REACHED` / `MODULE_DISABLED`  | Salesman, warehouse or module limit                                                      |
| 403  | `TENANT_SUSPENDED` / `TENANT_READ_ONLY`   | Trial ended or account suspended                                                         |
| 402  | `SMS_CREDIT_INSUFFICIENT`                 | Not enough SMS credits                                                                   |
| 429  | `TOO_MANY_REQUESTS`                       | Rate limit hit                                                                           |
| 500  | `SERVER_ERROR`                            | Unexpected (details never exposed)                                                       |

---

## 2. Authentication

| Method | Path                                      | Notes                                                                                                                            |
| ------ | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| POST   | `/api/v1/auth/login`                      | `login` (username, email or mobile) + `password` + device info; returns token. New devices need approval when strict mode is on. |
| POST   | `/api/v1/auth/logout`                     | Revokes the current token                                                                                                        |
| GET    | `/api/v1/auth/me`                         | User, role, permissions, branches, plan limits, trial status, feature flags                                                      |
| POST   | `/api/v1/auth/otp/send` and `/otp/verify` | Rate limited by IP and mobile; hashed code, 5-minute expiry, 5 attempts                                                          |
| POST   | `/api/v1/devices`                         | Register or update device and push token                                                                                         |
| POST   | `/api/v1/devices/{device}/revoke`         | Supervisor or owner revokes; the app wipes local data on the next connection                                                     |

---

## 3. Back-Office Endpoints

Branch routes are prefixed `/api/v1/branches/{branch}` where the data belongs to a branch.

### Catalog

| Method       | Path                                        | Notes                                                                                            |
| ------------ | ------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| GET/POST/PUT | `/products`, `/categories`, `/brands`       | Barcodes, units, conversions, batch and expiry flags; cost fields only with `products.view-cost` |
| GET/POST/PUT | `/price-lists`, `/price-lists/{list}/items` | Effective dates, customer groups                                                                 |
| GET/POST/PUT | `/schemes`                                  | Quantity, value, product or category based; validity; audience                                   |
| POST         | `/imports/{type}`                           | `products`, `customers`, `suppliers`, `opening-stock`, `price-lists`; `dry_run` then commit      |

### Suppliers & Purchase

| Method       | Path                 | Notes                                                                                          |
| ------------ | -------------------- | ---------------------------------------------------------------------------------------------- |
| GET/POST/PUT | `/suppliers`         | Opening balance                                                                                |
| GET/POST     | `/purchase-orders`   | Statuses: draft, approved, partly received, closed                                             |
| POST         | `/goods-receipts`    | Batch, expiry, cost, differences; posts stock movements and payables. Needs `Idempotency-Key`. |
| POST         | `/purchase-returns`  | Posts stock and supplier ledger                                                                |
| GET/POST     | `/supplier-payments` | Allocation to purchase invoices                                                                |

### Warehouses & Stock

| Method       | Path                                         | Notes                                               |
| ------------ | -------------------------------------------- | --------------------------------------------------- |
| GET/POST/PUT | `/warehouses`                                | Including van warehouses                            |
| GET          | `/stock?filter[warehouse]=&filter[product]=` | Balances (on hand, reserved, available), batch view |
| GET          | `/stock/ledger?product=&warehouse=`          | Movements                                           |
| POST         | `/transfers` / `/transfers/{id}/receive`     | Between warehouses                                  |
| POST         | `/van-issues` / `/van-returns`               | Scanned lines, differences, reconciliation          |
| POST         | `/stock-adjustments`                         | Reason required; approval above the limit           |
| GET/POST     | `/stock-counts`, `/stock-counts/{id}/post`   | Variance review then post                           |
| POST         | `/expiry-writeoffs`                          | Reason and approval                                 |

### Customers, Routes & Salesmen

| Method       | Path                                                     | Notes                                                                                            |
| ------------ | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| GET          | `/customers?filter[q]=`                                  | Search by name, code or mobile (scoped for salesmen)                                             |
| POST         | `/customers`                                             | May return `DUPLICATE_CUSTOMER_SUSPECTED` unless `confirm_new=true`; can enter an approval state |
| GET/PATCH    | `/customers/{customer}`                                  | Credit limit changes need permission                                                             |
| GET          | `/customers/{customer}/ledger` / `/statement?format=pdf` | Ledger and statement                                                                             |
| GET          | `/customers/{customer}/history`                          | Purchase trend and last orders                                                                   |
| GET/POST/PUT | `/routes`, `/visit-plans`, `/salesmen`, `/targets`       | Setup                                                                                            |

### Sales

| Method | Path                                                      | Notes                                                                                                                                                                         |
| ------ | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ----------------------------------- |
| POST   | `/orders`                                                 | Source `app`, `office` or `phone`. Applies price list and schemes, runs the credit check. May return `CREDIT_LIMIT_EXCEEDED` or `APPROVAL_REQUIRED`. Needs `Idempotency-Key`. |
| POST   | `/orders/{order}/approve` / `/reject`                     | Supervisor                                                                                                                                                                    |
| POST   | `/orders/{order}/invoice`                                 | Creates the invoice and issues stock (FEFO); returns `INSUFFICIENT_STOCK` when needed                                                                                         |
| POST   | `/invoices`                                               | Direct invoice (office or van sale)                                                                                                                                           |
| GET    | `/invoices/{invoice}/pdf?format=a4                        | a5                                                                                                                                                                            | thermal` | PDF; `/challan` for a delivery note |
| POST   | `/sales-returns`                                          | Good or damaged, reason, photo, original invoice; approval above the limit                                                                                                    |
| POST   | `/credit-notes`                                           | Posted documents are corrected only by credit notes                                                                                                                           |
| POST   | `/credit-checks`                                          | Dry run: given customer and amount, returns the credit decision                                                                                                               |
| POST   | `/pricing/quote`                                          | Given customer and lines, returns prices, schemes and totals (used by the app)                                                                                                |
| GET    | `/approvals` / `POST /approvals/{id}/approve` / `/reject` | Approvals inbox (credit, price, new customer, return, adjustment, shortage)                                                                                                   |

### Collections & Settlement

| Method | Path                                          | Notes                                                                                                                                                                                             |
| ------ | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- | ---------------------- |
| POST   | `/collections`                                | Customer, amount, method (`cash`, `bkash`, `nagad`, `bank`, `cheque`), reference, optional invoice list. Allocates oldest first by default; unallocated becomes advance. Needs `Idempotency-Key`. |
| GET    | `/collections/{collection}/receipt?format=pdf | image`                                                                                                                                                                                            | Money receipt |
| POST   | `/collections/{collection}/reverse`           | Reason and approval; may return `RECEIPT_ALREADY_REVERSED`                                                                                                                                        |
| GET    | `/cheques` / `POST /cheques/{cheque}/status`  | `deposited`, `cleared`, `bounced`                                                                                                                                                                 |
| GET    | `/salesmen/{salesman}/cash-position`          | Collected but not settled                                                                                                                                                                         |
| POST   | `/settlements`                                | Expected versus counted by method; shortage needs approval (`SETTLEMENT_DIFFERENCE_UNAPPROVED`)                                                                                                   |
| POST   | `/day-close`                                  | May return `DAY_NOT_CLOSEABLE`                                                                                                                                                                    |
| GET    | `/outstanding?group_by=customer               | salesman                                                                                                                                                                                          | route&aging=` | Aging buckets, overdue |
| POST   | `/outstanding/reminders`                      | Bulk SMS with an estimate step                                                                                                                                                                    |

### Delivery (Pro)

| Method | Path                              | Notes                                                    |
| ------ | --------------------------------- | -------------------------------------------------------- |
| POST   | `/delivery-trips`                 | Assign invoices to a delivery man                        |
| GET    | `/delivery/my-trips`              | Delivery man's list                                      |
| POST   | `/deliveries/{delivery}/complete` | Proof (signature or OTP), partial quantities, collection |
| POST   | `/deliveries/{delivery}/fail`     | Reason                                                   |

### Targets, Dashboard, Reports & Messaging

| Method   | Path                                      | Notes                                            |
| -------- | ----------------------------------------- | ------------------------------------------------ | ----- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| GET      | `/targets/progress?salesman=&month=`      | Target versus achieved                           |
| GET/POST | `/incentive-statements` / `/approve`      | Computed from posted data; locked after approval |
| GET      | `/salesmen/{salesman}/day-timeline?date=` | Check-ins, visits, orders, collections           |
| GET      | `/dashboard?branch=all&period=today`      | KPIs and alerts                                  |
| GET      | `/reports/{type}?format=json              | pdf                                              | xlsx` | `sales`, `purchase`, `stock`, `expiry`, `outstanding`, `collections`, `salesman-performance`, `customer-trend`, `returns`, `payables`, `profit` |
| GET/PUT  | `/sms-templates`                          | Bilingual with preview and segment count         |
| POST     | `/sms/estimate` / `/sms/bulk`             | Credit estimate then send (`Idempotency-Key`)    |
| GET      | `/sms/credits` / `/sms/log`               | Balance and log                                  |
| GET/POST | `/expenses`                               | Category and attachment                          |

---

## 4. Salesman App Endpoints (`/api/v1/field`)

All routes are scoped to the authenticated salesman and support offline use via section 5.

| Method | Path                                                     | Notes                                                                                 |
| ------ | -------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| GET    | `/today`                                                 | Route, planned visits, targets, van stock summary                                     |
| POST   | `/day/start` / `/day/end`                                | Location and time; end may return `DAY_NOT_CLOSEABLE`                                 |
| POST   | `/visits/check-in` / `/visits/{visit}/check-out`         | Location, accuracy, outcome (`ordered`, `collected`, `no_order`, `closed`) and reason |
| GET    | `/customers/{customer}/card`                             | Banner data, open invoices, last orders, top products                                 |
| POST   | `/customers`                                             | New customer with location and photo                                                  |
| GET    | `/products?updated_since=`                               | Products, prices and schemes for the salesman                                         |
| POST   | `/orders`, `/collections`, `/returns`, `/expense-claims` | Same rules as the back-office endpoints, using the device's number series             |
| GET    | `/van-stock`                                             | Current van balances                                                                  |
| POST   | `/photos`                                                | Upload shop, return or cheque photos (private disk)                                   |

---

## 5. Offline Sync (`/api/v1/sync`)

| Method | Path                                 | Notes                                                                                                                                                                                                                                              |
| ------ | ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET    | `/sync/bootstrap`                    | Full data set for this user (customers, routes, products, prices, schemes, van stock, open invoices, settings) plus a `data_version`                                                                                                               |
| GET    | `/sync/changes?since={data_version}` | Delta updates, including deactivations and deletions                                                                                                                                                                                               |
| POST   | `/sync/push`                         | Body: an array of items. Each item has `client_id` (ULID), `type` (`visit`, `order`, `collection`, `return`, `customer`, `van_sale`, `day_start`, `day_end`, `expense_claim`), `payload`, `created_at_device` and `device_location` (if captured). |

**Rules**

- Every item is processed **idempotently** by `client_id`; resending returns the earlier result.
- The response gives a per-item result: `accepted`, `duplicate` or `conflict` with a code (`INSUFFICIENT_STOCK`, `CREDIT_LIMIT_EXCEEDED`, `PRICE_CHANGED`, `CUSTOMER_INACTIVE`, `BATCH_EXPIRED`).
- Items are applied in order of `created_at_device`. One failing item never blocks the others.
- **Collections are always accepted** and recorded; a doubt is flagged for review, not rejected.
- Offline orders use the price snapshot taken on the device; the server records the `data_version`. A price difference is recorded as a conflict for the supervisor, not silently changed.
- Document numbers created offline come from the device's own series (`field_series`), so they never collide.
- Conflicts are stored and appear in the Sync Conflicts screen.
- A revoked token returns `TOKEN_REVOKED`; the app wipes local data after the final sync attempt is refused.

---

## 6. Platform API (`/api/v1/platform`, super-admin only)

| Method       | Path                                      | Notes                                       |
| ------------ | ----------------------------------------- | ------------------------------------------- |
| GET/POST/PUT | `/tenants`                                | Create and manage distributors              |
| PUT          | `/tenants/{tenant}/plan`                  | Plan, limits, module toggles, trial dates   |
| POST         | `/tenants/{tenant}/suspend` / `/resume`   | Suspension                                  |
| GET/POST     | `/tenants/{tenant}/subscription-payments` | Subscription ledger                         |
| POST         | `/tenants/{tenant}/sms-credits`           | Add credits (adjustments need a reason)     |
| POST         | `/tenants/{tenant}/impersonate`           | Short-lived, audited, with a visible banner |
| POST         | `/tenants/{tenant}/export`                | Full data export job                        |
| POST         | `/tenants/{tenant}/imports`               | Import monitor: upload, `dry_run`, commit   |
| GET          | `/tenants/{tenant}/usage`                 | Salesmen, warehouses, documents, SMS        |

---

## 7. SMS Provider & Mobile Money

- `SmsService` calls BdBulkSMS from a queued job with retries. Credits are reserved and deducted in the same transaction that writes the message log; a failed send refunds the reservation. Message logs store the template, status and cost, not extra personal data. Sender ID registration depends on the provider and local rules; confirm with the provider.
- **bKash and Nagad collections** are recorded by the salesman with the TrxID and amount, flagged `unverified`, and verified by the accountant against the merchant statement. Direct gateway integration is out of scope for v1.
- Webhooks: none are required in v1. Any future webhook is a public web route, signature-verified, idempotent and processed in a queue.

---

## 8. Dashboards and Real-Time

There is no real-time server in v1. Dashboards refresh on load and every few minutes; approvals reach supervisors by in-app inbox, optional SMS, and later push notifications.
