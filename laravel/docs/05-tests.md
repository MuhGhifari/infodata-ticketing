# 5. Tests (Pest 5)

Follows the Boost `testing-best-practices` skill: feature tests first, unit tests only for logic that doesn't need the framework, policy rules tested as a full matrix, and one HTTP test per endpoint proving the authorization is actually called.

## 5.1 Setup

- Enable `RefreshDatabase` for feature tests in `tests/Pest.php`:
  `pest()->extend(TestCase::class)->use(RefreshDatabase::class)->in('Feature');`
- Tests run on SQLite `:memory:` (already in `phpunit.xml`).
- Freeze time in any SLA-related test with `$this->travelTo(...)` / `CarbonImmutable::setTestNow()`.
- Create test files with `php artisan make:test --pest {Name}` (add `--unit` for unit tests). Test files mirror the class path (`app/Actions/CreateTicket.php` → `tests/Feature/Actions/CreateTicketTest.php`).
- Names: `it('…')` for behavior, `test('…')` for declarative facts; datasets when only the input differs.
- Run: `php artisan test --compact` (narrow with a path or `--filter`). Pint after PHP changes.
- Remove the two example tests only with your approval (Boost rule: don't delete tests without approval).

## 5.2 Unit tests (`tests/Unit`)

| File | Covers |
|---|---|
| `Support/Sla/SlaCalculatorTest.php` | P1/P2 due = created + hours (24/7, across midnight/weekend); P3/P4 count working hours only; skips weekends and holidays; created outside working hours starts at next opening; snapshot states at 0%, 74%, 75% (at risk), 100%+ (breached); paused state and extended deadline; Met vs Missed after resolution; label formatting (dataset) |
| `Support/Sla/BusinessCalendarTest.php` | working-hour addition edge cases (Friday 16:00 + 2h → Monday 09:00; holiday on Monday) |
| `Enums/*Test.php` | labels and category flags (`pausesSla`, `requiresReason`) as `test()` facts |

## 5.3 Feature tests (`tests/Feature`)

### Authentication
| File | Cases |
|---|---|
| `Auth/LoginTest.php` | page renders for guests; logged-in users are redirected; login with username; login with email (case-insensitive); wrong password → generic error, no session; inactive user refused; 6th attempt within a minute is throttled; "Ingat saya" sets the remember cookie; session id regenerated; `last_login_at` set |
| `Auth/LogoutTest.php` | logout invalidates session and redirects to login; guests can't POST logout |
| `Auth/PasswordChangeTest.php` | `must_change_password` user is redirected from every page to `/password`; wrong current password rejected; weak password rejected (dataset); success clears the flag and other sessions |
| `Auth/DeactivatedUserTest.php` | user deactivated mid-session is logged out on next request |
| `Database/AdminSeederTest.php` | creates Muamar Azzriel Fattah as Admin with `must_change_password`; uses `ADMIN_PASSWORD` when set; running twice doesn't change an existing password |

### Pages and tickets
| File | Cases |
|---|---|
| `Http/DashboardControllerTest.php` | guests redirected; KPI numbers match seeded tickets (active, breached, at risk, closed in 7 days, on-time %); team filter; fixed query count |
| `Http/BoardControllerTest.php` | team board shows that team's statuses as columns in order; all-teams board shows 4 category columns; `mine=1`; filters; unknown team → 404 |
| `Http/TicketControllerTest.php` | index: each filter (dataset), search, sort, closed/active; create page; **store**: validation errors (dataset), number format and increment per type, default status of the team, steps copied from template, SLA due set, events written, assignee must be a team member; **show**: full page and `X-Drawer` fragment; **update**: field changes create events, priority change recomputes SLA, engineer can't edit another team's ticket (403) |
| `Http/TicketStatusControllerTest.php` | move to active status; Paused needs `reason` (422 JSON / redirect with errors) and pauses SLA; leaving Paused adds paused time; Done needs `resolution`, sets `resolved_at`, optionally ticks remaining steps; reopen by engineer → 403, by lead → OK with reason; status of another team → 422; auto-assign when leaving Open; JSON response for drag-and-drop |
| `Http/TicketNoteControllerTest.php` | adds note event; empty body rejected; unauthorized user refused |
| `Http/TicketStepControllerTest.php` | add, toggle, rename, delete; step of another ticket → 404 (scoped binding); events written |
| `Http/TicketExportControllerTest.php` | CSV headers and rows respect filters; filename; BOM |
| `Actions/MoveTicketToTeamTest.php` | status mapped to same category in new team; fallback to default; assignee cleared when not a member; pause data adjusted |
| `Actions/SaveTeamStatusesTest.php` | add/rename/reorder; delete with tickets requires a target; tickets moved with events; category change re-applies SLA pause data; other teams untouched |

### Admin
| File | Cases |
|---|---|
| `Http/Admin/SlaPolicyControllerTest.php` | admin updates targets; lead → 403 |
| `Http/Admin/TeamStatusControllerTest.php` | validation: needs Open and Done categories, unique labels, move targets; success; lead → 403 |
| `Http/Admin/StepTemplateControllerTest.php`, `HolidayControllerTest.php`, `TeamControllerTest.php`, `UserControllerTest.php` | happy path, validation, non-admin → 403; new team gets default statuses; new/reset user gets `must_change_password` |

### Authorization matrix
| File | Cases |
|---|---|
| `Policies/TicketPolicyTest.php` | dataset of role × relationship (own team, other team, assigned, lead of team) × ability (`update`, `changeStatus`, `reopen`, `assign`, `manageSteps`, `addNote`) with the expected allow/deny from 03-authentication.md §3.5 |

### Architecture tests (optional, protect conventions)
`tests/Feature/ArchTest.php`: models extend `Model`; enums are backed; controllers don't use `DB::` directly; no `dd`/`dump` left in `app/`.

## 5.4 Not covered by automated tests (manual checklist)

Pest browser tests need `pestphp/pest-plugin-browser` (not installed; ask before adding). Until then, before each release, check by hand at 1440 px and on an iPhone: drag-and-drop on the board, the searchable picker (bottom sheet on mobile), the drawer and back button, the mobile filter panel, and no iOS zoom on inputs.

## 5.5 Done when

- `php artisan test --compact` passes with no skipped tests.
- Every route has an auth test, a validation test (if it accepts input), one refused-role test and a success test that asserts the stored data.
- The SLA calculator has unit tests for every rule in 04-controllers.md §4.5.
