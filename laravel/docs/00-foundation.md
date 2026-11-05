# 0. Foundation: environment, enums, schema, models, seed data

Everything here is built first, because views, routes, controllers and tests all depend on it.

## 0.1 Environment

| Setting | Value | Where |
|---|---|---|
| `APP_NAME` | `"PT. Info Solusindo Data Utama"` | `.env` / `.env.example` |
| `APP_TIMEZONE` | `Asia/Jakarta` (`config/app.php` reads `env('APP_TIMEZONE', 'UTC')`) | `.env`, `config/app.php` |
| `APP_LOCALE` / `APP_FAKER_LOCALE` | `id` / `id_ID` | `.env` |
| `DB_CONNECTION` | `sqlite` (dev), `mysql` (production) | `.env` |
| `SESSION_DRIVER` | `database` (already default) | `.env` |
| `ADMIN_USERNAME` / `ADMIN_EMAIL` / `ADMIN_PASSWORD` | Used once by the admin seeder (see 03-authentication.md) | `.env` only, never committed |

New config file **`config/ticketing.php`** keeps business settings out of code:

```php
return [
    'business_hours' => ['start' => '08:00', 'end' => '17:00', 'days' => [1, 2, 3, 4, 5]], // ISO weekdays
    'at_risk_ratio'  => 0.75,          // "Hampir lewat SLA" threshold
    'pause_sla_in_waiting' => true,
];
```

Fix the missing `.env.example` (decision D7) in the first commit.

## 0.2 Enums (`app/Enums`, backed string enums)

| Enum | Cases (value) | Extra methods |
|---|---|---|
| `Role` | `Admin` (`admin`), `Lead` (`lead`), `Engineer` (`engineer`) | `label(): string` → "Admin", "Lead Tim", "Engineer" |
| `StatusCategory` | `Open` (`open`), `Active` (`active`), `Paused` (`paused`), `Done` (`done`) | `label()`, `defaultColor()`, `pausesSla(): bool`, `requiresReason(): bool` |
| `TicketType` | `Incident` (`INC`), `ServiceRequest` (`SR`), `Change` (`CHG`), `Problem` (`PRB`) | `label()` → "Insiden", "Permintaan Layanan", "Perubahan", "Masalah Berulang"; `hint()` |
| `Priority` | `Critical` (`P1`), `High` (`P2`), `Medium` (`P3`), `Low` (`P4`) | `label()` → "Kritis", "Tinggi", "Sedang", "Rendah"; `color()` |
| `TicketEventKind` | `Created`, `Note`, `StatusChanged`, `Assigned`, `FieldChanged`, `StepDone`, `StepUndone`, `StepAdded`, `StepRemoved`, `StepRenamed` | — |
| `SlaState` | `OnTrack`, `AtRisk`, `Breached`, `Paused`, `Met`, `Missed` | `label()`, `badgeClass()` |

Using a Viewer role is postponed (PLAN.md lists it as optional).

## 0.3 Database schema

Migrations in this order (`php artisan make:migration`). All foreign keys constrained; all timestamps UTC.

### users (modify the existing migration's table with a new migration)
| Column | Type | Notes |
|---|---|---|
| username | string, unique | Login name, lowercase |
| role | string (enum `Role`) | default `engineer` |
| is_active | boolean, default true | Inactive users cannot log in |
| must_change_password | boolean, default false | Forces the password page after login |
| last_login_at | timestamp, nullable | |

### teams
`id, name (unique), code (unique, e.g. NET), color (hex), is_active, timestamps`

### team_user (pivot)
`team_id, user_id, is_lead (bool)`, unique `(team_id, user_id)`. A user can belong to several teams (PLAN.md §2). The mockup shows one team per user; the UI uses the first team as "Tim saya".

### statuses (per team)
| Column | Type | Notes |
|---|---|---|
| team_id | FK teams, cascade | |
| label | string | unique per team among non-archived rows (validated in the request) |
| color | string(7) | hex |
| category | string (enum `StatusCategory`) | rules come from this |
| position | unsignedSmallInteger | column order |
| archived_at | timestamp, nullable | deleted statuses are archived so history still shows their name |

Index `(team_id, archived_at, position)`.

### sla_policies
`priority (enum Priority, unique), amount (unsigned int), unit ('hours'|'business_days'), timestamps`. Seeded: P1 4 hours, P2 8 hours, P3 3 business_days, P4 5 business_days.

### holidays
`date (unique), name`.

