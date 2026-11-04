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

A user belongs to one or more teams. Permissions are enforced on the server; the UI only hides controls the user can't use.

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
| **Penanggung jawab** (assignee) | One user from the ticket's team; may be empty while the status is Terbuka |
| **Pelapor** (reporter) | The user who opened it (automatic) |
| **Status** | `Terbuka` (Open) → `Dikerjakan` (In Progress) ⇄ `Menunggu` (Waiting) → `Selesai` (Done) |
| **Batas SLA** (SLA deadline) | **Calculated automatically** from priority when the ticket is created; recalculated if priority changes |
| **Riwayat** (history) | Work notes plus every field and status change, with who made it and when |

### Status flow

```
Terbuka ──► Dikerjakan ──► Selesai
               ▲   │          │
               │   ▼          │ (reopen, Lead/Admin only,
             Menunggu         │  reason required)
               ▲──────────────┘
```

- Moving to **Menunggu** requires a reason (e.g. waiting on the customer, vendor or a change window).
- Moving to **Selesai** requires a resolution note.
- Reopening moves the ticket back to Dikerjakan and is recorded in the history.

### SLA rules (proposed defaults; admin can edit them)

| Priority | Target resolution time | Example |
|---|---|---|
| Kritis (P1) | 4 hours | Production service down |
| Tinggi (P2) | 8 hours | Major function degraded |
| Sedang (P3) | 3 business days | Normal request |
| Rendah (P4) | 5 business days | Minor or cosmetic |

- Times are in **WIB (Asia/Jakarta)**.
- The SLA clock **pauses while the ticket is in Menunggu**, and the deadline moves forward by the paused time. *(Needs client confirmation.)*
- SLA states shown everywhere: **Aman** (on track), **Hampir lewat** (≥ 75% of the time used), **Lewat SLA** (breached).
- SLA rules are stored in a table, not hard-coded, so the client can tune them without code changes.

---

## 4. Screens

All UI text is in **Bahasa Indonesia**. Branding comes from the client logo (`assets/brand/infodata-logo.jpg`): red **#E40114**, near-black **#231917**, white.

1. **Login**
2. **Dashboard** (home page)
   - KPI tiles: **Tiket aktif** (active tickets), **Lewat SLA** (breached), **Hampir lewat SLA** (at risk), **Selesai minggu ini** (done this week)
   - **Beban kerja per tim** (workload per team): active tickets per team, stacked by priority
   - Workload per engineer within a team (for leads)
   - Lists: tickets about to breach, recently opened
   - Filters: team, type, priority, status, assignee, date range, plus search
3. **Papan** (board, kanban): 4 fixed columns (Terbuka / Dikerjakan / Menunggu / Selesai). Drag a card to change its status; the dialog asks for a reason or resolution note when needed. Cards show the number, title, type, priority, assignee and an SLA countdown. Can be filtered per team, or "Tiket saya" (my tickets).
4. **Daftar tiket** (ticket list): a sortable table with the same filters and search, plus CSV export.
5. **Buat tiket** (new ticket form).
6. **Detail tiket**: fields, SLA bar, and a timeline of work notes and changes, with a box to add a work note.
7. **Admin**: users and roles, teams and members, SLA policy.

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
- **teams** — id, name, code, active
- **team_members** — team_id, user_id, is_lead
- **tickets** — id, number (`INC-000123`), type, title, description, team_id, priority, status, assignee_id?, reporter_id, sla_due_at, sla_paused_at?, sla_paused_total_minutes, first_response_at?, resolved_at?, resolution_note?, created_at, updated_at, search_vector (tsvector)
- **ticket_events** — id, ticket_id, actor_id, kind (`created` | `note` | `status_changed` | `assigned` | `priority_changed` | `team_changed` | `field_changed`), body (for notes/reasons), payload jsonb (`{from, to}`), created_at
- **sla_policies** — priority, resolution_minutes, business_hours_only (bool)
- **holidays** — date, name (Indonesian public holidays and cuti bersama)
- **ticket_counters** — type, next_number (incremented inside a transaction so numbers never repeat)

Key indexes: `(status, team_id)`, `(assignee_id, status)`, `(sla_due_at) WHERE status <> 'Selesai'`, plus a GIN index on `search_vector`.

Every change to a ticket writes a `ticket_events` row in **the same database transaction**, so the history is always complete.

---

## 7. API (REST)

```
POST   /auth/login | /auth/logout      GET /auth/me

GET    /tickets?q=&team=&type=&priority=&status=&assignee=&sla=breached|at_risk&from=&to=&sort=&page=
POST   /tickets
GET    /tickets/:id                    # including events
PATCH  /tickets/:id                    # title, description, type, team, priority, assignee
POST   /tickets/:id/status             # { status, reason? | resolution? }
POST   /tickets/:id/notes              # { body }
GET    /tickets/export.csv?…same filters

GET    /dashboard/summary?team=&from=&to=   # KPI tiles
GET    /dashboard/workload?…                # per team / per engineer

GET|POST|PATCH  /admin/users, /admin/teams, /admin/teams/:id/members
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
| **M1** | Auth & admin | Login, roles, user and team management, seed data |
| **M2** | Tickets core | Create, list, detail and edit tickets; numbering; work notes; history timeline |
| **M3** | Status & SLA | Status flow with reasons; SLA calculation (business hours, holidays, pause in Menunggu) with full unit tests; SLA badges |
| **M4** | Board | Kanban with 4 status columns, drag-to-change-status, team and "my tickets" filters |
| **M5** | Dashboard | KPI tiles, workload charts, at-risk list, filters, search, CSV export |
| **M6** | Hardening & handover | E2E tests, backups, deploy to the client server, short user guide in Indonesian, UAT with the client |

Each milestone ends with something we can demo to the client.

---

## 11. Questions for the client

1. **SLA numbers:** are the proposed targets (4 h / 8 h / 3 business days / 5 business days) right? Is SLA measured on **resolution only**, or also on **first response**?
2. **Business hours:** do P3/P4 count only working hours (e.g. Mon–Fri 08:00–17:00 WIB)? Do P1/P2 run 24/7?
3. **Menunggu:** does the SLA clock pause while a ticket is waiting?
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
