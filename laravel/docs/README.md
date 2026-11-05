# Implementation plan: Laravel app

This folder is the implementation plan for turning the HTML mockup (`../index.html` on `main`, live at https://muhghifari.github.io/infodata-ticketing/) into a working Laravel application in this `laravel/` folder.

**Status:** plan only, no application code written yet. · **Date:** 2026-11-05

The product requirements stay in [`../../PLAN.md`](../../PLAN.md). These docs describe *how* to build them in Laravel.

## Documents

| # | Doc | What it covers |
|---|---|---|
| 0 | [00-foundation.md](00-foundation.md) | Environment, enums, database schema, models, factories, seeders. Every later step depends on this. |
| 1 | [01-blade-views.md](01-blade-views.md) | Turning the mockup into layouts, Blade components and pages; splitting its CSS and JS |
| 2 | [02-routes.md](02-routes.md) | Every route, its name, middleware and controller |
| 3 | [03-authentication.md](03-authentication.md) | Login/logout, remember me, default Admin, forced password change, roles |
| 4 | [04-controllers.md](04-controllers.md) | Controllers, form requests, policies, actions, the SLA calculator |
| 5 | [05-tests.md](05-tests.md) | Pest test plan: unit tests, feature tests, policy matrix, what each file covers |

## What's installed (checked on the `laravel` branch)

| Package | Version | Notes |
|---|---|---|
| PHP | `^8.3` (this container: 8.3.6) | Boost guidelines mention 8.4; both work |
| laravel/framework | 13.x | |
| pestphp/pest | 5.x (+ pest-plugin-laravel) | Test runner |
| laravel/boost | 2.x | Guidelines in `CLAUDE.md`, skills in `.claude/skills/` |
| laravel/pint | 1.x | Code style; run `vendor/bin/pint --dirty --format agent` after PHP changes |
| tailwindcss | 4.x via `@tailwindcss/vite` | CSS-first config (`@theme`) |
| vite | 8.x + laravel-vite-plugin 3.x | |
| Database | SQLite by default; tests use SQLite `:memory:` | See decision D2 |

No auth starter kit (Breeze, Fortify, Jetstream) is installed, and no frontend framework (Livewire, Alpine, Vue).

## Order of work

The steps you listed are kept, with one addition in front:

```
0. Foundation   enums, migrations, models, factories, seeders
1. Blade views  layout + components + pages (fed by seeded data)
2. Routes       web.php with names, middleware, route model binding
3. Auth         login, logout, remember me, default admin, password change
4. Controllers  controllers, form requests, policies, actions, SLA service
5. Tests        written alongside steps 2–4, completed and reviewed in step 5
```

**Why step 0:** the Blade views, routes and controllers all need real models (tickets, teams, statuses). Building the schema first means the views render real seeded data instead of the mockup's fake JavaScript data, and nothing has to be redone.

**Why tests alongside:** each controller ships with its feature tests in the same commit, so step 5 is about filling gaps (policy matrix, edge cases, SLA unit tests) rather than writing everything at the end. This follows the Boost `testing-best-practices` skill.

Each step ends with `php artisan test --compact` green and Pint clean, and is committed separately on the `laravel` branch.

## Decisions that need your approval

The Boost guidelines say: no new dependencies and no new base folders without approval. These are the ones this plan needs:

| # | Decision | Recommendation | Alternative |
|---|---|---|---|
| D1 | **Frontend interactivity** (drag-and-drop, searchable picker, drawer, filter panel) | **Vanilla JS modules** in `resources/js/`, ported from the mockup. No new dependency. | Alpine.js or Livewire (both new dependencies) |
| D2 | **Database** | **SQLite** for development and tests (already configured). Production on **MySQL 8**. Schema kept portable (no database-specific column types). | PostgreSQL in production |
| D3 | **Styling** | **Port the mockup's CSS** into `resources/css/` files, with brand colors in Tailwind `@theme`. Use Tailwind utilities for small layout tweaks only. Faithful to the approved design, low risk. | Rewrite every view in Tailwind utility classes (more work, same look) |
| D4 | **Authentication** | **Hand-built** with Laravel's built-in `Auth` (login, logout, remember me, rate limit). Small: there is no registration, email verification or 2FA. No new dependency. | `laravel/fortify` (new dependency) |
| D5 | **New folders under `app/`** | `app/Enums`, `app/Actions`, `app/Support/Sla`, `app/Policies` | Put the logic in models and controllers |
| D6 | **Fonts** | Self-host Plus Jakarta Sans in `public/fonts/` (no external request; works on an office network) | Keep the Google Fonts link |
| D7 | **`.env.example` is missing from the branch** | The repo-root `.gitignore` has `.env.*`, which also ignores `laravel/.env.example`. Add `!laravel/.env.example` to the root `.gitignore` and commit the file. | — |
| D8 | **Time zone and locale** | `APP_TIMEZONE=Asia/Jakarta` in `.env`, `APP_LOCALE=id`, `APP_FAKER_LOCALE=id_ID`. Dates stored in UTC, shown in WIB. | — |

## Open product questions (from PLAN.md §11)

These don't block the build. The plan uses the defaults shown and keeps each one configurable:

1. SLA targets: P1 4 h, P2 8 h (24/7); P3 3 and P4 5 working days. Stored in `sla_policies`, editable in Admin.
2. Working hours: Mon–Fri 08:00–17:00 WIB, skipping holidays. Stored in config `ticketing.business_hours`.
3. SLA pauses in *Menunggu*-category statuses: on.
4. Login: username or email plus password. Microsoft 365 / Google SSO can be added later without changing the rest.
5. Hosting: anything that runs PHP 8.3+ and MySQL.

## Conventions used throughout

- UI text in **Bahasa Indonesia**; code, comments and docs in English.
- Named routes everywhere (`route('tickets.show', $ticket)`), never hard-coded URLs.
- Create files with `php artisan make:* --no-interaction` (model, migration, controller, request, policy, test, component, enum, class).
- Every write action: a Form Request for validation, a policy check for permission, and a `ticket_events` row in the same database transaction.
- Behavior comes from enums and categories, never from display names (for example, never `if ($status->label === 'Selesai')`).
