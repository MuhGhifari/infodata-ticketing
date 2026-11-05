# 1. Blade views: from mockup to layouts, components and pages

The mockup is one 1,300-line HTML file that renders everything with JavaScript from fake data. In Laravel, **pages are rendered on the server with Blade** from real data, and JavaScript only adds interactivity on top (drag-and-drop, the searchable picker, the drawer, the filter panel). Every page still works as a normal link or form if a script fails.

## 1.1 Folder structure

```
resources/
├── css/
│   ├── app.css                 @import "tailwindcss"; @theme tokens; @import of the files below
│   ├── base.css                reset, typography, focus ring
│   ├── layout.css              sidebar, top bar, page header, responsive rules
│   └── components/
│       ├── buttons.css  forms.css  badges.css  cards.css  tables.css
│       ├── board.css    drawer.css  modal.css   picker.css  toast.css
│       ├── filters.css  nav.css     login.css   timeline.css checklist.css
├── js/
│   ├── app.js                  boots every module below on DOMContentLoaded
│   ├── http.js                 fetch wrapper: CSRF header, JSON, error → toast
│   ├── picker.js               searchable picker (bottom sheet on mobile)
│   ├── drawer.js               opens ticket detail in a side drawer, keeps URL in sync
│   ├── board.js                drag-and-drop between columns → status endpoint
│   ├── status-dialog.js        asks for reason/resolution before submitting a status change
│   ├── filters.js              mobile "Filter" panel, auto-submit filter form
│   ├── nav.js                  sidebar "Papan Tim" dropdown + team search, mobile menu
│   ├── checklist.js            tick/rename/add/delete steps without a full reload
│   └── toast.js                flash messages
└── views/
    ├── layouts/
    │   ├── app.blade.php       sidebar + top bar + <main> + drawer/modal/picker/toast roots
    │   └── guest.blade.php     login and change-password pages
    ├── partials/
    │   ├── sidebar.blade.php   brand, menu, Papan Tim dropdown, Admin link (admins only), user + logout
    │   └── topbar.blade.php    search form, "Buat Tiket", notifications
    ├── components/             see 1.3
    ├── auth/
    │   ├── login.blade.php
    │   └── change-password.blade.php
    ├── dashboard.blade.php
    ├── board/index.blade.php
    ├── tickets/
    │   ├── index.blade.php     list
    │   ├── show.blade.php      full page; wraps the detail in @fragment('detail')
    │   ├── create.blade.php    full page; wraps the form in @fragment('form') for the modal
    │   └── partials/ status-dialog.blade.php, timeline.blade.php, steps.blade.php
    └── admin/
        ├── sla.blade.php       statuses.blade.php      step-templates.blade.php
        ├── teams.blade.php     users.blade.php
        └── partials/tabs.blade.php
```

## 1.2 Mapping mockup → Blade

| Mockup (JS function) | Blade |
|---|---|
| `renderLogin()` | `auth/login.blade.php` (layout `guest`) |
| `renderNav()` | `partials/sidebar.blade.php`; counts come from a **view composer** (`SidebarComposer`) so every page gets them without repeating queries in controllers |
| top bar markup | `partials/topbar.blade.php`; search is a `GET` form to `tickets.index?q=` |
| `renderDashboard()` | `dashboard.blade.php` + `<x-kpi>`, `<x-workload-bar>` |
| `renderBoard()` + `card()` | `board/index.blade.php` + `<x-ticket-card>` |
| `renderList()` | `tickets/index.blade.php` + `<x-sortable-th>` |
| `renderDrawer()` | `tickets/show.blade.php` (`@fragment('detail')` is returned alone when the drawer asks for it) |
| `openCreate()` | `tickets/create.blade.php` (`@fragment('form')` for the modal) |
| `requestStatus()` dialog | `tickets/partials/status-dialog.blade.php` (a `<dialog>` element) |
| `renderAdmin()` tabs | `admin/*.blade.php` + `admin/partials/tabs.blade.php` |
| `filterBar()` | `<x-filter-bar>` |
| `openPicker()` | `<x-picker>` (see 1.4) |
| `toast()` | session flash `status` → `<x-toast>`; JS toasts for fetch actions |

