# DESIGN SYSTEM: DT Trado

> Shared Datatech design system, adapted for distributors and wholesalers. Sections 6, 7 and 9 to 11 are specific to DT Trado; the rest is generic. Salesmen work on phones in shops with weak internet, so the field app is mobile-first and offline-first. Screens per role are listed in `prd.md`.

---

## 1. Principles

- **Fast, exact and forgiving.** Back-office staff enter many documents a day and salesmen have two minutes per shop. Make the common action (add order lines, record a collection, receive stock) take seconds, and make risky actions (credit override, price override, reverse, write-off) deliberate. Minimize clicks, scrolling and whitespace.
- **Instant feedback.** Every action shows a Sonner toast (success, error, or warning).
- **Light and dark mode** are both first-class. Never ship a screen that was only checked in one.
- **Two density modes:**
  - **Compact** (back-office tables, forms, reports): control sizes, spacing and radii from **section 24**, table rules from **section 25**.
  - **Touch** (the salesman app, delivery and warehouse screens on phones or tablets, anything used on a touchscreen): minimum **44px** tap targets, `text-base` or larger, generous spacing between buttons to avoid mis-taps.
- **Always show which customer and how much they owe.** Every screen that touches a customer shows a persistent customer banner (name, code, route, outstanding, credit limit and available credit, overdue amount) so nobody sells or collects blind.
- **Mobile first and offline first for salesmen.** Design the field app at 360px width first, one-handed, readable in sunlight, with a visible sync status and no feature that silently fails without internet.
- **Never rely on color alone.** Pair profit/loss/status colors with an icon or text label (arrow-up / arrow-down / check / warning icons, see section 23).

---

## 2. Design Tokens (Tailwind CSS v4)

Define tokens once in `resources/css/app.css`. Components use token classes (`bg-primary`, `text-success`), never raw palette classes like `bg-slate-900`. Dark mode swaps the variables.

```css
@import "tailwindcss";

@custom-variant dark (&:where(.dark, .dark *));

@theme {
  --font-sans:
    "Inter", "Noto Sans Bengali", ui-sans-serif, system-ui, sans-serif;

  --color-background: oklch(0.985 0.002 247);
  --color-surface: oklch(1 0 0);
  --color-foreground: oklch(0.21 0.034 265);
  --color-muted: oklch(0.55 0.02 255);
  --color-border: oklch(0.92 0.005 255);

  --color-primary: oklch(0.21 0.034 265); /* slate-900 */
  --color-primary-foreground: oklch(0.98 0.003 248);
  --color-accent: oklch(0.546 0.245 262.881); /* blue-600 */
  --color-accent-foreground: oklch(0.97 0.014 254.604);
  --color-warning: oklch(0.769 0.188 70); /* amber-500 */
  --color-success: oklch(0.596 0.145 163); /* emerald-600 */
  --color-danger: oklch(0.586 0.253 17); /* rose-600 */
}

.dark {
  --color-background: oklch(0.129 0.042 265); /* slate-950 */
  --color-surface: oklch(0.208 0.042 265); /* slate-900 */
  --color-foreground: oklch(0.984 0.003 248);
  --color-muted: oklch(0.704 0.04 256);
  --color-border: oklch(0.279 0.041 260);
  --color-primary: oklch(0.984 0.003 248);
  --color-primary-foreground: oklch(0.21 0.034 265);
}
```

**Meaning of colors**

- Primary (slate): navigation, main buttons, headers.
- Accent (blue): active tab, selected items, links, highlights. Trustworthy and businesslike.
- Warning (amber): due soon, low stock, near expiry, pending approval, caution badges.
- Success (emerald): collected, paid, delivered, in stock.
- Danger (rose): overdue, over credit limit, out of stock, expired, bounced cheque, reversal, delete.

Adjust token values to taste, but keep the names stable.

---

## 3. Typography

- **Fonts:** `Inter` for Latin, **`Noto Sans Bengali`** for Bangla (Inter has no Bangla glyphs). Self-host fonts; don't depend on a CDN.
- Bangla text needs slightly more line height: use `leading-relaxed` when the locale is `bn`.

| Use                     | Classes                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| Dashboard metric value  | `text-2xl font-bold tracking-tight`                                         |
| Page title              | `text-lg font-semibold`                                                     |
| Section / card header   | `text-sm font-semibold`                                                     |
| Table / body / controls | `text-[13px] leading-[18px]`                                                |
| Badge / label           | `text-xs font-medium uppercase tracking-wider` (skip `uppercase` in Bangla) |

- Currency: `৳` prefix, thousands grouping, 2 decimals in documents, 0 decimals allowed on dashboard cards. Digits (English or Bangla) follow the user's language setting through one shared formatter.

---

## 4. Components

Use shadcn-vue primitives. Do not invent custom versions of components that already exist.

- **Data table:** sticky header, compact rows, inline search, filter dropdowns (branch, date range, status), column sorting, pagination, and an empty state with a clear message.
- **Slide-over:** right side, 30–40% of screen width, for create/edit forms and detail views, so the user keeps their place. Use a centered modal only for short confirmations.
- **Dashboard card:** label, big value, small trend (arrow-up / arrow-down icon with %), optional sparkline.
- **Status badges:** one shared component mapping each Enum value to color + label + icon.
- **Confirmations:** destructive actions (void, delete, adjust stock) require a confirm dialog and, where the PRD says so, a reason field.
- **Forms:** labels above inputs, errors below fields in `text-danger`, primary action bottom-right, `Enter` submits. Sizes follow section 24.

