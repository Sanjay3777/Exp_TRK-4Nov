# Spec 07: Add Expense

## Overview

- Step 07 lets a logged-in user submit a new expense through a dedicated form page at `/expenses/add`.
- The route already exists as a GET placeholder. This step upgrades it to a full GET and POST handler.
- Validated data is inserted into the `expenses` table, and the user is redirected back to the profile page on success.
- A reusable `insert_expense()` query helper is added to `database/queries.py`.
- An "Add Expense" button is added to `profile.html` and a nav link is added to `base.html`, so users can navigate to the form.

## Depends on

- Step 01: Database setup (the `expenses` table exists with all required columns)
- Step 01: the `SPENDLY_DB` database override, used by the tests
- Step 03: Login and Logout (`session["user_id"]` is set; the `login_required` decorator exists)
- Steps 04 and 05: Profile page exists and is the natural redirect target after saving

## Routes

- `GET /expenses/add`: render the add-expense form. Logged-in only.
- `POST /expenses/add`: validate and insert the new expense. Logged-in only.

## Database changes

- No database changes.
- The `expenses` table already has all required columns: `id`, `user_id`, `amount`, `category`, `date`, `description`, `created_at`.

## Templates

- Create `templates/add_expense.html`:
  - Extends `base.html`.
  - A form with `method="POST"` and its action generated with `url_for('add_expense')`.
  - Fields:
    - `amount`: number input labelled "Amount (₹)", `step="0.01"`, `min="0.01"`, `max="10000000"`, required.
    - `category`: `<select>` with exactly 7 fixed options: Food, Transport, Bills, Health, Entertainment, Shopping, Other.
    - `date`: `<input type="date">`, required, defaults to today's date, `max` set to today's date.
    - `description`: text input, optional, `maxlength="200"`.
  - A submit button labelled "Save Expense", and a cancel link back to the profile page using `url_for('profile')`.
  - A flash/error message block to display the validation error, re-populating previously submitted values on failure (amount, selected category, date and description).
  - The category options are rendered from the shared `CATEGORIES` list passed in by the route, not typed out in the template.
- Modify `templates/profile.html`:
  - Add an "Add Expense" button or link pointing to `url_for('add_expense')`, near the transaction table heading.
  - Display flashed messages near the top of the page (category `success`), so "Expense added." is visible after the redirect.
- Modify `templates/base.html`:
  - Add an "Add Expense" link in the navbar, visible only when `session.user_id` is set.

## Files to change

- `app.py`: replace the GET-only placeholder at `/expenses/add` with a GET and POST handler:
  - Both methods are protected with the shared `login_required` decorator from Step 03.
  - GET: render `add_expense.html`.
  - POST: read the form fields, validate, call `insert_expense()`, flash "Expense added." (category `success`) and redirect to `url_for("profile")` on success.
- `database/queries.py`:
  - Add the constant `CATEGORIES` (the seven names, in order). The add form, the validator and Step 08's edit form all use it as the single source.
  - Add `insert_expense(user_id, amount, category, date, description)`, which returns the new row's id and closes its connection.
- `templates/profile.html`: add the "Add Expense" button.
- `templates/base.html`: add the "Add Expense" navbar link for logged-in users.

## Files to create

- `templates/add_expense.html`

## New dependencies

- None

## Rules for implementation

- No SQLAlchemy or ORMs. Raw `sqlite3` only, via `get_db()`.
- Parameterised queries only. Never string-format values into SQL.
- `PRAGMA foreign_keys = ON` must be enabled on every connection (already handled in `get_db()`).
- Unauthenticated access to both GET and POST `/expenses/add` must redirect to `/login`, and an unauthenticated POST must not insert a row.
- The owner of the new row is always `session["user_id"]`. Any `user_id` field in the form is ignored.
- Put the validation in one shared function, `validate_expense_form(form, today)`, that returns either the cleaned values (amount rounded, date normalised, description stripped or `None`) or the first error message. Step 08 reuses it. It takes `today` as an argument so tests are not clock-dependent.
- Validation rules for POST, checked in this order, stopping at the first failure and showing only that one error:
  1. `amount`: required; parse with `float()` and catch `ValueError`. It must be finite (reject `inf` and `nan`), greater than 0 and at most 10,000,000. Round to 2 decimals before saving, so `0.001` is rejected because it rounds to 0.
  2. `category`: required; must be one of `CATEGORIES`; reject anything else.
  3. `date`: required; parse with `datetime.strptime(value, "%Y-%m-%d")`, then re-format with `strftime("%Y-%m-%d")` and save that value, so `2026-3-5` is stored as `2026-03-05`. It must not be in the future (server date) and must not be before 2000-01-01.
  4. `description`: optional; strip whitespace; store `None` if blank; reject more than 200 characters.
