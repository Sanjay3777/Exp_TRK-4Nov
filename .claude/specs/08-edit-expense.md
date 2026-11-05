# Spec 08: Edit Expense

## Overview

- Step 08 lets a logged-in user edit any of their own expenses via a pre-populated form at `/expenses/<id>/edit`.
- The GET handler loads the existing expense from the database and renders the form with its current values.
- The POST handler validates the submission and updates the row in place.
- Ownership is enforced: a user can only edit expenses that belong to them. Both GET and POST load the expense first and return 404 if it is missing or belongs to someone else.
- Two new query helpers are added to `database/queries.py`: `get_expense_by_id` and `update_expense`.
- The transactions table in `profile.html` gains an "Edit" action link per row, which requires `get_recent_transactions` to also return the expense `id`.

## Depends on

- Step 01: Database setup (the `expenses` table exists with all required columns)
- Step 01: the `SPENDLY_DB` database override, used by the tests
- Step 03: Login and Logout (`session["user_id"]` is set; the `login_required` decorator exists)
- Step 05: Profile page renders transactions (the edit link lives there)
- Step 07: Add Expense (establishes the form pattern, the shared `CATEGORIES` constant and the shared `validate_expense_form` function this step reuses)

## Routes

- `GET /expenses/<int:expense_id>/edit`: render the edit form pre-populated with the existing expense values. Logged-in only.
- `POST /expenses/<int:expense_id>/edit`: validate and save the updated expense. Logged-in only.
- The URL variable is named `expense_id`, not `id`, so it does not shadow Python's built-in.

## Database changes

- No new tables or columns.
- All required columns already exist in `expenses`: `id`, `user_id`, `amount`, `category`, `date`, `description`.

## Templates

- Create `templates/edit_expense.html`:
  - Extends `base.html`.
  - A form with `method="POST"` and its action generated with `url_for('edit_expense', expense_id=expense.id)`.
  - Fields (identical to `add_expense.html`, all pre-filled with current values):
    - `amount`: number input labelled "Amount (₹)", `step="0.01"`, `min="0.01"`, `max="10000000"`, required, pre-filled and formatted to exactly 2 decimals (for example `346.24`, never `346.24000000000007`).
    - `category`: `<select>` with the options from `CATEGORIES`, pre-selected to the current value. If the stored category is not in `CATEGORIES`, no option is pre-selected and a blank "Select a category" option is shown first, so the user must choose one.
    - `date`: `<input type="date">`, required, `max` set to today's date, pre-filled to the current value.
    - `description`: text input, optional, `maxlength="200"`, pre-filled.
  - Hidden fields `date_from` and `date_to` carry the profile date filter that was active when the Edit link was clicked.
  - A submit button labelled "Save Changes", and a cancel link back to the profile page using `url_for('profile')`.
  - An error message block that re-populates the submitted (not original) values on validation failure.
- Modify `templates/profile.html`:
  - Add an "Actions" column header to the transactions table `<thead>`.
  - Add an "Edit" link cell per row, pointing to the edit route for that transaction, generated with `url_for('edit_expense', expense_id=tx.id, date_from=..., date_to=...)`, passing the active filter values (omitted when empty).
  - The Edit link has an accessible label that names the expense, for example "Edit Lunch on 2026-03-20".
  - Older expenses are reached by narrowing the profile date filter (Step 06): the table shows the latest 10 within the selected range, so a short range always exposes its expenses.

## Files to change

- `database/queries.py`
  - Add `get_expense_by_id(expense_id, user_id)`:
    - Fetches a single expense row only if it belongs to the given user.
    - Returns `None` if not found or if `user_id` does not match.
    - The query must scope to `id = ? AND user_id = ?`.
  - Add `update_expense(expense_id, user_id, amount, category, date, description)`:
    - Issues a parameterised `UPDATE` scoped to `id = ? AND user_id = ?` for ownership safety.
    - Commits, closes its connection, and returns the number of rows changed (0 or 1).
  - Modify `get_recent_transactions`:
    - Add `id` to the SELECT column list so templates can build edit links. Update the Step 05 tests that check the exact row keys so they expect `id` as well.
- `app.py`
  - Import `get_expense_by_id` and `update_expense` from `database.queries`.
  - Replace the GET-only placeholder at `/expenses/<int:id>/edit` with a full GET and POST handler:
    - Protect the route with the shared `login_required` decorator.
    - GET and POST both call `get_expense_by_id` first and `abort(404)` if it returns `None`, before any validation. A POST with bad data to someone else's expense therefore returns 404, not a form with errors.
    - GET: render `edit_expense.html` with the expense and `CATEGORIES`.
    - POST: validate with the shared `validate_expense_form` function from Step 07, call `update_expense`, and `abort(404)` if it returns 0 (the expense was deleted after the form opened). On success flash "Expense updated." (category `success`) and redirect to `url_for("profile")`, passing the validated `date_from` and `date_to` so the filter is kept. Re-render the form with the error otherwise.
    - The route accepts both GET and POST.
- `templates/profile.html`
  - Add `<th>Actions</th>` to the table header.
  - Add an "Edit" link cell per transaction row.

## Files to create

- `templates/edit_expense.html`

## New dependencies

- None

## Rules for implementation

