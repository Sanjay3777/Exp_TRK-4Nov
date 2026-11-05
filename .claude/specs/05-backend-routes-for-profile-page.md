# Spec 05: Backend Routes for Profile Page

## Overview

- Step 05 replaces all hardcoded data in the `/profile` route with live queries against the SQLite database.
- The profile page currently renders a static demo user, fixed summary stats, a hand-typed transaction list and a hardcoded category breakdown.
- This step wires those four sections to real data, so every logged-in user sees their own expenses.
- The four query helpers (user, stats, transactions, category breakdown) are independent of each other. They all live in one file, so they should be written one after another, not edited in parallel, and then integrated into the single `/profile` route.

## Depends on

- Step 01: Database setup (tables and `get_db()` exist)
- Step 02: Registration (users are stored in the database)
- Step 03: Login and Logout (`session["user_id"]` is set on login; the `login_required` decorator exists)
- Step 04: Profile page static UI (the template already renders all four sections and reads the context keys `user`, `stats`, `expenses`, `categories`)
- Step 01: seed data (8 expenses totalling 346.24, Bills the top category) and the `SPENDLY_DB` override used by the tests

## Routes

- No new routes.
- The existing `GET /profile` route is modified in place.

## Database changes

- No database changes.
- The `users` and `expenses` tables already have all required columns: `user_id`, `amount`, `category`, `date`, `description`, `created_at`.

## Templates

- Modify `templates/profile.html`:
  - Amounts must be rendered with the ₹ symbol (Indian Rupee). Never £ or $.
  - All four dynamic sections are already present in the template, so no structural changes are needed. Only the Jinja variables they consume are now real.

## Files to change

- `app.py`: replace the hardcoded data in the `profile()` view with calls to the new query helpers in `database/queries.py`.
  - If `get_user_by_id` returns `None` (the user row no longer exists), clear the session and redirect to `url_for("login")`.
- `templates/profile.html`: confirm the ₹ symbol is used for all currency display.

## Files to create

- `tests/test_backend_connection.py`: the tests described below.
- `database/queries.py`: pure query helpers with no Flask imports, one function per data concern:
  - `get_user_by_id(user_id)`
    - Returns a dict with `name`, `email`, `member_since`, or `None` if no such user exists.
  - `get_summary_stats(user_id)`
    - Returns a dict with `total_spent`, `transaction_count`, `top_category`.
    - `total_spent` is rounded to 2 decimals.
    - If two categories tie for the highest total, the one that comes first alphabetically wins.
  - `get_recent_transactions(user_id, limit=10)`
    - Returns a list of dicts, each with `date`, `description`, `category`, `amount`.
    - Ordered newest-first by `date`, then by `id` descending as a tie-break for expenses on the same date.
    - `date` is returned as stored (`YYYY-MM-DD`) and displayed that way.
    - Only the latest `limit` rows are returned; the page should note when more exist, for example "Showing latest 10 of N".
  - `get_category_breakdown(user_id)`
    - Returns a list of dicts, each with `name`, `amount`, `pct`.
    - `amount` is rounded to 2 decimals.
    - `pct` is the percentage of the total, rounded to the nearest integer.
    - Ordered by `amount` descending, then by `name` ascending for ties.

## New dependencies

- None

## Rules for implementation

- No SQLAlchemy or ORMs. Raw `sqlite3` only, via `get_db()`.
- Parameterised queries only. Never string-format values into SQL.
- `PRAGMA foreign_keys = ON` must be enabled on every connection (already handled in `get_db()`).
- Currency must always display as ₹. Never £ or $.
- `member_since` must be derived from `users.created_at` and formatted as "Month YYYY" (for example "January 2026").
  - Use a fixed English list of month names, not the machine's locale, so the output is the same everywhere.
- `pct` values in the category breakdown must sum to 100.
  - Use integer rounding and adjust the largest category to absorb any rounding remainder.