- On any validation error, re-render the form with status 200, the error message and the previously submitted values pre-filled.
- After a successful insert, redirect to `url_for("profile")`. Do not render the form again.
- Currency must always display as ₹. Never £ or $.
- Use CSS variables. Never hardcode hex values.
- No inline styles.
- All templates extend `base.html`.
- Use `url_for()` for every internal link. Never hardcode paths.

## Tests to write

- File: `tests/test_add_expense.py`
- Every test uses its own temporary database via `SPENDLY_DB` (a fresh file per test), so these tests never change the data the Step 05 tests rely on.

### Unit tests

| Function | Input | Expected output |
|---|---|---|
| `insert_expense` | Valid `user_id`, `amount=50.0`, `category="Food"`, `date="2026-03-20"`, `description="Lunch"` | Row inserted; querying the DB returns the new row |
| `insert_expense` | `description=None` | Row inserted with `description` stored as NULL |
| `insert_expense` | Valid input | Returns the new row's id |
| `insert_expense` | `user_id` that does not exist | Raises a foreign key constraint error |

### Route tests

- `GET /expenses/add` unauthenticated: redirects to `/login` (302).
- `GET /expenses/add` authenticated: returns 200; the response contains the category `<select>` with all 7 options; the response contains a form with method POST.
- `POST /expenses/add` unauthenticated: redirects to `/login` (302).
- `POST /expenses/add` authenticated, valid data (`amount=50.0`, `category=Food`, `date=2026-03-20`, `description=Lunch`): redirects to `/profile` (302); a new expense row exists in the DB for the test user.
- `POST /expenses/add` authenticated, missing amount: returns 200; the response contains an error message.
- `POST /expenses/add` authenticated, `amount` = 0: returns 200; the response contains an error message.
- `POST /expenses/add` authenticated, non-numeric amount: returns 200; the response contains an error message.
- `POST /expenses/add` authenticated, invalid category (not in the fixed list): returns 200; the response contains an error message.
- `POST /expenses/add` authenticated, invalid date string: returns 200; the response contains an error message.
- `POST /expenses/add` authenticated, negative amount, `inf`, `nan` and `0.001`: each returns 200 with an error message and inserts nothing.
- `POST /expenses/add` authenticated, description of 201 characters: returns 200 with an error message.
- `POST /expenses/add` authenticated, future date or a date before 2000-01-01: returns 200 with an error message.
- `POST /expenses/add` authenticated, date `2026-3-5`: the stored date is `2026-03-05`.
- `POST /expenses/add` authenticated, whitespace-only description: stored as NULL.
- `POST /expenses/add` authenticated, with a forged `user_id` form field: the row belongs to the session user.
- `POST /expenses/add` unauthenticated: no row is inserted.
- On a failed submission, the response keeps the entered amount, the selected category, the date and the description.
- After a valid POST, following the redirect shows "Expense added." and the new expense on the profile page.
- `POST /expenses/add` authenticated, no description (optional field): redirects to `/profile` (302); the row is inserted with `description` = NULL.

## Definition of done

- [ ] Visiting `/expenses/add` while logged out redirects to `/login`
- [ ] Visiting `/expenses/add` while logged in shows a form with amount, category, date and description fields
- [ ] The category dropdown contains exactly: Food, Transport, Bills, Health, Entertainment, Shopping, Other
- [ ] Submitting a valid expense redirects to `/profile`, shows "Expense added.", and the expense is counted in the totals (it appears in the table when it is among the latest 10)
- [ ] Submitting with a missing or zero amount re-renders the form with an error and previously entered values retained
- [ ] Submitting with an invalid category re-renders the form with an error
- [ ] Submitting with an invalid, future or unpadded date is rejected or normalised as specified
- [ ] Non-finite, negative, over-limit and sub-cent amounts are rejected
- [ ] The saved row always belongs to the logged-in user
- [ ] Tests run against a fresh temporary database each time
- [ ] Submitting without a description saves the expense with no error
- [ ] The "Add Expense" button on the profile page navigates to `/expenses/add`
- [ ] Navbar shows the "Add Expense" link when logged in