- No SQLAlchemy or ORMs. Raw `sqlite3` only, via `get_db()`.
- Parameterised queries only. Never string-format values into SQL.
- `PRAGMA foreign_keys = ON` must be enabled on every connection (already handled in `get_db()`).
- Categories come from the shared `CATEGORIES` constant. Never type the list into this template.
- `get_expense_by_id` must scope its query to `id = ? AND user_id = ?`. Return `None` if not found or if the ownership check fails.
- `update_expense` must also include `user_id = ?` in its `WHERE` clause as a second ownership guard.
- Unauthenticated access to both GET and POST must redirect to `/login`.
- If the expense does not exist or belongs to another user, return 404 for both GET and POST, checked before validation.
- The owner is always `session["user_id"]`. A forged `user_id` form field is ignored.
- Validation uses the single shared `validate_expense_form` function from Step 07, so both forms follow the same rules (finite amount above 0 and at most 10,000,000 rounded to 2 decimals, category in `CATEGORIES`, date normalised to `YYYY-MM-DD` and not in the future or before 2000-01-01, description at most 200 characters and `None` when blank). Do not copy the rules into this route.
- On any validation error, re-render the form with the first error message and the submitted (not original) values pre-filled, status 200.
- After a successful update, redirect to `url_for("profile")` with the active date filter. Do not render the form again.
- Changes to one expense never touch any other row.
- Currency must always display as ₹. Never £ or $.
- Use CSS variables. Never hardcode hex values.
- No inline styles.
- All templates extend `base.html`.
- Use `url_for()` for every internal link. Never hardcode paths.

## Tests to write

- File: `tests/test_edit_expense.py`
- Every test uses its own temporary database via `SPENDLY_DB`, and tests that need "another user" create a real second user.

### Unit tests

| Function | Input | Expected output |
|---|---|---|
| `get_expense_by_id` | Valid `expense_id` and correct `user_id` | Returns the matching row as a dict-like object |
| `get_expense_by_id` | Valid `expense_id` and wrong `user_id` | Returns `None` |
| `get_expense_by_id` | Non-existent `expense_id` | Returns `None` |
| `update_expense` | Valid `expense_id`, correct `user_id`, new `amount=99.0` | Row in DB reflects the updated amount |
| `update_expense` | Valid `expense_id`, wrong `user_id` | Row in DB unchanged; returns 0; no error raised |
| `update_expense` | Valid `expense_id`, correct `user_id` | Returns 1 |
| `update_expense` | `description=None` | Row's `description` becomes NULL |
| `update_expense` | Two expenses, one updated | The other expense is unchanged |
| `get_recent_transactions` | `user_id` with expenses | Each item also has `id` |

### Route tests

- `GET /expenses/<id>/edit` unauthenticated: redirects to `/login` (302).
- `GET /expenses/<id>/edit` authenticated, own expense: returns 200; the response contains a form pre-filled with the current values; the `<select>` has the correct category pre-selected.
- `GET /expenses/<id>/edit` authenticated, other user's expense: returns 404.
- `GET /expenses/<id>/edit` authenticated, non-existent id: returns 404.
- `POST /expenses/<id>/edit` unauthenticated: redirects to `/login` (302).
- `POST /expenses/<id>/edit` authenticated, valid data: redirects to `/profile` (302); the updated values are reflected in the database.
- `POST /expenses/<id>/edit` authenticated, other user's expense: returns 404, even when the submitted data is invalid, and the row is unchanged.
- `POST /expenses/<id>/edit` authenticated, non-existent id: returns 404.
- `POST /expenses/<id>/edit` authenticated, with a forged `user_id` form field: the row keeps its original owner.
- `POST /expenses/<id>/edit` authenticated, missing amount: returns 200; the response contains an error message.
- `POST /expenses/<id>/edit` authenticated, `amount` = 0: returns 200; the response contains an error message.
- `POST /expenses/<id>/edit` authenticated, non-numeric amount: returns 200; the response contains an error message.
- `POST /expenses/<id>/edit` authenticated, invalid category: returns 200; the response contains an error message.
- `POST /expenses/<id>/edit` authenticated, invalid date string: returns 200; the response contains an error message.
- `POST /expenses/<id>/edit` authenticated, negative amount, `inf`, `nan`, `0.001`, a 201-character description, a future date, or an unpadded date `2026-3-5` (stored as `2026-03-05`): handled exactly as in Step 07.
- On a failed POST, the response keeps the submitted amount, category, date and description (not the original values).
- A POST with `date_from` and `date_to` redirects to `/profile` with those values in the query string.
- After a valid POST, following the redirect shows "Expense updated.".
- `GET /profile` authenticated: each transaction row has an Edit link pointing to its own edit URL, and the table has an "Actions" column header.
- `POST /expenses/<id>/edit` authenticated, no description: redirects to `/profile` (302); the row is updated with `description` = NULL.

## Definition of done

- [ ] Visiting `/expenses/<id>/edit` while logged out redirects to `/login`
- [ ] Visiting `/expenses/<id>/edit` for a non-existent or other user's expense returns 404
- [ ] Visiting `/expenses/<id>/edit` while logged in shows a form pre-filled with the expense's current values
- [ ] The category dropdown has the correct category pre-selected
- [ ] Submitting valid changes redirects to `/profile` with "Expense updated." and the updated values appear in the transaction list (when the expense is among the latest 10 in the active range)
- [ ] The active date filter is kept after saving
- [ ] A POST to another user's or a missing expense returns 404 before any validation
- [ ] Validation matches Step 07 exactly because both use `validate_expense_form`
- [ ] Tests run against a fresh temporary database
- [ ] Submitting with a missing or zero amount re-renders the form with an error and the submitted values retained
- [ ] Submitting with an invalid category re-renders the form with an error
- [ ] Submitting with an invalid date re-renders the form with an error
- [ ] Submitting without a description saves the expense with no error
- [ ] Each row in the profile transaction table has an "Edit" link pointing to the correct URL
