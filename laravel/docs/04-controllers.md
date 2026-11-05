# 4. Controllers, requests, policies and business logic

Controllers stay thin: authorize → validate (Form Request) → call an action → return a view or redirect. Business rules live in **actions** (`app/Actions`) and the **SLA calculator** (`app/Support/Sla`), so they can be tested without HTTP and reused by seeders and future jobs.

## 4.1 Controllers

| Controller | Methods | Notes |
|---|---|---|
| `Auth\LoginController` | `create`, `store`, `destroy` | see 03-authentication.md |
| `Auth\PasswordController` | `edit`, `update` | |
| `DashboardController` | `__invoke` | KPIs, workload per team (by priority), workload per engineer, "Perlu perhatian" list; optional `?team=` |
| `BoardController` | `__invoke(Request, ?Team)` | team board: columns = team's active statuses; no team: 4 category columns; `?mine=1` |
| `TicketController` | `index`, `create`, `store`, `show`, `update` | `show` returns a fragment for the drawer |
| `TicketStatusController` | `update` | drag-and-drop and the status picker both call this |
| `TicketNoteController` | `store` | |
| `TicketStepController` | `store`, `update`, `destroy` | `update` toggles `done` or renames |
| `TicketExportController` | `__invoke` | streams CSV (`response()->streamDownload`, BOM for Excel), same filters as the list |
| `Admin\SlaPolicyController` | `edit`, `update` | |
| `Admin\HolidayController` | `store`, `destroy` | |
| `Admin\TeamStatusController` | `edit`, `update` | |
| `Admin\StepTemplateController` | `edit`, `update` | |
| `Admin\TeamController` | `index`, `store`, `update` | new teams get the 4 default statuses |
| `Admin\UserController` | `index`, `store`, `update` | create user (temporary password), change role/teams, reset password, (de)activate |

Read pages build their data with query objects/scopes, not in Blade:
- `Ticket::query()->filter($filters)->with(['team', 'status', 'assignee'])->withCount([...steps...])`
- Dashboard aggregates use grouped queries (`selectRaw count … group by team_id, priority`), not one query per team.

## 4.2 Form Requests (`app/Http/Requests`)

| Request | Rules (summary) |
|---|---|
| `LoginRequest` | `login`, `password` required; rate limiting (03) |
| `UpdatePasswordRequest` | `current_password`, `password` confirmed + `Password::min(10)->letters()->numbers()` |
| `TicketFilterRequest` | all filter params optional, invalid values ignored (`prepareForValidation` + `safe()`) |
| `StoreTicketRequest` | `title` required ≤200; `type` `Rule::enum(TicketType::class)`; `team_id` exists & active; `priority` enum; `assignee_id` nullable, must be a member of `team_id`; `description` required ≤10 000 |
| `UpdateTicketRequest` | same fields, all `sometimes`; `assignee_id` checked against the (new) team; `assign` ability checked in `authorize()` |
| `UpdateTicketStatusRequest` | `status_id` must be an active status **of the ticket's team**; `reason` required when target category is Paused or when reopening; `resolution` required when target is Done; `complete_remaining_steps` boolean |
| `StoreTicketNoteRequest` | `body` required ≤5 000 |
| `StoreTicketStepRequest` / `UpdateTicketStepRequest` | `text` ≤200; `done` boolean |
| `UpdateTeamStatusesRequest` | `statuses.*.{id?, label, color, category}` in order; `moves.{deletedId} => targetId`; after-rules: ≥1 Open and ≥1 Done category, labels unique (case-insensitive), every deleted status that still has tickets has a target in the same team |
| `UpdateSlaPoliciesRequest`, `UpdateStepTemplatesRequest`, `StoreHolidayRequest`, `StoreTeamRequest`/`UpdateTeamRequest`, `StoreUserRequest`/`UpdateUserRequest` | straightforward; users: unique username/email, role enum, teams exist |

Controllers only use `$request->validated()` / `safe()->only(...)`.

## 4.3 Policies

`TicketPolicy` (rules in 03-authentication.md §3.5). Admin screens use the `admin` gate. Every write controller method calls `$this->authorize(...)` or the request's `authorize()`; a test proves one refused role per endpoint.

## 4.4 Actions (`app/Actions`, decision D5)

Each action is one class with a `handle()` method, wraps its work in `DB::transaction()`, and writes the `ticket_events` row(s) in the same transaction.

| Action | What it does |
|---|---|
| `CreateTicket` | locks `ticket_counters` row → number; status = team's default; `sla_due_at` from `SlaCalculator`; copies `step_templates` for the type; events `Created` (+ `Assigned`) |
| `UpdateTicketFields` | applies changed fields; one `FieldChanged`/`Assigned` event per field; priority change → recompute `sla_due_at` from `created_at`; team change → delegates to `MoveTicketToTeam` |
| `MoveTicketToTeam` | new status = first status of the **same category** in the new team, else the new team's default; clears assignee if not a member; adjusts SLA pause data; events |
| `ChangeTicketStatus` | checks the status belongs to the team; reopen rules; pause/resume (`sla_paused_at`, `sla_paused_seconds`); Done → `resolved_at`, `resolution_note`, optionally ticks remaining steps; auto-assigns the actor when leaving Open with no assignee; `StatusChanged` event with reason |
| `AddTicketNote` | `Note` event |
| `AddTicketStep`, `ToggleTicketStep`, `RenameTicketStep`, `RemoveTicketStep` | step changes + events |
| `SaveTeamStatuses` | upserts the ordered list; archives removed statuses; moves their tickets to the chosen targets (with `StatusChanged` events by the admin); re-applies SLA pause data where a ticket's category changed |
| `CreateUser`, `ResetUserPassword` | temporary password, `must_change_password = true` |

## 4.5 SLA calculator (`app/Support/Sla`)

Pure PHP + Carbon, no database calls (holidays and policies are passed in), so it is unit-testable.

```php
final class SlaCalculator
{
    public function __construct(
        private BusinessCalendar $calendar,   // working hours + holiday dates
        private array $policies,              // Priority value => ['amount' => int, 'unit' => 'hours'|'business_days']
        private float $atRiskRatio = 0.75,
    ) {}

    public function dueAt(Priority $priority, CarbonImmutable $createdAt): CarbonImmutable;
    public function snapshot(Ticket $ticket, CarbonImmutable $now): SlaSnapshot; // state, label, due, percentUsed
}
```

- `hours` (P1, P2): calendar time, 24/7.
- `business_days` (P3, P4): counted in working hours (Mon–Fri 08:00–17:00 WIB, skipping `holidays`), so "3 hari kerja" = 27 working hours.
- Pause: effective deadline = `sla_due_at + sla_paused_seconds (+ now − sla_paused_at while paused)`. (v1 extends by wall-clock pause time for all priorities; documented as a known simplification.)
- `SlaSnapshot` returns `SlaState` (`OnTrack`, `AtRisk` ≥ 75% used, `Breached`, `Paused`, `Met`, `Missed`), the Indonesian label ("Sisa 2j 45m", "Lewat 1 hari 3j", "Dijeda", "Tercapai") and the due time in WIB.
- Bound as a singleton in `AppServiceProvider`, built from `sla_policies`, `holidays` and `config('ticketing')` (cached per request).

## 4.6 Done when

- Every route in 02-routes.md hits a controller method that authorizes, validates and delegates to an action.
- Creating, editing, moving, status changes, notes and steps all write history events.
- The board, list and dashboard pages run a fixed number of queries regardless of ticket count (checked in a test with `DB::enableQueryLog`).
