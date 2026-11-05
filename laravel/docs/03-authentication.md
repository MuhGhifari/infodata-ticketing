# 3. Authentication and roles

Built on Laravel's built-in session authentication (`Auth::attempt`, the `web` guard, `remember_token`). No starter kit or extra package (decision D4). The app has no self-registration, no email verification and no 2FA in v1, so a starter kit would mostly add code to delete.

## 3.1 Login

**Page:** `GET /login` → `auth/login.blade.php` (the minimal card from the mockup: logo, company name, username/email, password with show/hide, "Ingat saya", help line, footer).

**Submit:** `POST /login` → `LoginController@store` with `LoginRequest`:

1. Validate `login` (required, string) and `password` (required, string).
2. Rate limit: key `mb_strtolower(login)|ip`, 5 attempts per minute (`RateLimiter` in `LoginRequest::ensureIsNotRateLimited()`). When exceeded: error "Terlalu banyak percobaan. Coba lagi dalam :seconds detik." and a `Lockout` event.
3. Find the field: contains `@` → `email`, otherwise `username` (case-insensitive).
4. `Auth::attempt([$field => $login, 'password' => $password, 'is_active' => true], $request->boolean('remember'))`.
5. On failure: hit the limiter; one generic error "Username atau password salah." (never says which part was wrong or whether the account exists).
6. On success: clear the limiter, `session()->regenerate()`, set `last_login_at`, redirect to `intended(route('dashboard'))`. If `must_change_password` is true, the `password.changed` middleware sends the user to the password page first.

**Logout:** `POST /logout` → `Auth::logout()`, `session()->invalidate()`, `session()->regenerateToken()`, redirect to `login`. The button in the sidebar is a small form (CSRF), not a link.

**Remember me:** checkbox "Ingat saya", ticked by default, uses Laravel's remember cookie.

**"Lupa password?":** text only, "Hubungi Admin untuk reset password." Admin resets it in Admin → Pengguna, which sets a temporary password and `must_change_password = true`. (Email reset can be added later; the office may not have mail configured.)

## 3.2 Default Admin

`AdminSeeder` runs in every environment, and is safe to run twice (`updateOrCreate` by username, never overwrites an existing password):

| Field | Value |
|---|---|
| name | **Muamar Azzriel Fattah** |
| username | `env('ADMIN_USERNAME', 'admin')` |
| email | `env('ADMIN_EMAIL')` (required in production) |
| role | `Role::Admin` |
| password | `env('ADMIN_PASSWORD')`; if empty, a random 16-character password is generated and **printed once** to the console |
| must_change_password | `true` |

So no password is ever stored in the repository. The mockup's visible `admin / admin123` box does **not** exist in the Laravel app; demo logins only come from `DemoSeeder` on a local/demo install.

## 3.3 Forced password change

- `EnsurePasswordIsChanged` middleware: if `auth()->user()->must_change_password`, redirect every request (except `password.*` and `logout`) to `password.edit`.
- `PasswordController@update` with `UpdatePasswordRequest`: `current_password` (rule `current_password`), `password` (`confirmed`, `Password::min(10)->letters()->numbers()`, different from current).
- On success: hash, set `must_change_password = false`, `Auth::logoutOtherDevices()`, regenerate session, redirect to dashboard with a toast.
- The same page is reachable later from the user menu to change password voluntarily.

## 3.4 Deactivated users

`EnsureUserIsActive` logs out a user whose `is_active` became false while they were logged in (for example, someone leaves the company). `Auth::attempt` already blocks them at login (`'is_active' => true` in the credentials).

## 3.5 Roles and authorization

Roles come from `App\Enums\Role`. Authorization is done with **policies and gates**, never by hiding buttons alone.

| Ability | Admin | Lead Tim | Engineer |
|---|:-:|:-:|:-:|
| View dashboard, boards, list, any ticket | ✅ | ✅ | ✅ |
| Create a ticket (any team) | ✅ | ✅ | ✅ |
| Edit fields / change status / add notes / steps | ✅ any | ✅ tickets of teams they lead or belong to | ✅ tickets of their team(s) or assigned to them |
| Assign or reassign | ✅ | ✅ within their team | ✅ only to themselves |
| Reopen a closed ticket (out of a Done status) | ✅ | ✅ | ❌ |
| Admin pages (SLA, statuses, templates, teams, users, holidays) | ✅ | ❌ | ❌ |

- `Gate::define('admin', fn (User $u) => $u->isAdmin())` in `AppServiceProvider`.
- `Gate::before()` is **not** used for admins, so policy rules are always explicit and tested.
- `TicketPolicy`: `viewAny`, `view`, `create`, `update`, `changeStatus`, `reopen`, `assign(User $actor, Ticket $t, ?User $assignee)`, `manageSteps`, `addNote`.
- The sidebar and buttons use `@can` so users only see what they can do.

## 3.6 Security checklist

- CSRF on every form and JS request (`<meta name="csrf-token">`, `X-CSRF-TOKEN` header in `http.js`).
- Passwords hashed with the framework default (bcrypt; `BCRYPT_ROUNDS=4` only in tests).
- Session cookie `secure` + `http_only` + `same_site=lax` in production (`SESSION_SECURE_COOKIE=true`).
- Mass assignment: models list `$fillable`; `role`, `is_active`, `must_change_password` are only set by admin actions.
- Output escaped with `{{ }}`; descriptions and notes are plain text (no HTML), shown with `nl2br(e(...))`.

## 3.7 Done when

- Login works with username or email; wrong credentials, inactive users and rate limiting all behave as above.
- The default Admin is created by `php artisan db:seed` with no password in git, and must change it on first login.
- Every rule in the role table has a policy test (see 05-tests.md).