---

## 5. Required States

Every list, form and page must handle: **loading** (skeletons, not spinners), **empty** (message + action button), **error** (message + retry), and **no permission** (clear 403 page).

---

## 6. Sales Desk & Collection Desk (back office, compact, keyboard-first)

Back-office staff create orders and invoices for phone and walk-in customers, and record collections.

- **Invoice entry layout:** top block: customer search with the customer banner, salesman, warehouse, price list, date, payment terms. Middle: a line grid (product search by name, code or barcode; unit; quantity; price; discount; scheme; line total). Right or bottom: totals (subtotal, discounts, scheme benefits, VAT if enabled, total, previous outstanding, new outstanding), and **Save**, **Save and print**, **Save and new**.
- **Line grid behavior:** `Enter` moves right, then down to a new line. Typing a product shows stock available in the chosen warehouse (and by batch with expiry when enabled). Quantity above available stock shows an inline warning; quantity changes recalculate schemes instantly. Prices come from the customer's price list; a price override is allowed only with permission and shows an icon.
- **Credit check:** if the invoice would exceed the credit limit or the customer has overdue invoices, a banner explains it and shows the action required (block, warning, or ask a manager for approval).
- **Collection desk:** search a customer, see open invoices (date, invoice no., amount, paid, due, days overdue) with checkboxes, enter the amount and method (cash, bKash, Nagad, bank, cheque with number and date), and see how it will be allocated (oldest first by default) and any advance created. **Save and print** produces the money receipt.
- **Shortcuts:** `/` customer or product search, `F2` new customer, `F4` focus payment, `F9` save and print, `Esc` close dialog.

---

## 7. Printed Documents (invoice, challan, receipt, statement)

- **Tax or sales invoice** (A4 or A5): seller name, logo, address, BIN or VAT number if applicable, customer name, address and code, invoice number and date, salesman, items (code, name, unit, quantity, rate, discount, amount) following section 25 alignment, subtotal, discounts, VAT, total, in-words amount, previous balance, net outstanding, payment terms and due date, signature blocks, and a verification QR code. Confirm the legal invoice format (including Mushak requirements) with the client's accountant before launch.
- **Delivery challan:** same layout without prices (option to show prices), with received-by signature and mobile.
- **Money receipt** (A5 or 80mm): receipt number, date, customer, amount in figures and words, method, reference (TrxID or cheque details), invoices settled, remaining balance, received by.
- **Customer statement:** period, opening balance, a ledger table (date, document, debit, credit, running balance), closing balance, and an aging summary.
- Documents can be printed, downloaded as PDF, shared from the phone as image or PDF, and sent by SMS link. Bilingual when the tenant chooses. Bangla text must render correctly in PDFs; test early.

---

## 8. Accessibility & Quality

- Contrast ratio at least 4.5:1 for text in both themes.
- Visible focus rings; everything usable by keyboard.
- Icon-only buttons have an `aria-label` / tooltip.
- No horizontal page scroll; wide tables scroll inside their own container.

---

## 9. Salesman App (mobile-first, offline-first, touch mode)

- **Home:** greeting, today's route and visit progress (done, remaining, ordered, collected), the **target progress** bar, a status pill (Online, Offline, Syncing with pending count), and big buttons: Start day, Visits, New order, Collect, Customers, Van stock. A salesman sees only their own data.
- **Start and end day:** start day records the time and location, shows the day plan and **van stock** received; end day shows sales, collections, stock left, and cash to settle, and requires a reconciliation step before it can be marked complete.
- **Visit list and map:** today's customers in route order with distance, outstanding and last order; tap to open the customer card. **Check in** captures location and time (with an accuracy warning and an out-of-range flag set by the tenant); **Check out** requires an outcome (ordered, collected, no order with reason, shop closed).
- **Customer card:** the customer banner (outstanding, overdue, available credit), last orders, top products, open invoices, notes, and actions: New order, Collect payment, Return, Call, Navigate. A **new customer** form is short (name, shop name, mobile, address, location pin, photo, type) and sends the customer for approval if the tenant requires it.
- **Order entry (the key screen):** product search by name, code or **barcode scan**, product cards or a compact list with price, stock, and scheme tags; quantity stepper with the unit selector (carton, dozen, piece) and a numeric keypad; running total in a sticky bottom bar; **credit and scheme feedback** appears inline before submit ("Over credit limit by 5,000. Ask the supervisor"). Target: a 10-line order in under 2 minutes. A draft is saved automatically.
- **Collection:** shows open invoices, amount field, method, reference or cheque details, allocation preview, and **Save and share receipt** (PDF or image through the share sheet, plus SMS). Collections are never blocked while offline.
- **Van sale** (for tenants that sell from the van): the order screen uses van stock and creates the invoice immediately; **Returns** record good or damaged items with a reason and photo.
- **Everything saves locally first**, shows a clock icon while pending, a check when synced, and a warning icon when the server needs attention.
- **Performance and comfort:** large touch targets, high contrast for sunlight, no heavy images, works on mid-range Android phones, battery friendly (no continuous tracking).

---