### step_templates
`ticket_type (enum TicketType), position, text`. Index `(ticket_type, position)`.

### ticket_counters
`ticket_type (primary), next_number`. Incremented with `lockForUpdate()` inside the ticket-creation transaction so numbers never repeat.

### tickets
| Column | Type | Notes |
|---|---|---|
| number | string, unique | `INC-000123` |
| type | string (enum `TicketType`) | |
| title | string(200) | |
| description | text | shown escaped, line breaks kept |
| team_id | FK teams | |
| status_id | FK statuses | must belong to `team_id` (enforced in actions + tests) |
| priority | string (enum `Priority`) | |
| assignee_id | FK users, nullable, nullOnDelete | |
| reporter_id | FK users | |
| sla_due_at | timestamp | deadline without pauses |
| sla_paused_at | timestamp, nullable | set while in a Paused-category status |
| sla_paused_seconds | unsigned int, default 0 | total time paused so far |
| resolved_at | timestamp, nullable | set when entering a Done-category status |
| resolution_note | text, nullable | |
| timestamps | | |

Indexes: `(team_id, status_id)`, `(assignee_id, resolved_at)`, `(resolved_at, sla_due_at)`, `(priority)`. Search uses `LIKE` on number/title/description (portable; good enough for tens of thousands of rows). MySQL `FULLTEXT` can be added later behind the same scope.

### ticket_steps
`ticket_id (cascade), position, text, done_at (nullable), done_by_id (nullable FK users)`.

### ticket_events
`ticket_id (cascade), actor_id (FK users), kind (enum TicketEventKind), body (text, nullable: note text, reason or resolution), payload (json, nullable: {from, to, step}), created_at`. No `updated_at`; events are never edited.

## 0.4 Models

| Model | Relationships | Casts / notes |
|---|---|---|
| `User` | `teams()` belongsToMany (pivot `is_lead`), `assignedTickets()` | `role` → `Role`; `isAdmin()`, `isLeadOf(Team)`, `belongsToTeam(Team)`; `$fillable` limited |
| `Team` | `members()`, `leads()`, `statuses()` (ordered by `position`, active only), `tickets()` | `defaultStatus()` = first `Open`-category status; `firstStatusIn(StatusCategory)` |
| `Status` | `team()`, `tickets()` | `category` → `StatusCategory`; scope `active()` |
| `Ticket` | `team()`, `status()`, `assignee()`, `reporter()`, `steps()`, `events()` (latest first) | casts for enums and dates; scopes `open()`, `closed()`, `filter(TicketFilters)`; accessors `sla` (returns `SlaSnapshot`), `progress` (done/total) |
| `TicketStep` | `ticket()`, `doneBy()` | `done_at` datetime |
| `TicketEvent` | `ticket()`, `actor()` | `kind` enum, `payload` array; `const UPDATED_AT = null` |
| `SlaPolicy`, `Holiday`, `StepTemplate`, `TicketCounter` | — | small lookup models |

All list pages eager-load `team`, `status`, `assignee` and step counts (`withCount`) to avoid N+1 queries. Database access stays out of Blade views.

## 0.5 Factories and seeders

**Factories** (one per model) with states the tests need:
- `UserFactory`: `admin()`, `lead(Team $team)`, `engineer(Team $team)`, `inactive()`, `mustChangePassword()`
- `TeamFactory`: `withDefaultStatuses()` creates Terbuka/Dikerjakan/Menunggu/Selesai
- `TicketFactory`: `forTeam()`, `inStatusCategory(StatusCategory)`, `paused()`, `resolved()`, `breached()`, `withSteps(int $total, int $done)`

**Seeders** (`DatabaseSeeder` calls them in order):

| Seeder | Production | Demo/dev | Content |
|---|---|---|---|
| `SlaPolicySeeder` | ✅ | ✅ | the 4 SLA rows |
| `StepTemplateSeeder` | ✅ | ✅ | default steps per type (from the mockup) |
| `AdminSeeder` | ✅ | ✅ | default Admin **Muamar Azzriel Fattah** (see 03-authentication.md) |
| `HolidaySeeder` | ✅ | ✅ | known public holidays for the current and next year |
| `DemoSeeder` | ❌ | ✅ | the mockup's 6 teams with their status flows, 15 users, ~40 tickets with history. Only runs when `app()->environment('local')` or with `--class=DemoSeeder`. |

The demo users keep the mockup logins (`rizky / demo123`, `fajar / demo123`) so the client can click through roles on a demo server.