- Every query is filtered by `user_id` (a parameter), so a user can only ever see their own data.
- Amounts are shown with exactly two decimals in the template (for example ₹346.24 and ₹0.00), including zero totals.
- If a user has no expenses, all helpers must return zeros and empty lists. Never raise exceptions.
- All helpers in `database/queries.py` must call `get_db()` internally and close the connection before returning.
- Use CSS variables. Never hardcode hex values.
- No inline styles.
- All templates extend `base.html`.

## Tests to write

- File: `tests/test_backend_connection.py`
- Tests must run against a temporary database by setting `SPENDLY_DB` (Step 01), never the real `spendly.db`.
- Money comparisons use 2-decimal rounding, not strict float equality.

### Unit tests

| Function | Input | Expected output |
|---|---|---|
| `get_user_by_id` | Valid `user_id` | Dict with correct `name`, `email`, `member_since` |
| `get_user_by_id` | Non-existent id | `None` |
| `get_summary_stats` | `user_id` with expenses | Correct `total_spent`, `transaction_count`, `top_category` |
| `get_summary_stats` | `user_id` with no expenses | `{"total_spent": 0, "transaction_count": 0, "top_category": "—"}` |
| `get_recent_transactions` | `user_id` with expenses | List ordered newest-first, each item has `date`, `description`, `category`, `amount` |
| `get_recent_transactions` | `user_id` with no expenses | Empty list |
| `get_category_breakdown` | `user_id` with expenses | List ordered by `amount` descending; `pct` values are integers summing to 100 |
| `get_category_breakdown` | `user_id` with no expenses | Empty list |
| `get_category_breakdown` | `user_id` with three equal categories | `pct` values are 33, 33 and 34 (largest absorbs the remainder) and sum to 100 |
| `get_recent_transactions` | Two expenses on the same date | Higher `id` comes first |
| `get_summary_stats` | Two categories with equal totals | Alphabetically first category is `top_category` |
| All four helpers | Any valid call | The connection is closed afterwards (for example, patch `get_db` and assert `close()` was called) |

### Route tests

- `GET /profile` unauthenticated: redirects to `/login` (302).
- `GET /profile` authenticated as the seed user:
  - Returns 200.
  - Response contains "Demo User".
  - Response contains "demo@spendly.com".
  - Response contains the ₹ symbol.
  - `total_spent` matches the sum of all seed expenses (346.24).
  - `transaction_count` is 8.
  - `top_category` is "Bills" (highest single-category total).
  - The transaction list appears in newest-first order.
  - The category breakdown contains all 7 categories.
- `GET /profile` authenticated as a newly registered user with a unique name and email:
  - The response contains that name and email, not "Demo User".
  - Total spent shows ₹0.00, the transaction count is 0, and the breakdown is empty.
- Isolation: with two users who each have expenses, the profile of one shows none of the other's descriptions, and its totals cover only its own rows.
- `GET /profile` with a session `user_id` that has no matching row: redirects to `/login` and clears the session.

## Definition of done

- [ ] Logging in as `demo@spendly.com` / `demo123` shows "Demo User" and "demo@spendly.com" on the profile page, not the hardcoded strings
- [ ] Total spent displayed on the profile page equals ₹346.24
- [ ] Transaction count displayed is 8
- [ ] Top category displayed is "Bills"
- [ ] Transaction list shows 8 rows ordered newest date first
- [ ] Category breakdown shows 7 categories with percentages that add up to 100%
- [ ] All amounts on the page display the ₹ symbol
- [ ] A second user's profile never shows another user's expenses
- [ ] Totals are rounded to 2 decimals and displayed as for example ₹346.24
- [ ] A session pointing at a deleted user redirects to `/login` instead of crashing
- [ ] Tests run against a temporary database via `SPENDLY_DB`
- [ ] Registering a brand-new user and visiting `/profile` shows ₹0.00 total spent, 0 transactions and an empty category breakdown, with no errors