## 10. Owner, Supervisor & Warehouse Screens

- **Owner dashboard:** see section 15. Mobile layout shows the five numbers that matter: sales today, collected today, total outstanding, overdue, and stock value, with a salesman leaderboard below.
- **Outstanding screen:** the most important screen for the owner. A table of customers with outstanding, overdue, oldest due days, credit limit, last payment and salesman, filterable by route, salesman, aging bucket and customer group; an **aging bar chart** (current, 1-30, 31-60, 61-90, 90+ days); row action opens the customer ledger; bulk reminder SMS shows segments and credits first.
- **Salesman performance:** target versus achieved, orders, visits, productive visit rate, collection, new customers, a day timeline of check-ins on a map, and **cash with salesman not yet settled**.
- **Stock screens:** a stock position table by warehouse (on hand, reserved, available, value), batch and expiry view with near-expiry highlighting, and a **fast and slow moving** analysis. Stock count: a mobile-friendly counting list with barcode scan and a variance review before posting.
- **Warehouse:** a pick list per order group, loading list per van, van issue and return screens with scan, goods receipt against a purchase order with batch and expiry entry, and a discrepancy dialog when quantities differ.
- **Cash settlement:** a salesman's day collections by method, expected versus counted cash, shortage handling with approval, and cheque list.

---

## 11. Supervisor Approvals & Sync Conflicts

- **Approvals inbox:** credit overrides, price or discount overrides, new customers, returns, stock adjustments and shortages in one list with the requester, amount and reason; **Approve** and **Reject** (reason required) are one tap on mobile.
- **Sync Conflicts screen:** items the server could not accept in full (for example van stock would go negative, price changed, customer deactivated) with a clear explanation and a resolve action. Money collected is always accepted and flagged for review rather than rejected.
- **Notifications:** approval requests and day-end reminders appear in the bell and as optional push or SMS to supervisors.

---

## 12. Dialogs, Modals & Confirmations

**Never use `window.alert()`, `window.confirm()` or `window.prompt()`.** All dialogs use the shared components below (shadcn-vue `Dialog` / `AlertDialog` / `Sheet`).

### Which container to use

| Situation                                          | Use                                                              |
| -------------------------------------------------- | ---------------------------------------------------------------- |
| Quick yes/no, destructive or irreversible action   | **Confirm dialog** (small, centered)                             |
| Create / edit a record, detail view, filters       | **Slide-over** (right, 30–40% width)                             |
| Short form with 1–3 fields (reason, PIN, quantity) | **Small dialog**                                                 |
| Phone / small tablet                               | Dialogs become **bottom sheets**, slide-overs become full screen |
| Hint or small extra info                           | **Popover / tooltip**, never a dialog                            |

Never open a dialog from inside a dialog (maximum one level; a confirm on top of a slide-over is allowed).

### Confirm dialog anatomy

- Centered, `max-w-md`, rounded, soft shadow, dimmed blurred backdrop, 150ms fade + scale-in.
- A **colored icon circle** at top (danger = rose with warning/trash icon, warning = amber, info = slate, success = emerald).
- **Title** = the question ("Reverse receipt MR-0412?"). **Message** = the consequence in plain words ("The collection of 15,000.00 will be reversed and the invoices become due again. The salesman is notified.").
- Show the key facts of the thing being changed (invoice number, amount, item name) so users don't confirm the wrong row.
- **Buttons:** right-aligned, Cancel on the left, action on the right. The action label is a **verb + object** ("Reverse receipt", "Delete route"), never "OK" or "Yes". Danger actions use the danger color.
- Keyboard focus starts on **Cancel** for destructive dialogs, on the action for harmless ones. `Esc` cancels. `Enter` triggers the focused button. Focus is trapped; background scroll is locked.
- While the request runs, the action button shows a spinner and is disabled; the dialog cannot be dismissed. On success it closes and a toast confirms. On failure the dialog stays open with the error shown inside it.

### Confirmation levels

| Level      | When                                                                                                    | Pattern                                                                                  |
| ---------- | ------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| 1 Simple   | Reversible, low risk (archive, cancel draft)                                                            | Confirm dialog                                                                           |
| 2 Reason   | Needs an audit trail (credit override, price override, receipt reversal, stock adjustment or write-off) | Dialog with a required reason (dropdown of reason codes plus optional note)              |
| 3 Approval | Needs a higher role (void, large discount, refund)                                                      | Dialog asking for the **manager's PIN or login**; shows who approved in the activity log |
| 4 Typed    | Irreversible and high impact (deactivate warehouse, delete tenant data, close an accounting period)     | User must type the exact name or the word `CONFIRM` before the button enables            |

### Other dialog types

- **Unsaved changes guard:** leaving a form with edits shows "Discard changes?" with _Keep editing_ / _Discard_.
- **Success dialog:** only for major completed flows (payment completed: big check animation, change due, Print receipt / New order buttons). Everything else uses a toast.
- **Session timeout:** warning 60 seconds before logout with a "Stay signed in" button. The salesman app and shared desk screens use a PIN lock instead of logout.
- **Blocking error dialog:** only when the user cannot continue (permission lost, system error). Show a short message, a reference ID, and a Retry or Go back button.

### Implementation rule

Provide a promise-based composable so any page can ask for confirmation in one line:

