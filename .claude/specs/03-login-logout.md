# Spec: Login and Logout

## Overview
Login and Logout lets a registered user authenticate with their email and password and later end that session. This is Step 3 of the roadmap: Step 1 built the `users` table and password hashing, Step 2 wired up account creation, but there is still no way to actually sign in — `/login` only renders the form on `GET`, and `/logout` is a raw-string stub. This step adds session-based authentication (Flask's built-in signed-cookie `session`) so a submitted email/password is verified against the stored hash, a logged-in session is established, and `/logout` clears it. This unblocks Step 4 (profile) and any future route that needs to know who is logged in.

## Depends on
- Step 1 (database setup) — `users` table and `get_db()` with `PRAGMA foreign_keys = ON`. Already implemented.
- Step 2 (registration) — `create_user()` and `get_user_by_email()` in `database/db.py`, and at least one real account (plus the seeded demo user `demo@spendly.com` / `demo123`) to log in with. Already implemented.

## Routes
- `POST /login` — accept the sign-in form submission, verify email/password against the stored hash, establish a session on success, redirect to `/` on success or re-render the form with an error — public
- `GET /login` — unchanged (already implemented, keep as-is)
- `GET /logout` — clear the session and redirect to `/login` — logged-in (safe to hit anonymously too; it just no-ops and redirects)

## Database changes
No database changes. `users.password_hash` and the existing `get_user_by_email()` helper in `database/db.py` are sufficient to verify credentials.

## Templates
- **Create:** none
- **Modify:** `templates/login.html` — replace the hardcoded `action="/login"` with `action="{{ url_for('login') }}"` per the no-hardcoded-URLs rule. The existing `{% if error %}` block and `.auth-error` styling already support displaying a validation error; no other markup changes needed.

## Files to change
- `app.py` —
  - Set `app.secret_key` from an environment variable (`os.environ.get("SECRET_KEY", "dev")`) so Flask's `session` can sign cookies; this is stdlib/Flask-only, not a new dependency.
  - Add `methods=["GET", "POST"]` to the `/login` route; on `POST`, read `email`/`password` from the form, look up the user with `get_user_by_email`, verify the password with `werkzeug.security.check_password_hash` against `user["password_hash"]`, and on success store `session["user_id"] = user["id"]` then redirect to `url_for("landing")`. On missing fields or a bad email/password combo, re-render `login.html` with a generic error (do not reveal whether the email exists).
  - Replace the `/logout` stub: clear the session (`session.clear()` or `session.pop("user_id", None)`) and redirect to `url_for("login")`. No template render needed for a redirect-only route.
- `templates/login.html` — fix hardcoded form `action` to use `url_for()`.

## Files to create
No new files.

## New dependencies
No new dependencies. `flask.session` and `werkzeug.security.check_password_hash` are already available via the existing `flask` and `werkzeug` packages in `requirements.txt`.

## Rules for implementation
- No SQLAlchemy or ORMs
- Parameterised queries only
- Passwords hashed with werkzeug — for login, verify with `check_password_hash`, never compare plaintext or re-hash
- Use CSS variables — never hardcode hex values
- All templates extend `base.html`
- No hardcoded URLs in templates — use `url_for()` everywhere, including the `login.html` form action
- Use a single generic error message ("Invalid email or password.") for both "no such user" and "wrong password" cases, so login does not leak which emails are registered
- Do not implement `/profile` or route-protection decorators in this step — that belongs to Step 4; this step only needs to establish and clear `session["user_id"]`
- Do not put session/credential-checking logic anywhere but `app.py`'s route function; DB lookups still go through `database/db.py` only

## Definition of done
- [ ] Visiting `GET /login` still renders the form exactly as before
- [ ] Submitting the seeded demo credentials (`demo@spendly.com` / `demo123`) logs in successfully and redirects to `/`
- [ ] Submitting a correct email with the wrong password shows the generic `.auth-error` message and does not log in
- [ ] Submitting an email that doesn't exist shows the same generic `.auth-error` message and does not log in
- [ ] Submitting with a missing email or missing password shows an inline error and does not raise a server error
- [ ] After a successful login, the session cookie is set and persists across requests
- [ ] Visiting `GET /logout` after logging in clears the session and redirects to `/login`
- [ ] Visiting `GET /logout` while not logged in does not error and redirects to `/login`
- [ ] `login.html`'s form now posts via `url_for('login')` instead of a hardcoded path
- [ ] No new packages were added to `requirements.txt`
- [ ] The app still starts cleanly on port 5001 with no errors (`python app.py`)
