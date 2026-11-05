# Spec 03: Login and Logout

## Overview

- This step converts the `/login` stub into a functional POST handler that:
  - Verifies credentials against the `users` table.
  - Stores the authenticated user's id in `session["user_id"]`.
  - Redirects to the landing page (until a dashboard route exists).
- It also implements `/logout`, which clears the session and redirects to `/`.
- After this step the app can distinguish logged-in users from guests.
- This is a prerequisite for all expense features.

## Depends on

- Step 01: Database Setup (the `users` table must exist)
- Step 02: Registration (`create_user` and password hashing must be in place; a user must exist to log in against)
- `app.secret_key` set in `app.py` (done in Step 02). Without it `session["user_id"]` cannot be stored.
- The seeded demo user from Step 01 `seed_db()` (`demo@spendly.com` / `demo123`), used by the Definition of done.

## Routes

- `GET /login`: render the login form. Public.
- `POST /login`: validate credentials, set the session, redirect. Public.
- `GET /logout`: clear the session and redirect to `/`. Public (no login required).

## Database changes

- No database changes.
- The `users` table from Step 01 already stores `email` and `password_hash`.
- One new DB helper to add to `database/db.py`: `get_user_by_email(email)`
  - Queries the `users` table by email.
  - Returns the matching user row, or `None` if not found.
  - Closes its own connection before returning, the same convention as `create_user`.
  - Must live in `database/db.py`, not inline in the route.

## Templates

- Modify `templates/login.html`:
  - Add a POST form with `email` and `password` fields.
  - Set the form action to `url_for('login')` with `method="post"`.
  - Add a block to display all flashed messages, with distinct styling by category (`error` and `success`). This is needed so the "registration successful" message from Step 02 appears after the redirect to `/login`.
  - Add a link to the register page (`url_for('register')`) for new users.

## Files to change

- `app.py`: implement `login()` as a GET and POST handler, implement `logout()`, and add the shared `login_required` guard.
- `database/db.py`: add the `get_user_by_email()` helper.
- `templates/login.html`: add the POST form and flash message display.

## Files to create

- None

## New dependencies

- None.
- `werkzeug.security.check_password_hash` is already available.

## Rules for implementation

- No SQLAlchemy or ORMs. Use raw `sqlite3` via `get_db()`.
- Parameterised queries only. Never use f-strings in SQL.
- Normalise the submitted email exactly as Step 02 does before the lookup: strip whitespace and lower-case it. `Demo@Spendly.com` must log in as `demo@spendly.com`. Never strip the password.
- Verify passwords with `werkzeug.security.check_password_hash`.
- The session key for the logged-in user must be `session["user_id"]` (integer).
- Also store `session["user_name"]` (the user's name) at login, so the navbar can show it without a DB query on every request. It is cleared with the rest of the session on logout.
- Use `flask.session`. Do not roll a custom session mechanism.
- On failed login, show a generic flash error: "Invalid email or password."
  - Do not reveal which field was wrong.
- Treat blank email or password as a failed login with the same generic error.
- When the email is not found, still run a dummy `check_password_hash` so response time does not reveal whether the email exists.
- Call `session.clear()` before storing `session["user_id"]` on a successful login.
- If a logged-in user opens `GET /login`, redirect them to `url_for("landing")`.
- Add one reusable `login_required` decorator in `app.py`:
  - If `session.get("user_id")` is missing, redirect to `url_for("login", next=<requested path>)`.
  - After a successful login, redirect to `next` only if it is a relative path on this site (starts with `/` and not `//`); otherwise redirect to `url_for("landing")`.
  - Later steps (profile, add, edit, delete) must use this decorator instead of repeating the session check.
- After a successful login with no valid `next`, redirect to `url_for("landing")` until a dashboard exists.
- `logout()` must call `session.clear()` and then redirect to `url_for("landing")`.
- All templates must extend `base.html`.
- Use CSS variables. Never hardcode hex values.
- Use `url_for()` for every internal link. Never hardcode paths.

## Definition of done

- [ ] `GET /login` renders the login form with email and password fields
- [ ] Submitting valid credentials (`demo@spendly.com` / `demo123`) sets `session["user_id"]` and redirects to `/`
- [ ] Submitting a wrong password shows the "Invalid email or password." flash and stays on the login page
- [ ] Submitting an unregistered email shows the same generic error flash
- [ ] `Demo@Spendly.com` (different case, surrounding spaces) logs in successfully
- [ ] A blank email or password shows the generic error
- [ ] A logged-in user opening `/login` is redirected to `/`
- [ ] A route decorated with `login_required` redirects guests to `/login?next=...`, and login returns them there; an external `next` such as `//evil.com` is ignored
- [ ] `GET /logout` clears the session and redirects to `/`
- [ ] After logout, `session["user_id"]` is no longer present
- [ ] The `/logout` route no longer returns the raw stub string