```ts
const confirm = useConfirm();

const result = await confirm({
  variant: "danger", // danger | warning | info
  level: "reason", // simple | reason | approval | typed
  title: t("orders.void.title", { invoice }),
  message: t("orders.void.message"),
  confirmText: t("orders.void.confirm"),
  reasons: voidReasons, // for level 'reason' / 'approval'
});

if (result.confirmed) {
  router.post(route("admin.orders.void", order), {
    reason: result.reason,
    approver_pin: result.pin,
  });
}
```

All confirm texts live in `lang/en` and `lang/bn`.

---

## 13. Forms & Validation

**The server (Form Request) is the source of truth.** Client-side checks are only for convenience (required, type, length via HTML attributes). Do not add a second validation library.

### Where messages appear

- **Field error:** directly under the field, `text-sm text-danger` with a small alert icon, red border on the input, `aria-invalid="true"`, linked with `aria-describedby`.
- **Summary banner** at the top of the form when there are 3 or more errors: "3 fields need your attention" with links that jump to each field.
- **Toast** only for outcomes ("Order saved", "Could not save, please try again"), never for field validation.
- **Server / network error (500, timeout):** error toast with a short message and a "Retry" button, plus a reference ID for support. No stack traces ever shown.

### Behavior

- On submit with errors: scroll to the first invalid field and focus it. **Never clear the form**; keep what the user typed (Inertia `useForm` does this).
- Validate on **blur** the first time, then on every keystroke once a field has an error, so the error disappears the moment it is fixed.
- Submit button: shows a spinner and is disabled while processing. Prevent double submit (also protected server-side by idempotency keys).
- Required fields are marked with a red `*` and the form footer says "\* required". Optional fields say "(optional)" instead.
- Helper text under a field (gray) explains format or purpose; replaced by the error when invalid.
- Error messages say **what is wrong and how to fix it** ("Mobile number must be 11 digits starting with 01", not "Invalid input"). Translated in `en` and `bn`.
- `Enter` submits single-purpose forms; in long forms `Ctrl+Enter` saves.
- Long forms (price list builder, scheme builder) show a sticky bottom action bar (Save / Cancel) and **auto-save a local draft** so a refresh doesn't lose work.
- Fields the user lacks permission to edit are shown read-only with a lock icon and tooltip, not hidden.

### Input components (build once, reuse everywhere)

- `FormField` (label + control + helper + error), `MoneyInput` (৳ prefix, grouped digits), `MobileInput` (01XXXXXXXXX mask, accepts +880), `QuantityStepper` (− / + with big touch targets), `Combobox` (searchable select with recent items), `MultiSelect` (chips), `DateRangePicker` (with presets), `FileUpload` (drag and drop, preview, progress, size/type errors), `ImageCropper`, `SwitchField`, `RadioCards` (for payment method, order type).
- Numeric inputs use `inputmode="decimal"`/`numeric` so mobile shows the right keyboard.
- Searchable selects debounce at 300ms and show "No results" with a "Create new" option when allowed.

---

## 14. Toasts & Notifications

- Library: `vue-sonner`, one wrapper `notify.success|error|warning|info|promise()`. Position: top-right on desktop, top-center on mobile.
- **Success:** 3s, auto-dismiss. **Error:** stays until dismissed (or 8s). **Warning:** 6s.
- Every toast has an icon and a short title; optionally one line of detail and one action button (e.g. **Undo**, **View order**, **Retry**).
- Show at most 3 at once; merge duplicates.
- Use `notify.promise()` for slow actions ("Generating report…" → "Report ready").
- **Undo pattern:** for safe reversible actions (archive, remove cart item) act immediately and offer Undo for 5 seconds instead of asking to confirm.
- **Notification center:** a bell in the top bar lists persistent alerts (approvals waiting, low stock, near-expiry batches, unsettled cash, over-limit customers) with unread badge, mark all read, and a link to the related page. Refreshed on page load and by light polling.
- **Haptic feedback** on the salesman app after saving an order or collection (optional) (short beep, volume and mute toggle saved per device). Respect browser autoplay rules by enabling sound after the first tap.

---

## 15. Dashboard Design

**Layout (top to bottom)**

1. **Sticky filter bar:** period presets (Today, Yesterday, 7 days, This month, Last month, Custom), branch selector (All branches / one branch), channel filter. Filters sync to the URL so a view can be shared and the Back button works.
2. **KPI row:** 4–6 cards (Sales, Collected, Outstanding, Overdue, Stock value, Gross margin). Each card: label, big value, **change vs previous period** (an arrow-up icon with +8.2% in emerald, or an arrow-down icon with −3.1% in rose; the icon, not color alone, carries the meaning), and a tiny sparkline. Clicking a card drills into the matching report with the same filters.
3. **Main charts:** sales trend (line/area), sales by channel (donut), top items (horizontal bar), profit breakdown.
4. **Alerts panel:** customers over credit limit, low stock, near-expiry batches, unsettled salesman cash, pending approvals, each with a one-click action.
5. **Tables:** top customers, top and slow products, salesman target versus achieved (compact, 5–10 rows with "View all").

**Behavior**

