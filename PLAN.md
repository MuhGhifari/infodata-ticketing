# Kanban Ticketing App — Plan

A multi-team SaaS app for tracking work as tickets on kanban boards.

**Status:** draft for review · **Last updated:** 2026-11-02

---

## 1. Goals

- Teams organize work into **workspaces → projects/boards → columns → tickets**.
- Drag-and-drop tickets between columns and reorder them, with changes showing up live for everyone viewing the board.
- Rich tickets: description, priority, labels, due date, assignee, comments, activity history.
- Search and filter across a board.
- Secure multi-tenancy: a user only ever sees workspaces they belong to.

### Not in v1 (later)
Billing/subscriptions, SSO/SAML, swimlanes, sprints, WIP limits, custom fields, automations, file attachments, email notifications, mobile apps, public API.

---

## 2. Tech stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | **React 19 + TypeScript + Vite** | What you picked; fast dev loop |
| Routing | TanStack Router (or React Router) | Type-safe routes |
| Server state | **TanStack Query** | Caching, optimistic updates for drag-and-drop |
| UI | Tailwind CSS + shadcn/ui (Radix) | Accessible primitives, quick to style |
| Drag-and-drop | **dnd-kit** | Maintained, accessible, supports sortable lists |
| Forms/validation | React Hook Form + **Zod** | Zod schemas shared with the backend |
| Backend | **Node + Fastify + TypeScript** | Fast, typed, same language as the frontend |
| API style | REST + JSON (typed with shared Zod schemas) | Simple; tRPC is an alternative |
| Realtime | WebSockets (Fastify websocket) + Postgres LISTEN/NOTIFY or Redis pub/sub | Live board updates |
| Database | **PostgreSQL** | Relational data, full-text search, row-level isolation |
| ORM | **Prisma** (or Drizzle) | Migrations + typed queries |
| Auth | Email/password + OAuth (Google/GitHub) via **Better Auth** or Lucia-style sessions | Cookie sessions, no 3rd-party lock-in |
| Monorepo | **pnpm workspaces** (+ Turborepo) | `apps/web`, `apps/api`, `packages/shared` |
| Testing | Vitest, Testing Library, Playwright (E2E) | |
| Infra | Docker Compose locally; deploy web to a static host, API + Postgres to Fly.io/Render/Railway | |

---

## 3. Repo layout

```
kanban/
├── apps/
│   ├── web/            # React + Vite frontend
│   └── api/            # Fastify backend
├── packages/
│   └── shared/         # Zod schemas, types, constants (roles, priorities)
├── docker-compose.yml  # Postgres (+ Redis later)
├── PLAN.md
└── package.json        # pnpm workspace root
```

---

## 4. Domain model

```
User ──< Membership >── Workspace ──< Project(Board) ──< Column ──< Ticket
                              │                                  │
                              └──< Label                         ├──< Comment
                                                                 ├──< Activity
                                                                 └──< TicketLabel >── Label
```

### Tables (key fields)

- **users** — id, email (unique), name, avatar_url, password_hash?, created_at
- **workspaces** — id, name, slug (unique), created_by, created_at
- **memberships** — workspace_id, user_id, role (`owner` | `admin` | `member` | `viewer`), unique(workspace_id, user_id)
- **invites** — id, workspace_id, email, role, token, expires_at, accepted_at
- **projects** (boards) — id, workspace_id, name, key (e.g. `WEB`), description, archived_at, next_ticket_number
- **columns** — id, project_id, name, position, (category: `todo` | `in_progress` | `done` for reporting)
- **tickets** — id, workspace_id, project_id, column_id, number (unique per project → `WEB-42`), title, description (markdown), priority (`none`/`low`/`medium`/`high`/`urgent`), assignee_id?, reporter_id, due_date?, position, created_at, updated_at, archived_at, search_vector (tsvector)
- **labels** — id, workspace_id, name, color
- **ticket_labels** — ticket_id, label_id
- **comments** — id, ticket_id, author_id, body (markdown), created_at, edited_at, deleted_at
- **activities** — id, ticket_id, actor_id, type (`created`, `moved`, `assigned`, `priority_changed`, `commented`, …), payload (jsonb with from/to), created_at

### Key design decisions

- **Ordering:** use **fractional indexing** (string `position` keys, e.g. the `fractional-indexing` package) for columns and tickets. Moving a card updates one row instead of renumbering the whole column.
- **Tenancy:** every tenant-owned row carries `workspace_id` (denormalized onto tickets for fast filtering). Every API query is scoped through the user's membership. Later, add Postgres Row-Level Security as a second layer.
- **Ticket numbers:** take `next_ticket_number` from the project inside a transaction (`UPDATE … RETURNING`) so numbers never collide.
- **History:** write activity rows in the same transaction as the change they record.
- **Soft deletes:** use `archived_at` / `deleted_at` on tickets, projects and comments.

---

## 5. Roles & permissions

| Action | Owner | Admin | Member | Viewer |
|---|:-:|:-:|:-:|:-:|
| View boards/tickets | ✅ | ✅ | ✅ | ✅ |
| Create/edit/move tickets, comment | ✅ | ✅ | ✅ | ❌ |
| Manage columns, labels, projects | ✅ | ✅ | ❌ | ❌ |
| Invite/remove members, change roles | ✅ | ✅ | ❌ | ❌ |
| Delete workspace, transfer ownership | ✅ | ❌ | ❌ | ❌ |

Enforced on the server through one `authorize(user, workspaceId, action)` helper; the UI only hides controls.

