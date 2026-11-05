# 2. Routes

All routes are web routes in `routes/web.php` (session + CSRF). There is no public API in v1; the few JavaScript calls (drag-and-drop, checklist, drawer) hit the same web routes and get JSON or an HTML fragment back when they ask for it.

## 2.1 Middleware groups

| Alias / group | Meaning |
|---|---|
| `guest` | Only when logged out (login page) |
| `auth` | Logged in and session valid |
| `active` (new, `EnsureUserIsActive`) | Logs out users an admin has deactivated |
| `password.changed` (new, `EnsurePasswordIsChanged`) | Redirects to `password.edit` while `must_change_password` is true |
| `can:admin` | Gate `admin` → `$user->role === Role::Admin` |
| `throttle:login` | Rate limiter defined in `AppServiceProvider` (5 attempts / minute per username + IP) |

Middleware aliases are registered in `bootstrap/app.php` (Laravel 13 has no `Kernel.php`).

## 2.2 Route table

| Method | URI | Name | Controller@action | Middleware |
|---|---|---|---|---|
| GET | `/login` | `login` | `Auth\LoginController@create` | guest |
| POST | `/login` | `login.store` | `Auth\LoginController@store` | guest, throttle:login |
| POST | `/logout` | `logout` | `Auth\LoginController@destroy` | auth |
| GET | `/password` | `password.edit` | `Auth\PasswordController@edit` | auth, active |
| PUT | `/password` | `password.update` | `Auth\PasswordController@update` | auth, active |
| | | | *everything below: `auth`, `active`, `password.changed`* | |
| GET | `/` | `home` | redirect → `dashboard` | |
| GET | `/dashboard` | `dashboard` | `DashboardController` (invokable) | |
| GET | `/board/{team:code?}` | `board` | `BoardController` (invokable) | no team = all-teams board grouped by category; `?mine=1` = "Tiket Saya" |
| GET | `/tickets/export` | `tickets.export` | `TicketExportController` (invokable) | declared **before** the resource |
| GET | `/tickets` | `tickets.index` | `TicketController@index` | |
| GET | `/tickets/create` | `tickets.create` | `TicketController@create` | |
| POST | `/tickets` | `tickets.store` | `TicketController@store` | |
| GET | `/tickets/{ticket:number}` | `tickets.show` | `TicketController@show` | `X-Drawer: 1` → fragment |
| PATCH | `/tickets/{ticket:number}` | `tickets.update` | `TicketController@update` | priority, assignee, team, type, title, description |
| PATCH | `/tickets/{ticket:number}/status` | `tickets.status.update` | `TicketStatusController@update` | |
| POST | `/tickets/{ticket:number}/notes` | `tickets.notes.store` | `TicketNoteController@store` | |
| POST | `/tickets/{ticket:number}/steps` | `tickets.steps.store` | `TicketStepController@store` | |
| PATCH | `/tickets/{ticket:number}/steps/{step}` | `tickets.steps.update` | `TicketStepController@update` | toggle or rename; `scopeBindings()` |
| DELETE | `/tickets/{ticket:number}/steps/{step}` | `tickets.steps.destroy` | `TicketStepController@destroy` | `scopeBindings()` |
| | | | *admin: prefix `/admin`, name `admin.`, `can:admin`* | |
| GET | `/admin` | `admin.index` | redirect → `admin.sla.edit` | |
| GET / PUT | `/admin/sla` | `admin.sla.edit` / `.update` | `Admin\SlaPolicyController` | |
| POST / DELETE | `/admin/holidays`, `/admin/holidays/{holiday}` | `admin.holidays.store` / `.destroy` | `Admin\HolidayController` | |
| GET / PUT | `/admin/teams/{team:code}/statuses` | `admin.teams.statuses.edit` / `.update` | `Admin\TeamStatusController` | whole ordered list in one request |
| GET / PUT | `/admin/step-templates` | `admin.step-templates.edit` / `.update` | `Admin\StepTemplateController` | |
| resource (index, store, update) | `/admin/teams` | `admin.teams.*` | `Admin\TeamController` | |
| resource (index, store, update) | `/admin/users` | `admin.users.*` | `Admin\UserController` | includes reset password + activate/deactivate |

Ticket URLs use the ticket number (`/tickets/INC-001182`) instead of the database id: readable, and it's what people will paste in chat. Board URLs use the team code (`/board/NET`).

## 2.3 Query-string filters

Board, list, export and dashboard share one filter object, `App\Support\TicketFilters`, built from a `TicketFilterRequest`:

| Param | Values | Used on |
|---|---|---|
| `q` | text | all |
| `team` | team code | list, export, dashboard |
| `type` | `INC` `SR` `CHG` `PRB` | all |
| `priority` | `P1`–`P4` | all |
| `status` | `active`, `closed`, `cat:open|active|paused|done`, or a status id of the selected team | list, export |
| `assignee` | user id or `none` | all |
| `sla` | `breached`, `at_risk`, `on_track` | all |
| `mine` | `1` | all |
| `sort`, `dir` | `number`, `team`, `priority`, `status`, `progress`, `assignee`, `due` / `asc`, `desc` | list |

Invalid values are dropped rather than causing an error page, so an old bookmarked URL never breaks.

## 2.4 Response conventions

- Form submissions → redirect (`back()` or to the resource) with a flash `status` message shown as a toast. Validation errors → back with errors, shown under the fields.
- Requests with `Accept: application/json` (drag-and-drop, checklist) → JSON `{ ok, message, ticket: {...} }`, or `422` with errors (for example `reason` missing), which `status-dialog.js` uses to open the reason dialog.
- Unauthorized → `403` page in Indonesian; missing ticket → `404`.

## 2.5 Done when

`php artisan route:list --except-vendor` matches the table above, and every route has at least one feature test (see 05-tests.md).