- Role-specific dashboards: owner sees all branches; branch manager sees their branch; salesmen see only their own customers, route and targets; supervisors see their team; warehouse staff see stock tasks.
- Cards and charts load independently with skeletons (never one global spinner). Show "Last updated 10:42" and a refresh button.
- Dashboards refresh on load and every few minutes (no real-time server in v1) with a subtle highlight flash (no jumping layout).
- Highlight problems with semantic color **and** icon: Collection rate below target shows rose with a triangle-exclamation icon, within target shows emerald with a circle-check icon. Targets are configurable.
- Money formatting follows Bangladesh grouping (**12,34,567**). Large dashboard numbers may use compact form (৳12.3L / ৳1.2Cr, or লাখ / কোটি in Bangla). Tooltips always show the exact value.
- Empty state for a new install: a setup checklist (add warehouse, import products and opening stock, import customers and balances, set price lists, add salesmen and routes, install the salesman app) instead of empty charts.
- Mobile: KPI cards become a horizontal swipe row; charts stack in one column.

---

## 16. Charts & Graphs

- Library: **Chart.js with `vue-chartjs`**, wrapped in one `ChartCard` component (title, subtitle, period, legend, export menu, loading, empty and error states). Do not mix chart libraries.
- Colors come from CSS tokens (read the CSS variables at runtime so dark mode works). Use a fixed, color-blind-safe series palette of up to 6 colors; semantic colors (success / danger) are reserved for profit / loss meaning.

| Data                                                         | Chart                                                                        |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| Trend over time (sales, collected, outstanding)              | Line or area, with optional previous-period dashed line                      |
| Compare categories (products, customers, salesmen)           | Horizontal or grouped bar, sorted descending                                 |
| Share of total (channels, payment methods)                   | Donut, **6 slices maximum** (group the rest as "Other"), total in the center |
| Busy hours / weekdays                                        | Column chart or heatmap grid                                                 |
| Collection build-up (billed -> discount -> collected -> due) | Waterfall-style stacked bar                                                  |
| Single KPI progress (collection rate vs target)              | Gauge or progress bar                                                        |

**Rules**

- Bar charts start at zero; no 3D, no gradients that distort meaning, no more than 6 series.
- Always label axes with units (৳, %, orders) and format ticks (compact numbers, Bangla digits when `bn`).
- Tooltips show exact values with currency and the date; hovering one series highlights it. Legends are clickable to toggle series.
- Animations are short (400ms) and disabled under `prefers-reduced-motion`.
- Every chart has: skeleton while loading, a friendly "No data for this period" state, and an error state with Retry.
- **Accessibility:** each chart has an `aria-label` summary and a "View as table" toggle showing the same data.
- Charts must be responsive and readable at 360px wide; hide minor gridlines and rotate or abbreviate labels on small screens.
- Export menu on every chart: PNG and CSV.

---

## 17. Reports UX

Every report page uses the same layout so users learn it once:

1. **Page header:** report title, short description, **Export** dropdown (PDF, Excel, CSV) and **Print** button on the right.
2. **Filter bar:** date range with presets, branch, channel, plus report-specific filters. An **Apply** button (reports can be heavy) and a **Reset** link. Filters are saved in the URL; "Save this view" stores a named filter set per user.
3. **Summary cards:** the 3–5 headline totals for the current filters.
4. **Chart** (optional, one per report) following section 16.
5. **Data table** with sortable columns, group-by option, column chooser, **totals row pinned at the bottom**, and pagination (or virtual scroll above 500 rows).
6. **Drill-down:** clicking a row or a number opens a slide-over with the underlying transactions.

**Rules**

- Numbers are right-aligned with tabular figures; text left-aligned; negative amounts show a minus sign **and** rose color; zero shows "–".
- Compare-to option ("vs previous period") adds change columns.
- Exports contain exactly what the screen shows (same filters, same totals) and a header block: business name, branch, report name, date range, generated by, generated at.
- **PDF reports** embed Noto Sans Bengali so Bangla renders correctly; Excel exports keep real numeric cells (not text) and a frozen header row.
- Heavy reports (more than about 10,000 rows or over 5 seconds) run as a **queued job**: show "We're preparing your report", then a notification and toast with a download link when done.
- The **Print view** is a clean layout (no sidebar, no buttons, page breaks between sections, A4) and also 80mm for receipt-style reports (shift close, daily summary).
- Reports must match the dashboard numbers exactly for the same filters.

---

## 18. Navigation & Layout

- **Sidebar:** grouped by module (Sales, Inventory, Customers, Delivery, Accounting, Reports, Settings), icons plus labels, collapsible to icons-only, remembers its state, shows only what the user's permissions allow. Active item uses the accent color.
- **Top bar:** branch switcher (always visible), global search, notification bell, language switch (EN / বাং), theme toggle, user menu.
- **Page header pattern:** breadcrumbs, title, optional description, and the single **primary action on the right** ("+ New invoice"). Secondary actions go in a "More" menu.
- **Command palette** (`Ctrl+K`): search pages, actions ("New customer"), customers by name, code or mobile; products by name or barcode; invoices by number.
- **Tabs** split long pages (Overview / Items / History) and sync with the URL.
- Active filters, page, sort and tab are kept in the URL, so refresh and Back behave correctly.
- Mobile admin: sidebar becomes a drawer; primary action becomes a floating button; tables switch to card lists.

---

## 19. Data Table Interactions

