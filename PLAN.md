# Engineer Case Ticketing — Plan

Internal web app for **PT. INFO SOLUSINDO DATA UTAMA** (Info Data) for logging and tracking engineer cases across all teams.

**Status:** draft v2 (rewritten from the client brief) · **Last updated:** 2026-11-04

Client address: Menara Hijau, Jl. MT Haryono Kav. 33, RT.3/RW.2, Cikoko, Pancoran, Jakarta Selatan 12770

---

## 1. Background & goals

Today each team's topics and cases are scattered across chats and personal notes, which makes them hard to monitor. In the new app, every case is logged as a ticket with a clear structure.

**Goals (from the brief)**
- Cases no longer live in chats or personal notes.
- Team leads can see progress and workload directly.
- Priorities and handling deadlines (SLA) are the same across all teams.

**Scope change from v1 of this plan:** this is a **single-company internal tool**, not a multi-tenant SaaS. That removes workspaces, invites, tenant isolation and billing. In their place we get **teams** inside one company, **fixed statuses**, **SLA**, and a **dashboard**.

### Out of scope for v1
Customer-facing portal, email-to-ticket, file attachments, email/WhatsApp notifications, custom fields, automation rules, mobile apps, integrations (monitoring, chat). Each is listed in §12 as a later option.

---

## 2. Users & roles

| Role | Who | Can do |
|---|---|---|
| **Admin** | IT/app owner | Everything: manage users, teams, SLA policy, categories |
| **Lead Tim** | Team lead | All Engineer actions, plus reassign and edit any ticket in their team(s), see the team dashboard |
| **Engineer** | Engineers | Create tickets for any team; update tickets assigned to them or in their team; add work notes |
| **Viewer** *(optional)* | Management | Read-only: all tickets and the dashboard |

A user belongs to one or more teams. Permissions are enforced on the server; the UI only hides controls the user can't use. Only Admins see the Admin menu; only Lead Tim and Admin can reopen a closed ticket.

### Login & default admin

- Everyone signs in on a **login page** (username or email + password, "Ingat saya", "Lupa password?" → contact Admin). There is no self-registration; Admin creates accounts.
- On first install the database is seeded with one **default Admin: Muamar Azzriel Fattah** (username `admin`). The initial password is generated during setup (or set via an environment variable) and **must be changed on first login**; it is never stored in the code.
- The mockup shows demo accounts with visible passwords (`admin / admin123`, `rizky / demo123`, `fajar / demo123`) purely so the client can click through roles. That box does not exist in the real app.

---

## 3. Ticket model (from the brief)

| Field | Values / rules |
|---|---|
| **Nomor** (ticket number) | Auto-generated, prefixed by type: `INC-000123`, `SR-…`, `CHG-…`, `PRB-…` |
| **Judul** (title) | Required |
| **Jenis** (type) | `Insiden` (Incident) · `Permintaan Layanan` (Service Request) · `Perubahan` (Change) · `Masalah Berulang` (Problem) |
| **Tim** (team) | Required; the team responsible |
| **Prioritas** (priority) | `Kritis` (P1) · `Tinggi` (P2) · `Sedang` (P3) · `Rendah` (P4) |
| **Deskripsi** (description) | Markdown |
| **Penanggung jawab** (assignee) | One user from the ticket's team; may be empty while the ticket is in a *Belum dikerjakan* status |
| **Pelapor** (reporter) | The user who opened it (automatic) |
| **Status** | One of **its team's** admin-defined statuses (see below). Defaults from the brief: `Terbuka` → `Dikerjakan` ⇄ `Menunggu` → `Selesai` |
| **Batas SLA** (SLA deadline) | **Calculated automatically** from priority when the ticket is created; recalculated if priority changes |
| **Langkah pengerjaan** (work steps) | A checklist copied from the ticket type's template when the ticket is created; engineers can tick, add, rename or delete steps. **Progress = ticked steps ÷ total steps** |
| **Riwayat** (history) | Work notes plus every field and status change, with who made it and when |

