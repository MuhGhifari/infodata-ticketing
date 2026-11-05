# Handoff note (for the next AI/developer)

**Date:** 2026-11-05 · **Repo:** `MuhGhifari/infodata-ticketing` · **Work branch:** `laravel` · **App folder:** `laravel/`

## Project in one paragraph

Internal engineer-case ticketing app for **PT. Info Solusindo Data Utama** (Jakarta). Tickets have type (Insiden / Permintaan Layanan / Perubahan / Masalah Berulang), team, priority (Kritis/Tinggi/Sedang/Rendah) with an auto-calculated SLA deadline, assignee, per-team statuses (each status has a category: open/active/paused/done; paused pauses the SLA, done requires a resolution note), a checklist of work steps (progress = ticked ÷ total), and a full history. Dashboard shows active, breached and at-risk tickets and workload per team/engineer. UI language: **Bahasa Indonesia**. Brand: red `#E40114`, near-black `#231917`, logo in `assets/brand/infodata-logo.jpg`.

## Where things are

| What | Where |
|---|---|
| Product requirements | `PLAN.md` (repo root) |
| Approved UI design (clickable HTML mockup, vanilla JS, demo data) | `index.html` on `main`; live at https://muhghifari.github.io/infodata-ticketing/ (GitHub Pages, deploy from branch `main`, root). Demo login: `admin / admin123` |
| Laravel implementation plan | `laravel/docs/README.md` + `00`–`05` docs in this folder. **Read these first.** |
| Fresh Laravel install | `laravel/` (Laravel 13, Pest 5, Tailwind 4, Vite 8, Laravel Boost) |
| Project rules for AI agents | `laravel/CLAUDE.md` (Boost guidelines) and skills in `laravel/.claude/skills/` |

## What the user asked for (in order)

1. Convert the HTML mockup into organized Blade templates
2. Build the routes
3. Build authentication
4. Build the controllers
5. Build the tests

The plan adds a step 0 (enums, migrations, models, factories, seeders) before step 1, and writes tests alongside each step. **Latest instruction:** "do the templating and basic routing first so I can click things", meaning step 0 + step 1 + routes + a simple working login. Pages read real seeded data. Save actions (status change, field edits, notes, steps, create form, admin forms) can show "Belum tersedia" until step 4.

## Current state

- `laravel/docs/` plan is committed and pushed. **No application code has been written yet.**
- Nothing in `laravel/` was modified (lock files untouched).

## Blocker: PHP 8.4 is required

The project's `composer.lock` needs **PHP ≥ 8.4** (Pest 5, PHPUnit and Symfony 8 use 8.4 syntax). The previous cloud container only had PHP 8.3.6. Installing 8.4 via `ppa:ondrej/php` or a downloaded static binary was blocked by session permissions. Running on 8.3 with `--ignore-platform-req=php` fails with a parse error in PHPUnit. **Do not downgrade dependencies without the user's approval.**

The fix suggested to the user: add this to the cloud environment setup script.
```bash
add-apt-repository -y ppa:ondrej/php && apt-get update && apt-get install -y php8.4-cli php8.4-sqlite3 php8.4-mbstring php8.4-xml php8.4-curl php8.4-zip php8.4-intl php8.4-bcmath && update-alternatives --set php /usr/bin/php8.4
```
Then, in `laravel/`: `composer install`, `npm install`, create `.env` (see below), `php artisan key:generate`, `php artisan migrate --seed`, `npm run build`.

## Decisions (recommended in docs/README.md; the user didn't object, use these defaults)

- **D1** Interactivity: vanilla JS modules ported from the mockup (no Alpine/Livewire).
- **D2** SQLite for dev and tests (already configured; tests use `:memory:`), MySQL in production; keep the schema portable.
- **D3** Port the mockup's CSS into `resources/css/` files; brand tokens in Tailwind `@theme`.
- **D4** Hand-built auth with Laravel's `Auth` (no Breeze/Fortify).
- **D5** New folders `app/Enums`, `app/Actions`, `app/Support/Sla`, `app/Policies`.
- **D6** Font Plus Jakarta Sans (a Google Fonts link is fine until self-hosting is set up).
- **D7** `laravel/.env.example` is **missing from git** because the repo-root `.gitignore` has `.env.*`. Add `!laravel/.env.example` to the root `.gitignore` and commit a standard Laravel `.env.example` (with `APP_NAME="PT. Info Solusindo Data Utama"`, `APP_TIMEZONE=Asia/Jakarta`, `APP_LOCALE=id`, `APP_FAKER_LOCALE=id_ID`, `DB_CONNECTION=sqlite`, and empty `ADMIN_USERNAME/ADMIN_EMAIL/ADMIN_PASSWORD`).
- **D8** Time zone Asia/Jakarta, locale `id`.

## Requirements the user has stated (don't regress these)

- **Default Admin: Muamar Azzriel Fattah**, username `admin`. Password from `ADMIN_PASSWORD` (or generated and printed once); must change it on first login. No password in git.
- App name shown in the UI is just the company name **"PT. Info Solusindo Data Utama"**. Do **not** use "Tiket Engineer". No "Mockup" labels.
- **Minimal, uncluttered UI**: one "Buat Tiket" button (top bar only); board title is the team switcher; board/list icon toggle; compact cards (number + priority, title, SLA chip + mini progress "2/5" + avatar).
- Login page: one white card on the app's light gray background (logo, company name, username/email, password, "Ingat saya", Masuk, "Lupa password? Hubungi Admin…", footer). No marketing panel.
- **Each team has its own board and its own statuses** (admin-editable per team); the "Semua tim" board groups by the 4 categories.
- Sidebar: Dashboard, Tiket Saya, Daftar Tiket, a collapsible **"Papan Tim" dropdown with team search**, Admin (admins only).
- **Mobile:** filters collapse into one "Filter" button that opens a panel; form text must be **16px on phones** (iOS zoom); choice fields (Status, Prioritas, Penanggung jawab, Tim, Jenis) use a **custom searchable picker** (bottom sheet on mobile, dropdown on desktop), not native `<select>`.
- Reopening a closed ticket: Lead Tim/Admin only.

## Working conventions

- Follow `laravel/CLAUDE.md`: `php artisan make:* --no-interaction`, Pest tests (`php artisan make:test --pest`), run `vendor/bin/pint --dirty --format agent` after PHP changes, no new dependencies or base folders without approval, don't delete existing tests without approval.
- Commit to the `laravel` branch, push with `git push -u origin laravel`. Commit trailers used so far: `Co-Authored-By: …` and `Claude-Session: …`.
- Before finishing, verify pages in a browser (Playwright/Chromium is preinstalled in the cloud container) at 1440 px and 390 px.

## Suggested next steps

1. Get PHP 8.4 working (see Blocker), then `composer install && npm install`.
2. Fix D7 (`.env.example`).
3. Step 0 from `00-foundation.md`: enums, migrations, models, factories, seeders (`DemoSeeder` ports the mockup's 6 teams, their status flows, 15 users and ~40 tickets).
4. Simple login/logout (`03-authentication.md` §3.1) so pages can sit behind `auth`.
5. Step 1 from `01-blade-views.md`: layouts, components and pages; GET routes from `02-routes.md`.
6. Smoke tests: each page returns 200 for a logged-in user and redirects guests.
7. Commit, push, and tell the user how to run it (`composer run dev`).