- Search (300ms debounce), quick filter chips (Status: Pending / Paid / Void), advanced filters in a popover, removable active-filter chips, "Clear all".
- Column sorting, show/hide columns, optional density toggle (Compact / Comfortable), sticky header and sticky first column on small screens.
- **Row actions:** an `ellipsis` icon menu on the right with the most common action also available by clicking the row (opens the slide-over). Destructive actions are last, in danger color, and go through a confirm dialog.
- **Bulk actions:** selecting rows reveals a bar ("3 selected: Export, Archive, Clear"); "Select all matching filters" is offered explicitly.
- **Inline edit** for simple fields (price, stock count): click, edit, `Enter` saves, `Esc` cancels, with a toast on save and an error under the cell on failure.
- Status cells use the shared `StatusBadge` (color + icon + label). Time columns show relative time ("5 min ago") with the exact time in a tooltip.
- **States:** skeleton rows while loading; empty state with an icon, a sentence and a button ("No suppliers yet. Add supplier"); different message when empty because of filters ("No results. Clear filters").
- Drag-and-drop reordering only where order matters (product categories and routes), with a visible drag handle and keyboard alternative.
- Large lists: server-side pagination (25 / 50 / 100 per page); never load everything.

---

## 20. Interaction & Feedback Patterns

- **Instant feedback:** every click gives a response within 100ms (pressed state, spinner, or optimistic update). Buttons that start work show a spinner and keep their width.
- **Optimistic updates** for fast, low-risk actions (add to cart, toggle availability) with rollback and an error toast on failure.
- **Skeletons** for content, spinners only inside buttons. No full-page blocking loaders.
- **Empty states:** icon, one helpful sentence, and a primary action. First-run screens add a short checklist or wizard.
- **Error pages:** friendly 403, 404, 500, and offline pages with a clear next step (Go to dashboard / Retry).
- **Tooltips** on icon-only buttons and truncated text; **inline help** (small "?" popover) for business terms such as "Collection rate".
- **Search selects** show recent and frequent choices first (customers, products, salesmen).
- **Touch:** minimum 44px targets in touch mode, no hover-only functionality, swipe actions optional but never the only way.
- **Safe destructive design:** delete buttons are never next to the primary action without spacing; disabled buttons explain why in a tooltip.
- **Print / reprint** actions confirm success with a toast and handle printer errors with a clear message and a "Download PDF" fallback.
- **Success moments** (payment complete, shift closed) use a brief check animation. Respect `prefers-reduced-motion`.

---

## 21. Microcopy & Tone

- Short, plain, friendly, and consistent. Buttons are verbs ("Save order", "Add item"). Avoid "Submit", "OK", "Yes/No".
- Error formula: **what happened + what to do** ("Could not reach bKash. Check the internet and try again.").
- Never blame the user, never show raw technical messages or exception text.
- Use one term for one concept everywhere (Order, Invoice, Challan, Receipt and Outstanding are different things; keep a glossary in `lang/en/glossary.php`).
- Bangla copy is written naturally, not word-for-word translation, and reviewed by a native speaker. Leave room for longer Bangla text in buttons and tables.
- Dates and times: `15 Oct 2026, 2:30 PM` (Asia/Dhaka), relative time for recent events. Numbers and currency go through the shared formatter only.

---

## 22. Shared UI Kit (build in Phase 0, reuse everywhere)

Build these once in `resources/js/components/ui-kit/` and use them everywhere. Do not create one-off versions inside pages.

`AppIcon`, `CustomerBanner`, `LineGrid`, `ProductPicker`, `AgingChart`, `SyncStatusPill`, `BarcodeScanner`, `DocumentPreview`, `PageHeader`, `ConfirmDialog` + `useConfirm()`, `ReasonDialog`, `ApprovalDialog` (manager PIN), `TypedConfirmDialog`, `SlideOver`, `FormField`, `MoneyInput`, `MobileInput`, `QuantityStepper`, `Combobox`, `MultiSelect`, `DateRangePicker`, `FileUpload`, `DataTable` (with filters, bulk actions, states), `FilterBar`, `StatusBadge`, `KpiCard`, `ChartCard`, `EmptyState`, `ErrorState`, `SkeletonRows`, `MoneyText`, `DateText`, `NotificationBell`, `CommandPalette`, `ThemeToggle`, `LanguageSwitch`, `ErrorPage`.

### UI Definition of Done (check for every screen)

- [ ] Uses UI-kit components only; no `alert()`, `confirm()` or `prompt()`
- [ ] Loading, empty, error and no-permission states exist
- [ ] Every destructive or irreversible action has the right confirmation level (section 12)
- [ ] Validation errors show under fields, focus the first error, and keep user input
- [ ] Success and failure produce a toast; slow work shows progress
- [ ] Works in light and dark mode, in `en` and `bn` (no overflow with longer Bangla text)
- [ ] Keyboard usable (tab order, `Esc`, `Enter`, visible focus) and labelled for screen readers
- [ ] Looks right at 1366×768, 1024px tablet and 390px phone (or documented as desktop only)
- [ ] Filters, tabs and pagination are reflected in the URL
- [ ] Numbers, dates and money use the shared formatters
- [ ] No emojis or Unicode symbol icons anywhere; icons come from `AppIcon` (Font Awesome, section 23)
- [ ] Controls follow the compact size scale (section 24) and tables follow the alignment standard (section 25)

