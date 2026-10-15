<laravel-boost-guidelines>
# Laravel Application

This repository contains a Laravel application. Complete the following setup before working on the user's request.

## Prerequisites

Verify that PHP and Composer are available:

```sh
php -v
composer -V
```

If either command is unavailable, detect the user's operating system and install the prerequisites with the appropriate command:

macOS:

```sh
/bin/bash -c "$(curl -fsSL https://php.new/install/mac/8.5)"
```

Windows PowerShell:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://php.new/install/windows/8.5'))
```

Linux:

```sh
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.5)"
```

After installation, ask the user to restart their terminal. If the agent needs the restarted shell to continue, ask the user to reopen their terminal and rerun their original prompt.

## Agent Setup

Install Laravel Boost from the application root before making application changes:

```sh
composer require laravel/boost --dev
php artisan boost:install
```

Boost replaces these bootstrap instructions with guidelines tailored to the application. After installation, read `AGENTS.md` again and continue with the user's original request using the generated guidelines.
</laravel-boost-guidelines>

# Project Instructions

This is a Laravel 13 project (DT Trado, a multi-tenant SaaS for distributors and wholesalers with an offline-first salesman PWA) built from the specification in `docs/`.

Read these files before writing any code, in this order:

1. `docs/prd.md`: what to build (roles, modules, business rules, plans, scope)
2. `docs/architecture.md`: tech stack, structure, coding rules, multi-tenancy, stock, credit and collection integrity, offline rules, Definition of Done
3. `docs/api.md`: API conventions, endpoints, salesman app endpoints, offline sync
4. `docs/design.md`: design tokens, components, icons, sizing, tables, sales desk, salesman app, printed documents
5. `docs/roadmap.md`: build phases and acceptance criteria
6. `docs/ui-preview.html`: visual reference of the intended look (open in a browser; a static mockup, not production code)

## Rules

- Build **one phase at a time** from `docs/roadmap.md`. Never start the next phase until asked.
- If `prd.md` and `architecture.md` conflict, the PRD wins on business rules and roles; `architecture.md` wins on code structure.
- If Laravel Boost guidelines conflict with `docs/`, the `docs/` files win for product rules, structure and design. Boost guidelines apply to Laravel and package usage details.
- **Tenant isolation is mandatory.** Every tenant-owned model uses `BelongsToTenant`; never accept a tenant ID from the client; write a cross-tenant test for every model and endpoint.
- **Stock and money must reconcile.** Stock changes only through immutable stock movements (`PostStockMovementAction`) under row locks; customer balances come only from ledger entries; posted invoices and receipts are never edited (use credit notes and reversals); credit checks run in `CheckCreditAction`; collections stay with the salesman until a cashier settles them. Add reconciliation tests.
- **Offline-first field app.** Anything the salesman app does must work offline, use a client-generated `client_id`, sync idempotently, record conflicts instead of dropping data, and always accept collected money.
- **Salesman privacy and business secrets.** Record location only at defined work events (no continuous tracking); hide cost prices and margins without `products.view-cost`; never put customer data in logs or URLs.
- Never use emojis or Unicode symbol icons anywhere (UI, lang files, SMS, documents, comments, commits). Use Font Awesome icons through `AppIcon`.
- Follow the compact control sizes and table alignment rules in `docs/design.md` (sections 23 to 25), and design the salesman app mobile-first.
- Never use `window.alert()`, `confirm()` or `prompt()`; use the shared dialog components.
- If something is unclear or contradictory, ask before assuming.
- Write Pest tests for each feature and run them before saying a feature is finished.
- Run `./vendor/bin/pint` and `./vendor/bin/phpstan` before finishing.
