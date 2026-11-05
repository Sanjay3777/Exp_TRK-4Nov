# Spec 09: Delete Expense

## Overview

- Step 09 lets a logged-in user permanently delete one of their own expenses directly from the profile transaction table.
- A "Delete" button per row submits a POST request to `/expenses/<id>/delete`.
- The handler removes the row only if it belongs to the signed-in user, flashes "Expense deleted.", and redirects back to `/profile`, keeping the active date filter.
- There is no separate confirmation page. A browser-side `confirm()` dialog, attached by a listener in `static/js/main.js` from a `data-confirm` attribute on the form, is used to prevent accidental deletions. No inline JavaScript is used.
- Known limitation: with JavaScript disabled the confirmation does not run and the form submits directly. This is accepted for this step.
- Only one new mutation helper is added: `delete_expense` in `database/queries.py`. It returns the number of rows deleted, so the route needs no separate ownership lookup.

## Depends on

- Step 01: Database setup (the `expenses` table exists)
- Step 01: the `SPENDLY_DB` database override, used by the tests
- Step 03: Login and Logout (`session["user_id"]` is set; the `login_required` decorator exists)
- Step 05: Profile page renders transactions (the delete button lives there)
- Step 06: the profile date filter (`date_from` and `date_to`) that the redirect preserves
- Step 08: Edit Expense (the Actions column already exists in the profile transactions table)

## Routes

- `POST /expenses/<int:expense_id>/delete`: delete the expense row if it belongs to the user, then redirect to `/profile`. Logged-in only.
- No GET handler. A bare GET to this URL must return 405 for every visitor, including logged-out ones. This differs from the other protected routes on purpose: the method check happens before the login check.

## Database changes

- No new tables or columns.
- The `expenses` table already has all required columns.

## Templates

- Modify `templates/profile.html`:
  - Inside the existing "Actions" `<td>` per transaction row, add a delete form:
    - A `<form>` with `method="POST"`, `class="inline-form"`, `data-confirm="Delete this expense?"` and its action generated with `url_for('delete_expense', expense_id=tx.id, date_from=..., date_to=...)`, passing the active filter values (omitted when empty).
    - No `style` attribute and no `onsubmit` attribute. The `.inline-form` class (`display: inline`) lives in the stylesheet, so the page has no inline styles at all.
    - A `<button type="submit">` with `class="btn-delete"` and the label "Delete", plus an accessible label naming the expense, for example `aria-label="Delete Lunch on 2026-03-20"`.
  - The "Edit" link from Step 08 remains alongside the new "Delete" button.

## Files to change

- `database/queries.py`
  - Add `delete_expense(expense_id, user_id)`:
    - Issues a parameterised `DELETE FROM expenses WHERE id = ? AND user_id = ?`.
    - The dual-column `WHERE` clause is the ownership guard.
    - Commits, and closes the connection in a `try/finally` so it is closed even when the delete raises.
    - Returns the number of rows deleted (0 or 1).
- `app.py`
  - Import `delete_expense` from `database.queries`.
  - Replace the GET-only placeholder at `/expenses/<int:expense_id>/delete` with a POST-only handler (the route accepts only POST):
    - Protect it with the shared `login_required` decorator.
    - Call `delete_expense(expense_id, session["user_id"])`. If it returns 0, `abort(404)`.
    - Otherwise flash "Expense deleted." (category `success`) and redirect to `url_for("profile")` with the validated `date_from` and `date_to`.
  - Set `app.config["SESSION_COOKIE_SAMESITE"] = "Lax"` explicitly, so cross-site POSTs do not carry the session cookie.
- `templates/profile.html`: add the delete form inside the existing Actions `<td>`.
- `static/js/main.js`: add one listener that, on submit of any form with a `data-confirm` attribute, calls `confirm()` with that text and cancels the submit if the user declines. The message is read from the attribute only and never built from expense data.
- `static/css/profile.css`
  - Add `.inline-form` and a `.btn-delete` style. `.btn-delete` uses the existing `--danger` and `--danger-light` variables from `style.css`, resets the native button border, font and background so it matches the Edit link beside it, and never hardcodes hex values.

## Files to create

- None

## New dependencies

- None