---

## 23. Icons & No-Emoji Rule

**Never use emojis anywhere.** This covers the UI, Vue templates, Blade views, `lang/` files, toasts, SMS text, emails, receipts, PDFs and Excel exports, seeders and demo data, code comments, commit messages and documentation. Emojis render differently on every device, break thermal printers and PDF fonts, and look unprofessional in a business product.

Also avoid decorative Unicode symbols used as icons (arrows, check marks, warning signs, stars, geometric bullets). Use real icons instead. The currency symbol `৳` and normal punctuation are fine.

**Icon library: Font Awesome Free (one family only).**

- Install `@fortawesome/fontawesome-svg-core`, `@fortawesome/vue-fontawesome` and `@fortawesome/free-solid-svg-icons` (add `free-regular-svg-icons` only if needed). Use the tree-shaken SVG approach: import only the icons that are used. Do not load the Font Awesome web font or a CDN stylesheet in production.
- Wrap it in one component, `AppIcon` (`<AppIcon name="trash" />`), so size, color and accessibility are handled in one place. Pages never import Font Awesome directly.
- shadcn-vue components ship with Lucide icons by default. Replace those with `AppIcon` so only one icon style appears in the product.
- Prefer the **solid** style for actions and status, **regular** only for subtle or empty-state icons. Never mix several styles in one toolbar.

**Sizing and color**

- 14px inside a default 28px control, 12px in small controls, 16px in large controls, 20 to 24px in empty states and dialog header circles.
- Icons inherit text color (`currentColor`). Use semantic colors only for meaning (success, danger, warning).
- Decorative icons are `aria-hidden="true"`. An icon-only button always has an `aria-label` and a tooltip.
- Icon plus text buttons put the icon on the left with an 6px gap. Do not put an icon on every button; use them where they speed up recognition (add, edit, delete, print, export, search, filter).

**Standard icon names (use these for consistency)**

| Meaning                              | Font Awesome name                                                        |
| ------------------------------------ | ------------------------------------------------------------------------ |
| Add / create                         | `plus`                                                                   |
| Edit                                 | `pen-to-square`                                                          |
| Delete                               | `trash`                                                                  |
| Save                                 | `floppy-disk`                                                            |
| Close                                | `xmark`                                                                  |
| Search / filter                      | `magnifying-glass` / `filter`                                            |
| Export / download                    | `file-export` / `download`                                               |
| Print                                | `print`                                                                  |
| Settings                             | `gear`                                                                   |
| Notifications                        | `bell`                                                                   |
| Success / error / warning / info     | `circle-check` / `circle-xmark` / `triangle-exclamation` / `circle-info` |
| Increase / decrease                  | `arrow-up` / `arrow-down`                                                |
| Dashboard / invoices / collection    | `chart-line` / `file-invoice-dollar` / `hand-holding-dollar`             |
| Products / stock                     | `box` / `boxes-stacked`                                                  |
| Customers / salesmen                 | `store` / `user-tie`                                                     |
| Payments (cash, mobile wallet, card) | `money-bill-wave` / `mobile-screen` / `credit-card`                      |
| Visit / route                        | `location-dot` / `route`                                                 |
| Lock / approval                      | `lock` / `user-shield`                                                   |
| Delivery / returns                   | `truck-fast` / `rotate-left`                                             |
| Online / offline / syncing           | `wifi` / `plug-circle-xmark` / `rotate`                                  |

**Enforcement:** add a Pest test (`tests/Feature/NoEmojiTest.php`) that scans `app/`, `resources/`, `lang/`, `database/` and `docs/` and fails when it finds characters in these ranges: `U+1F000-1FAFF`, `U+2600-27BF`, `U+2B00-2BFF`, `U+25A0-25FF`, `U+FE0F`. It must run in CI.

---

## 24. Compact Controls & Sizing (professional density)

Back-office screens use small, precise controls. The goal is a calm, information-dense interface like modern admin tools, not large marketing-style form elements. **Touch mode (POS, KDS, rider view, customer website) keeps its larger targets; everything else uses the sizes below.**

### Size scale (buttons, inputs, selects, date pickers all share one height per row)

| Size      | Height           | Text     | Used for                                                          |
| --------- | ---------------- | -------- | ----------------------------------------------------------------- |
| `xs`      | 24px (`h-6`)     | 12px     | Actions inside table rows, tags, small toolbars                   |
| `default` | **28px (`h-7`)** | **13px** | Almost everything in the admin: forms, toolbars, filters, dialogs |
| `lg`      | 36px (`h-9`)     | 14px     | Customer website forms, mobile admin                              |
| `touch`   | 44-48px          | 15-16px  | the salesman app, delivery and warehouse screens only             |

Mixed heights inside one row are not allowed: if a button sits next to an input, they have the same height.

### Buttons

- Radius 6px (`rounded-md`), horizontal padding 10px (`px-2.5`), weight 500, no uppercase, no shadow. Icon-only buttons are square (28x28).
- Variants: **primary** (slate solid), **secondary** (white / surface with border), **ghost** (no border, for toolbars and row actions), **accent** (teal, only for the single most important action on a screen, such as Save and print), **destructive** (rose). Use **one primary button per region**; everything else is secondary or ghost.
- Button groups: 8px gap, aligned right in forms and dialogs, Cancel (secondary) left of the primary action.
- States: hover = slight tone change (100ms), pressed, focus = 2px accent ring with 1px offset, disabled = 50% opacity with `cursor-not-allowed` and an explanatory tooltip, loading = spinner replaces the icon and the width does not change.