### Statuses (admin-defined, per team)

**Every team has its own board and its own list of statuses.** Admins manage each team's list in **Admin → Status & alur** (pick the team, then add, rename, recolor, reorder and delete). The order is the column order on that team's board. Each status belongs to one of four fixed **categories**, and all rules hang off the category, never off the status name:

| Category | SLA | Required on entry | Notes |
|---|---|---|---|
| **Belum dikerjakan** (open) | Running | — | The first status in this category is where new tickets start |
| **Sedang dikerjakan** (active) | Running | — | |
| **Menunggu** (paused) | Paused | Reason | |
| **Selesai** (done) | Stopped | Resolution note | Counts as closed in the dashboard and "selesai tepat waktu" |

Defaults (from the client brief): `Terbuka` (open) → `Dikerjakan` (active) ⇄ `Menunggu` (paused) → `Selesai` (done). Example additions: `Verifikasi User` (paused), `Eskalasi Vendor` (paused), `Review` (active).

Mockup examples per team: Network adds `Menunggu Vendor` and `Verifikasi User`; Database adds `Analisis`; Aplikasi uses `Backlog → Dikerjakan → Testing → Menunggu → Rilis / Selesai`; Security adds `Investigasi` and `Mitigasi`.

- **Team boards:** each team's board shows that team's statuses as columns.
- **All-teams board:** because teams' statuses differ, the "Semua tim" board groups cards by **category** (4 columns) and shows each card's own status on the card. Dropping a card on a category moves it to the first status of that category in the ticket's own team.
- **Moving a ticket to another team:** its status switches to the first status of the same category in the new team's flow (falls back to the new team's starting status), and this is logged in the history.
- Dashboard and reports compare teams by category, so different flows never break the numbers.

Rules:
- There must always be at least one *Belum dikerjakan* status and one *Selesai* status; names must be unique.
- Deleting a status that still has tickets requires choosing a status to move them to; each move is written to the ticket history.
- Changing a status's category updates the SLA data of tickets already in it (pause starts/ends, resolved time set/cleared).
- Any status can move to any other status (no transition rules in v1).
- Moving into a *Selesai* status with unticked steps warns and offers to tick them.
- Moving out of a *Selesai* status (reopening) is Lead/Admin only and requires a reason.

### SLA rules (proposed defaults; admin can edit them)

| Priority | Target resolution time | Example |
|---|---|---|
| Kritis (P1) | 4 hours | Production service down |
| Tinggi (P2) | 8 hours | Major function degraded |
| Sedang (P3) | 3 business days | Normal request |
| Rendah (P4) | 5 business days | Minor or cosmetic |

- Times are in **WIB (Asia/Jakarta)**.
- The SLA clock **pauses while the ticket is in any *Menunggu*-category status**, and the deadline moves forward by the paused time. *(Needs client confirmation.)*
- SLA states shown everywhere: **Aman** (on track), **Hampir lewat** (≥ 75% of the time used), **Lewat SLA** (breached).
- SLA rules are stored in a table, not hard-coded, so the client can tune them without code changes.

---

## 4. Screens

All UI text is in **Bahasa Indonesia**. Branding comes from the client logo (`assets/brand/infodata-logo.jpg`): red **#E40114**, near-black **#231917**, white.

1. **Login**: username/email + password, remember me; branded with the Info Data logo
2. **Dashboard** (home page)
   - KPI tiles: **Tiket aktif** (active tickets), **Lewat SLA** (breached), **Hampir lewat SLA** (at risk), **Selesai minggu ini** (done this week)
   - **Beban kerja per tim** (workload per team): active tickets per team, stacked by priority
   - Workload per engineer within a team (for leads)
   - Lists: tickets about to breach, recently opened
   - Filters: team, type, priority, status, assignee, date range, plus search
