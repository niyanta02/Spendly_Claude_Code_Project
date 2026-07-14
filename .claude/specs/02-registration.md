# Spec: Registration

## Overview
Registration lets a visitor create a Spendly account by submitting their name, email, and password from the existing `register.html` form. This is Step 2 of the roadmap: the database layer (Step 1) already provides `create_user()` and `get_user_by_email()` in `database/db.py`, but `app.py`'s `/register` route currently only handles `GET` and renders the template — there is no logic to process the submitted form, validate input, or persist a new user. This step wires that up so accounts can actually be created, unblocking login (Step 3) and everything downstream.

## Depends on
- Step 1 (database setup) — requires the `users` table and the `create_user(name, email, password)` / `get_user_by_email(email)` helpers in `database/db.py`. Both are already implemented, so this dependency is satisfied.

## Routes
- `POST /register` — accept the registration form submission, validate input, create the user, redirect to `/login` on success or re-render the form with an error — public
- `GET /register` — unchanged (already implemented, keep as-is)

## Database changes
No database changes. The `users` table, `create_user()`, and `get_user_by_email()` already exist in `database/db.py` from Step 1 and are sufficient for this step.

## Templates
- **Create:** none
- **Modify:** `templates/register.html` — replace the hardcoded `action="/register"` with `action="{{ url_for('register') }}"` per the no-hardcoded-URLs rule. The existing `{% if error %}` block and `.auth-error` styling already support displaying a validation error; no other markup changes needed.

## Files to change
- `app.py` — add `methods=["GET", "POST"]` to the `/register` route; on `POST`, read `name`, `email`, `password` from the form, validate them, check for an existing user via `get_user_by_email`, call `create_user` on success, then redirect to `url_for('login')`. On any validation failure or duplicate email, re-render `register.html` with an `error` message (no redirect).
- `templates/register.html` — fix hardcoded form `action` to use `url_for()`.

## Files to create
No new files.

## New dependencies
No new dependencies. Use the existing `werkzeug.security` (already used inside `create_user`) and stdlib only.

## Rules for implementation
- No SQLAlchemy or ORMs
- Parameterised queries only
- Passwords hashed with werkzeug (already handled inside `create_user` — pass the raw password, do not hash twice)
- Use CSS variables — never hardcode hex values
- All templates extend `base.html`
- No hardcoded URLs in templates — use `url_for()` everywhere, including the `register.html` form action
- Do not add session/login-on-register logic — no `SECRET_KEY` or session handling exists yet in `app.py`; that belongs to the login step, not this one
- Validate on the server even though the form has HTML5 `required`/`type="email"` attributes — never trust client-side validation alone
- Treat `sqlite3.IntegrityError` (or a pre-check via `get_user_by_email`) as an expected "email already registered" case, not a 500

## Definition of done
- [ ] Visiting `GET /register` still renders the form exactly as before
- [ ] Submitting the form with a new name/email/password creates a row in `users` with a hashed (not plaintext) password
- [ ] After a successful submission, the browser is redirected to `/login`
- [ ] Submitting an email that already exists shows the `.auth-error` message on the re-rendered `register.html` page and does not create a duplicate row
- [ ] Submitting with a missing name, missing/invalid email, or empty password shows an inline error and does not create a user
- [ ] `register.html`'s form now posts via `url_for('register')` instead of a hardcoded path
- [ ] No new packages were added to `requirements.txt`
- [ ] The app still starts cleanly on port 5001 with no errors (`python app.py`)
