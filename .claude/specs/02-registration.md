# Spec 02: Registration

## Overview

- This step upgrades the existing stub `GET /register` route into a fully functional route that handles both GET and POST.
- The form accepts four fields: `name`, `email`, `password` and `confirm_password`.
- On success, the app flashes a success message and redirects to `/login`.
- This is the entry point for all authenticated features. Login, profile and expense tracking all require a registered user.

## Depends on

- Step 01: Database setup (the `users` table and `get_db()`)

## Routes

- `GET /register`: render the registration form. Public. (Already exists as a stub; upgrade it.)
- `POST /register`: process the form, validate input, insert the user, then redirect to `/login`. Public.

## Database changes

- No new tables or columns.
- The existing `users` table covers all requirements.
- One new DB helper to add to `database/db.py`: `create_user(name, email, password)`
  - Hashes the password with werkzeug.
  - Inserts a row into `users`.
  - Returns the new user's id.
  - Raises `sqlite3.IntegrityError` if the email is already taken (UNIQUE constraint).
  - Commits the insert and closes its own connection before returning (or before raising), the same convention every DB helper follows.

## Templates

- Modify `templates/register.html`:
  - Change the form action to `url_for('register')` with `method="post"`.
  - Add `name` attributes to all inputs: `name`, `email`, `password`, `confirm_password`.
  - Add a confirm password input, since the current template has only name, email and password.
  - Replace the current single `error` message block with a block that displays flashed messages (for example "Email already registered" and "Passwords do not match"), using the flash category to style errors and successes differently.
  - On a failed submission, keep the entered `name` and `email` in the form. Never re-fill the password fields.
  - Keep all existing visual design unchanged. Reuse the existing auth classes.

## Files to change

- `app.py`: upgrade `register()` to handle GET and POST, and add flash and redirect logic.
- `database/db.py`: add the `create_user()` helper.
- `templates/register.html`: wire up the form action and method, add the confirm password field, and add flash message display.

## Files to create

- None

## New dependencies

- None.
- Uses `werkzeug.security` (already installed) and Flask's built-in `flash`, `redirect` and `url_for`.

## Rules for implementation

- No SQLAlchemy or ORMs.
- Parameterised queries only. Never use f-strings in SQL.
- Hash passwords with `werkzeug.security.generate_password_hash`. Never store plaintext.
- `app.secret_key` must be set in `app.py` for `flash()` to work. Read it from the `SECRET_KEY` environment variable, with a hardcoded dev string as the fallback. The fallback must be replaced before any deployment.
- Before validating, strip leading and trailing whitespace from `name` and `email` (never from passwords), and lower-case `email`. Store the lower-cased email.
- Server-side validation must check in this order:
  - 1. `name`, `email` and `password` are non-empty after stripping. Error: "All fields are required."
  - 2. `password` is at least 8 characters. Error: "Password must be at least 8 characters."
  - 3. `password` equals `confirm_password`. Error: "Passwords do not match."
  - 4. The email is not already registered (catch `sqlite3.IntegrityError`). Error: "Email already registered."
- On any validation failure, re-render the form with a flashed error (category `error`) and HTTP status 400. Do not redirect.
- On success, flash a success message (category `success`) and redirect to `url_for('login')`.
- The route accepts only GET and POST. Flask returns 405 for any other method, so no manual `abort(405)` is needed.
- All templates must extend `base.html`.
- Use CSS variables. Never hardcode hex values.
- Use `url_for()` for every internal link. Never hardcode URLs.

## Definition of done

- [ ] `GET /register` renders the registration form without errors
- [ ] Submitting with all valid fields creates a new user in `users` and redirects to `/login`
- [ ] Submitting with mismatched passwords re-renders the form with an error, with no DB insert
- [ ] Submitting with an already-registered email re-renders with "Email already registered"
- [ ] Submitting with any empty (or whitespace-only) field re-renders with "All fields are required."
- [ ] Submitting a password shorter than 8 characters re-renders with the minimum-length error, with no DB insert
- [ ] `A@x.com` and `a@x.com` are treated as the same email
- [ ] Failed submissions return status 400 and keep the entered name and email
- [ ] The success flash appears on the login page
- [ ] Password is stored as a hash, never plaintext, verifiable by inspecting `spendly.db`
- [ ] No duplicate user is created on repeated valid submissions with the same email