3. **Papan** (board, kanban): one board per team (listed in the sidebar under *Papan tim*), one column per status of that team, plus an all-teams board grouped by category. Scrolls sideways when a team has many statuses. Drag a card to change its status; the dialog asks for a reason or resolution note when needed. Cards show the number, title, type, priority, assignee and an SLA countdown. Can be filtered per team, or "Tiket saya" (my tickets).
4. **Daftar tiket** (ticket list): a sortable table with the same filters and search, plus CSV export.
5. **Buat tiket** (new ticket form).
6. **Detail tiket**: fields, SLA bar, and a timeline of work notes and changes, with a box to add a work note.
7. **Admin**: users and roles, teams and members, SLA policy, default work-step templates per ticket type.

**Progress vs SLA:** cards, the list and the ticket detail show two separate bars. The SLA bar (red/amber/green) shows *time used*; the progress bar (neutral) shows *work done* from the checklist.

---

## 5. Tech stack

| Layer | Choice |
|---|---|
| Frontend | React 19 + TypeScript + Vite, TanStack Router + TanStack Query |
| UI | Tailwind CSS + shadcn/ui (Radix), themed with the Info Data colors |
| Board drag-and-drop | dnd-kit |
| Charts | Recharts |
| Forms / validation | React Hook Form + Zod (schemas shared with the API) |
| Dates | date-fns + date-fns-tz (WIB), with a business-hours/holiday calendar for SLA |
| Backend | Node + Fastify + TypeScript |
| Database | PostgreSQL + Prisma |
| Auth | Session cookies; email/password first. SSO via Microsoft 365 or Google Workspace if the company uses one (see §11) |
| Background jobs | A small scheduler (e.g. pg-boss) that recalculates SLA states every minute |
| Monorepo | pnpm workspaces: `apps/web`, `apps/api`, `packages/shared` |
| Tests | Vitest (SLA logic gets thorough unit tests), Playwright for E2E |
| Deploy | Docker Compose (web + api + Postgres) on a company server or VPS |

Realtime WebSockets from v1 of this plan are dropped for now. The dashboard and board refetch every 30 s and when the window regains focus, which is enough for an internal tool. WebSockets can be added later.

---

## 6. Data model

```
users ──< team_members >── teams ──< tickets ──< ticket_events
                                        │
sla_policies (priority → minutes)       └── assignee_id → users
holidays (for business-day SLA)
```

- **users** — id, name, email (unique), password_hash, role (`admin` | `lead` | `engineer` | `viewer`), active, created_at
- **statuses** — id, team_id → teams, label, color, category (`open` | `active` | `paused` | `done`), position, archived_at? (archived instead of hard-deleted, so old history still shows the name)
- **teams** — id, name, code, active
- **team_members** — team_id, user_id, is_lead
- **tickets** — id, number (`INC-000123`), type, title, description, team_id, priority, status_id → statuses, assignee_id?, reporter_id, sla_due_at, sla_paused_at?, sla_paused_total_minutes, first_response_at?, resolved_at?, resolution_note?, created_at, updated_at, search_vector (tsvector)
- **ticket_steps** — id, ticket_id, position, text, done, done_by?, done_at?
- **step_templates** — type, position, text (default steps per ticket type, editable by Admin)
- **ticket_events** — id, ticket_id, actor_id, kind (`created` | `note` | `step_done` | `step_undone` | `step_added` | `step_removed` | `step_renamed` | `status_changed` | `assigned` | `priority_changed` | `team_changed` | `field_changed`), body (for notes/reasons), payload jsonb (`{from, to}`), created_at
- **sla_policies** — priority, resolution_minutes, business_hours_only (bool)
- **holidays** — date, name (Indonesian public holidays and cuti bersama)
- **ticket_counters** — type, next_number (incremented inside a transaction so numbers never repeat)

Key indexes: `(status_id, team_id)`, `(assignee_id, status_id)`, `(sla_due_at) WHERE resolved_at IS NULL`, plus a GIN index on `search_vector`.

Every change to a ticket writes a `ticket_events` row in **the same database transaction**, so the history is always complete.

---

## 7. API (REST)