### Form fields

- Height 28px, padding `px-2` (8px), radius 6px, border 1px `--border`, text 13px. Placeholder at 60% muted color. **Labels are never replaced by placeholders.**
- **Label** above the control: 12px, weight 500, 4px gap. Helper text below: 12px muted. Error text below: 12px danger with an icon.
- **Field spacing:** 12px between fields (`space-y-3`). Desktop forms use a two-column grid (`grid grid-cols-2 gap-x-4 gap-y-3`); long fields (address, notes) span both columns. Form pages are limited to `max-w-2xl` (672px) so lines of fields stay short and scannable. Group related fields under a small section title with a divider.
- **States:** default border `--border`; hover slightly darker; focus = accent border plus 2px ring; error = danger border and icon; disabled = muted background; read-only = plain text style with a lock icon.
- **Adornments:** currency, unit and icon prefixes/suffixes live inside the field (muted background block, same height): `৳ [ 0.00 ]`, `[ 0.8 ] kg`. Search inputs have a magnifying-glass icon and a clear button.
- Select, combobox and date picker share the same height; chevron icon 12px. Checkbox and radio 16px; switch 32x18px; textarea minimum 3 rows with vertical resize only.
- Number fields right-align their value when they hold money or quantities.

### Containers & spacing

- Cards: padding 12px, radius 8px, 1px border, no heavy shadow. Dialogs: radius 12px, padding 16px, header 16px semibold. Slide-over: padding 16px, header height 48px, footer with actions pinned to the bottom.
- Page padding 16px on desktop, 12px on mobile; gap between cards 12px; top bar height 48px; sidebar width 220px (56px collapsed), nav items 32px high with 13px text.
- Badges: height 20px, 11-12px text, `px-2`, `rounded-md`, optional 10-12px icon. Avatars 24px. Tooltips 12px.
- Never use border radius above 12px, oversized paddings, or large drop shadows in the admin.

---

## 25. Table Data Alignment Standard

Alignment is determined by the **type of data**, never by habit. A column header always has the **same alignment as its data**.

| Data type                             | Alignment              | Format rules                                                                                                                                        |
| ------------------------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Text (names, descriptions, notes)     | Left                   | One line, truncated with ellipsis, full text in a tooltip                                                                                           |
| Identifiers (invoice no., SKU, codes) | Left                   | Fixed format, `tabular-nums`, never wrapped                                                                                                         |
| Dates and times                       | Left                   | One fixed format (`15 Oct 2026, 2:30 PM`), never wrapped                                                                                            |
| Numbers (quantity, counts)            | **Right**              | `tabular-nums`, same decimals in the whole column                                                                                                   |
| Money                                 | **Right**              | Always 2 decimals in tables, thousands grouped (12,34,567.00). Show the currency once in the column header ("Revenue (৳)") instead of in every cell |
| Percentages                           | **Right**              | 1 decimal (`31.8%`)                                                                                                                                 |
| Status badges                         | Left                   | Badge with icon and text, fixed minimum width                                                                                                       |
| Yes/No, icon-only columns             | Center                 | Icon with `aria-label`                                                                                                                              |
| Row actions                           | **Right**, last column | Ghost icon buttons or a single `ellipsis` menu, fixed 40px wide, sticky on the right for wide tables                                                |
| Selection checkbox                    | Center, first column   | Fixed 40px wide                                                                                                                                     |

### Rules

- Row height 32px (`py-1.5`), header height 32px, cell padding `px-3`, text 13px, vertical alignment middle. Header text 12px, weight 600, muted color, sentence case (no uppercase, so Bangla headings stay correct).
- Numeric cells use `font-variant-numeric: tabular-nums` and `text-align: right` so digits and decimal points line up vertically. Never center numbers and never left-align them.
- Negative numbers show a minus sign **and** the danger color (`-1,250.00`). Zero and empty values show a muted en dash (`–`), not `0.00` and not blank.
- Do not put icons or badges inside numeric cells. Use a separate narrow status column so the numbers stay perfectly aligned.
- In a right-aligned header, put the sort arrow to the left of the label; in a left-aligned header, to the right, so the text stays flush with its data.
- **Widths:** minimum widths: money 96px, quantity 64px, date 112px, status 104px, actions 40px. Give the main text column the remaining space (`w-full`), and let the table scroll horizontally inside its own container when needed. Numbers, IDs and dates use `whitespace-nowrap`.
- **Lines:** 1px horizontal row borders only, no vertical borders, no zebra striping. Hover highlights the row; the selected row has a light accent tint. A cell can have a second line of muted 12px text (for example item name plus its category), but only in the first text column.
- **Totals row:** bold, 2px top border, pinned to the bottom of the table. The label ("Total") is left-aligned in the first column and each total sits in the same right-aligned column as its data.
- **Footer:** left shows "Showing 1-25 of 312", right shows the page-size select and the pager.
- **Small screens:** switch to a card list where each row shows label / value pairs with the value right-aligned.
- Tables in reports, dashboards, receipts (monospace columns on the printer) and exports use the same alignment rules.