---

## 6. API (REST, v1)

```
Auth
  POST   /auth/signup | /auth/login | /auth/logout
  GET    /auth/me

Workspaces
  GET    /workspaces                       # mine
  POST   /workspaces
  GET    /workspaces/:ws
  PATCH  /workspaces/:ws
  GET    /workspaces/:ws/members
  PATCH  /workspaces/:ws/members/:userId   # change role
  DELETE /workspaces/:ws/members/:userId
  POST   /workspaces/:ws/invites
  POST   /invites/:token/accept

Projects / boards
  GET    /workspaces/:ws/projects
  POST   /workspaces/:ws/projects
  GET    /projects/:id/board               # columns + tickets in one payload
  PATCH  /projects/:id
  DELETE /projects/:id                     # archive

Columns
  POST   /projects/:id/columns
  PATCH  /columns/:id                      # rename / reorder (position)
  DELETE /columns/:id                      # requires moving its tickets first

Tickets
  GET    /projects/:id/tickets?q=&label=&assignee=&priority=&due=
  POST   /projects/:id/tickets
  GET    /tickets/:id                      # incl. labels, comments, activity
  PATCH  /tickets/:id                      # fields
  POST   /tickets/:id/move                 # { columnId, beforeId?, afterId? }
  DELETE /tickets/:id                      # archive

Comments
  POST   /tickets/:id/comments
  PATCH  /comments/:id
  DELETE /comments/:id

Labels
  GET|POST /workspaces/:ws/labels
  PATCH|DELETE /labels/:id

Realtime
  WS     /ws  → subscribe { projectId } → events: ticket.created/updated/moved/archived,
                                                 column.*, comment.created
```

---

## 7. Frontend

### Routes
```
/login, /signup, /invite/:token
/w/:workspaceSlug                        → project list
/w/:workspaceSlug/p/:projectKey          → board
/w/:workspaceSlug/p/:projectKey/t/:num   → board with ticket drawer/modal open
/w/:workspaceSlug/settings               → members, labels
/w/:workspaceSlug/p/:projectKey/settings → columns
```

### Main screens
- **Workspace switcher** in the sidebar, with the list of projects.
- **Board:** horizontal columns, scrollable; cards show key, title, priority icon, labels, assignee avatar and due date. Inline "+ Add ticket" at the bottom of each column. Columns can be added, renamed and dragged to reorder.
- **Ticket detail** (side drawer, deep-linkable): title, markdown description, a side panel of fields (status/column, assignee, priority, labels, due date), and a tab that combines comments and activity in one timeline.
- **Filter bar:** text search, assignee, labels, priority, due (overdue / this week); filters stored in the URL query string so they can be shared.

### Drag-and-drop flow
1. dnd-kit `onDragEnd` → compute the new position key between the neighbouring cards.
2. Optimistically update the TanStack Query board cache.
3. `POST /tickets/:id/move`; on error, roll back and show a toast.
4. Other clients receive `ticket.moved` over WebSocket and patch their cache.

---

## 8. Search & filters

- Postgres full-text search: a generated `tsvector` column on `title + description` with a GIN index; also match exact ticket keys (`WEB-42`).
- Filters are combined with AND on indexed columns (`project_id, column_id`, `assignee_id`, `due_date`, plus the label join).
- Later: search across all projects in a workspace with ⌘K.

---

## 9. Security & non-functional

- HttpOnly, Secure, SameSite=Lax session cookies; CSRF protection on mutations.
- Hash passwords with Argon2id; rate-limit auth endpoints.
- Validate every request body with Zod; sanitize markdown on render (e.g. DOMPurify).
- Tenancy tests: an automated suite checking that a user in workspace A cannot read or write anything in workspace B.
- Performance targets: board load < 300 ms for 500 tickets; realtime updates delivered in < 1 s.
- Observability: structured logs (pino), error tracking (Sentry), and a `/health` endpoint.

---

## 10. Milestones

| # | Milestone | Deliverable |
|---|---|---|
| **M0** | Scaffold | pnpm monorepo, Vite app, Fastify API, Postgres in Docker, Prisma, lint/format, CI running tests |
| **M1** | Auth & workspaces | Signup/login, create workspace, workspace switcher, membership checks |
| **M2** | Boards core | Projects, columns, ticket CRUD, board view, drag-and-drop with fractional positions |
| **M3** | Ticket details | Drawer, description, priority, labels, assignee, due date, ticket keys |
| **M4** | Comments & history | Comment thread, activity log, combined timeline |
| **M5** | Search & filters | Full-text search, filter bar, filters in the URL |
| **M6** | Teams | Invites, roles, permissions matrix enforced, member management |
| **M7** | Realtime | WebSocket board subscriptions, live updates, presence (optional) |
| **M8** | Hardening & deploy | Tenancy test suite, E2E tests, rate limiting, Sentry, production deploy |

Each milestone ends with something you can demo and with its tests passing.

---

## 11. Open questions

1. **Auth:** roll our own sessions (Better Auth) or use a hosted provider (Clerk/Auth0)? Hosted is faster to build but costs money and adds lock-in.
2. **Hosting target:** Fly.io / Render / Railway / AWS?
3. **Ticket workflow:** are columns free-form per board, or do teams share a workflow template?
4. **Assignees:** one assignee per ticket (simpler) or several?
5. **Billing:** when do plans/limits come in? This affects whether `workspaces` needs a `plan` field now.
6. **Branding/name** for the app.