## Rules for implementation

- No SQLAlchemy or ORMs. Raw `sqlite3` only, via `get_db()`.
- Parameterised queries only. Never string-format values into SQL.
- `PRAGMA foreign_keys = ON` must be enabled on every connection (already handled in `get_db()`).
- `delete_expense` must scope its `DELETE` to `id = ? AND user_id = ?`. This is the ownership guard that prevents one user deleting another's expense.
- The route must only accept POST. A bare GET to the URL must return 405.
- Unauthenticated POST must redirect to `/login` (302) and delete nothing.
- If `delete_expense` reports 0 rows (the expense does not exist or belongs to another user), return 404. Do not look the expense up first.
- After a successful deletion, redirect to `url_for("profile")` with the active date filter. Do not render a template.
- No inline styles and no inline event handlers anywhere in `profile.html`.
- Deletion is permanent with no undo.
- Use CSS variables. Never hardcode hex values.
- All templates extend `base.html`.
- Currency must always display as ₹. Never £ or $.
- Use `url_for()` for every internal link. Never hardcode paths.

## Tests to write

- File: `tests/test_delete_expense.py`
- Every test uses its own temporary database via `SPENDLY_DB`, so deleting rows never affects other test files.

### Unit tests

| Function | Input | Expected output |
|---|---|---|
| `delete_expense` | Valid `expense_id` and correct `user_id` | Row removed from the DB; returns 1 |
| `delete_expense` | Valid `expense_id` and wrong `user_id` | Row remains in the DB; returns 0; no error raised |
| `delete_expense` | Non-existent `expense_id` | Returns 0, no error raised, DB unchanged |
| `delete_expense` | Two expenses, one deleted | The other expense (same user) and other users' expenses are unchanged |
| `delete_expense` | Forced error during the delete | The connection is still closed |

### Route tests

- `POST /expenses/<id>/delete` unauthenticated: redirects to `/login` (302).
- `POST /expenses/<id>/delete` authenticated, own expense: redirects to `/profile` (302); the row no longer exists in the database.
- `POST /expenses/<id>/delete` authenticated, other user's expense: returns 404; the row still exists in the database.
- `POST /expenses/<id>/delete` authenticated, non-existent id: returns 404.
- `POST /expenses/<id>/delete` unauthenticated: the row still exists afterwards.
- `POST /expenses/<id>/delete` with a non-numeric id (for example `/expenses/abc/delete`): returns 404 through routing.
- `POST /expenses/<id>/delete` with `date_from` and `date_to`: redirects to `/profile` with those values in the query string.
- After a valid POST, following the redirect shows "Expense deleted." and the deleted expense is gone from the totals.
- `GET /expenses/<id>/delete` any user, logged in or not: returns 405 (Method Not Allowed).
- `GET /profile` authenticated: each transaction row contains a delete form whose action is that row's own delete URL, has `data-confirm`, has no `style` or `onsubmit` attribute, and sits beside the Edit link.
- Browser-level check (manual, or an automated browser test if one is set up): clicking Delete shows the confirm dialog; accepting removes the expense and cancelling leaves it intact. Route tests cannot run `confirm()`.

## Definition of done

- [ ] `POST /expenses/<id>/delete` while logged out redirects to `/login`
- [ ] POSTing to `/expenses/<id>/delete` for a non-existent or other user's expense returns 404
- [ ] `GET /expenses/<id>/delete` returns 405
- [ ] Clicking "Delete" on a transaction row and confirming in the browser dialog removes that expense from the database
- [ ] After deletion, the user is redirected to `/profile` and the deleted expense no longer appears in the transaction list
- [ ] Cancelling the browser `confirm()` dialog does not submit the form and leaves the expense intact (verified in a browser)
- [ ] The confirmation comes from a `data-confirm` attribute and a listener in `main.js`, with no inline JavaScript or styles
- [ ] After deletion "Expense deleted." is shown and the active date filter is kept
- [ ] `delete_expense` returns the deleted-row count and closes its connection even on error
- [ ] The session cookie sets `SameSite=Lax` explicitly
- [ ] Tests run against a fresh temporary database
- [ ] Each transaction row in the profile table now shows both "Edit" and "Delete" actions