```
POST   /auth/login | /auth/logout      GET /auth/me

GET    /tickets?q=&team=&type=&priority=&status=&assignee=&sla=breached|at_risk&from=&to=&sort=&page=
POST   /tickets
GET    /tickets/:id                    # including events
PATCH  /tickets/:id                    # title, description, type, team, priority, assignee
POST   /tickets/:id/status             # { statusId, reason? | resolution? }  (rules from the status category)
POST   /tickets/:id/notes              # { body }
GET    /tickets/export.csv?…same filters

GET    /dashboard/summary?team=&from=&to=   # KPI tiles
GET    /dashboard/workload?…                # per team / per engineer

GET|POST|PATCH  /admin/users, /admin/teams, /admin/teams/:id/members
GET|PUT         /admin/teams/:id/statuses # team's whole ordered list; body may include { moves: {deletedId: targetId} }
GET|PUT         /admin/sla-policies
GET|POST|DELETE /admin/holidays
```

---

## 8. Search & filters

- Full-text search over number, title, description and work notes (Postgres `simple` config, which works fine for mixed Indonesian/English text), plus exact match on ticket numbers.
- Filters are kept in the URL, so a lead can bookmark or share a view (e.g. "Tim Network – Lewat SLA").

---

## 9. Security & non-functional

- Internal only: put it behind the office network/VPN or company SSO if possible.
- HttpOnly + Secure session cookies, CSRF protection, Argon2id password hashing, rate-limited login.
- All input validated with Zod; markdown sanitized when rendered.
- Daily Postgres backups (`pg_dump`) with a restore test.
- Personal data is minimal (names and emails); keep the data in Indonesia or on the company's own server in line with **UU PDP (Law 27/2022)**.
- Performance targets: dashboard loads in < 1 s with 50k tickets; the board loads in < 500 ms.

---

## 10. Milestones

| # | Milestone | Deliverable |
|---|---|---|
| **M0** | Scaffold | Monorepo, Docker Compose with Postgres, Prisma schema, lint + CI, Info Data theme and layout |
| **M1** | Auth & admin | Login page, sessions, roles, default Admin (Muamar Azzriel Fattah) with forced password change, user and team management, seed data |
| **M2** | Tickets core | Create, list, detail and edit tickets; numbering; work notes; history timeline |
| **M3** | Status & SLA | Status flow with reasons; SLA calculation (business hours, holidays, pause in Menunggu-category statuses); admin-defined statuses with categories with full unit tests; SLA badges |
| **M4** | Board | Kanban with one column per admin-defined status, drag-to-change-status, team and "my tickets" filters |
| **M5** | Dashboard | KPI tiles, workload charts, at-risk list, filters, search, CSV export |
| **M6** | Hardening & handover | E2E tests, backups, deploy to the client server, short user guide in Indonesian, UAT with the client |

Each milestone ends with something we can demo to the client.

---

## 11. Questions for the client

1. **SLA numbers:** are the proposed targets (4 h / 8 h / 3 business days / 5 business days) right? Is SLA measured on **resolution only**, or also on **first response**?
2. **Business hours:** do P3/P4 count only working hours (e.g. Mon–Fri 08:00–17:00 WIB)? Do P1/P2 run 24/7?
3. **Menunggu:** does the SLA clock pause while a ticket is waiting? And which extra statuses do teams want on top of the four in the brief?
4. **Teams:** what are the team names, and who leads each one?
5. **Login:** does the company use Microsoft 365 or Google Workspace (for SSO), or is email/password fine?
6. **Hosting:** on the company's own server, or a cloud VPS (which provider/region)?
7. **Who can open tickets:** only engineers, or also other staff?
8. **Language:** Indonesian only, or should English be switchable?
9. **Existing data:** do any current cases (spreadsheets, chat exports) need importing?
10. **Logo:** is there a higher-resolution or vector (SVG/AI) version of the logo?

---

## 12. Later options (after v1)

Email/WhatsApp/Teams notifications for assignment and SLA warnings · file attachments · linking tickets (an incident linked to a Masalah Berulang) · SLA reports per month/team · email-to-ticket · approvals for Perubahan · realtime board updates.