## 1.3 Blade components (`resources/views/components`)

Anonymous components unless logic is needed (then a class in `app/View/Components`). Templates use `$attributes->merge()` so callers can add classes.

| Component | Props | Replaces in mockup |
|---|---|---|
| `<x-icon name="board" :size="16"/>` | name, size, stroke | `icon()` |
| `<x-avatar :user="$u" :size="24"/>` | user (nullable → dashed "unassigned" circle) | `av()` |
| `<x-badge.priority :priority/>` | `Priority` enum | `prioBadge()` |
| `<x-badge.status :status/>` | `Status` model | `statusBadge()` |
| `<x-badge.type :type/>` | `TicketType` | `typeBadge()` |
| `<x-sla-chip :ticket :short="true"/>` | ticket | `slaChip()` |
| `<x-progress :ticket mini/>` | ticket | `pgHtml()`, card mini bar |
| `<x-ticket-card :ticket :show-team="false"/>` | ticket | `card()` |
| `<x-kpi tone="bad" icon="flame" :value :href>` + slot | | KPI tiles |
| `<x-card>` with `title`, `subtitle`, `actions` slots | | `.card` blocks |
| `<x-filter-bar :filters :board="true" :show-status="false"/>` | `TicketFilters` | `filterBar()` |
| `<x-view-toggle current="board|list"/>` | | board/list icon toggle |
| `<x-picker name label :options :value :searchable/>` | options: array of `{value,label,sub,icon}` | `openPicker()` |
| `<x-modal id>` / `<x-drawer>` | | modal & drawer shells |
| `<x-nav-link :href :active icon count>` | | sidebar links |
| `<x-empty>` | | empty states |
| `<x-toast/>` | reads session flash | toasts |

Ticket-related presentation logic (SLA label, progress) lives in the model/`SlaSnapshot`, not in Blade, so components stay thin.

## 1.4 The picker in Blade

`<x-picker>` renders a **real form control first**: a hidden `<input name="…">` plus a `<button>` showing the current value, and a `<template>` holding the options. `picker.js` turns it into the searchable dropdown / bottom sheet from the mockup. On choose, it sets the hidden input and either:
- submits the surrounding form (ticket detail fields: `PATCH tickets.update`, or the status dialog), or
- just updates the value (create form).

Without JavaScript the component falls back to a native `<select>` inside `<noscript>`.

## 1.5 Partial rendering (drawer and modal)

Laravel's `fragment()` is used so one view serves both the full page and the drawer:

```php
return view('tickets.show', $data)->fragmentIf($request->header('X-Drawer') === '1', 'detail');
```

`drawer.js` fetches `tickets/{number}` with `X-Drawer: 1`, injects the fragment, and `history.pushState`s the URL so the drawer is shareable and the back button closes it. Opening the same URL directly shows the full page.

## 1.6 CSS approach (decision D3)

1. Copy the mockup's `<style>` block into the files in 1.1, grouped by component. No visual changes.
2. Move the color tokens into Tailwind `@theme` so both custom CSS and utilities share them:
   ```css
   @theme {
     --font-sans: 'Plus Jakarta Sans', ui-sans-serif, system-ui, sans-serif;
     --color-brand: #E40114;  --color-brand-700: #B5000F;  --color-brand-50: #FFF1F2;
     --color-ink: #231917;    --color-ink-2: #4E4442;      --color-muted: #857B79;
     --color-line: #ECE6E5;   --color-bg: #F8F6F5;
   }
   ```
   (The mockup's `var(--brand)` etc. become `var(--color-brand)`.)
3. Keep the mobile rules from the mockup: 16px form text on phones (stops iOS zoom), collapsible filter panel, bottom-sheet picker, horizontal-scrolling board.
4. Logo: `public/images/infodata-logo.jpg` (from `assets/brand/`), not a base64 string.

## 1.7 Done when

- Every mockup screen exists as a Blade page rendering seeded data: login, change password, dashboard, team board, all-teams board, ticket list, ticket detail (page + drawer), create ticket (page + modal), the five admin tabs.
- Screenshots at 1440 px and 390 px match the mockup.
- No database queries inside Blade files; `php artisan view:cache` succeeds.
