# Spec 06: Date Filter on Profile

## Overview

- Step 06 adds a date-range filter to the existing `/profile` route so users can narrow the transaction list, summary stats and category breakdown to a specific period.
- The filter is driven entirely by query-string parameters (`date_from` and `date_to`) on `GET /profile`. No new routes are required.
- A compact filter bar lets users pick any range:
  - Four quick-select preset buttons: "This Month", "Last 3 Months", "Last 6 Months", "All Time".
  - Two `<input type="date">` fields for a custom range.
- All three data sections (summary stats, recent transactions, category breakdown) must respect the active date filter.

## Depends on

- Step 01: Database setup (the `expenses` table with its `date` column must exist)
- Step 04: Profile page UI (the template with all four sections must exist)
- Step 05: Backend connection (`get_summary_stats`, `get_recent_transactions` and `get_category_breakdown` in `database/queries.py` must exist and be wired to the `/profile` route)

## Routes

- No new routes.
- The existing `GET /profile` route is modified to read two optional query parameters:
  - `date_from`: ISO date string `YYYY-MM-DD`, inclusive lower bound.
  - `date_to`: ISO date string `YYYY-MM-DD`, inclusive upper bound.
- Each parameter is validated on its own. A parameter that is absent, empty or malformed is ignored; the other one still applies.
  - Only `date_from` given: show expenses on or after that date.
  - Only `date_to` given: show expenses on or before that date.
  - Neither valid: unfiltered "All Time" view.
- The route never errors out.
- `date_from` equal to `date_to` is allowed and shows a single day.

## Database changes

- No database changes.
- The `expenses.date` column (TEXT, `YYYY-MM-DD`) already supports `BETWEEN` comparison in SQLite.

## Templates

- Modify `templates/profile.html`:
  - Add a filter bar section above the summary stats row, containing:
    - Four quick-select preset buttons: "This Month", "Last 3 Months", "Last 6 Months", "All Time".
      - Preset ranges (all end today, using the server's date):
        - "This Month": the first day of the current month to today.
        - "Last 3 Months": the same day of the month 3 months ago (or that month's last day if it has no such day, so 31 May gives 28 or 29 February, never an invalid date) to today.
        - "Last 6 Months": the same rule, 6 months back, to today.
        - "All Time": no range.
      - Each is a link to `/profile` with the appropriate `date_from` and `date_to` query params computed in `app.py`.
      - "All Time" passes no query params (a clean `/profile` URL).
    - A custom range sub-form (`method="get"`) with two labelled `<input type="date">` fields (`date_from`, `date_to`) and an "Apply" submit button.
      - The fields are pre-filled with the currently applied dates.
    - The currently active preset or custom range must be visually highlighted with an active state on the button. The active preset also carries `aria-current="page"`.
      - A preset is active when the applied range equals its range exactly. "All Time" is active only when no range is applied. Any other range counts as custom.
      - After a rejected (reversed) range, the applied range is empty, so "All Time" is active.
    - A line showing the period in effect, for example "Showing 01 Mar 2026 to 31 Mar 2026", "Showing from 01 Mar 2026", "Showing up to 31 Mar 2026" or "Showing all time".
  - The flashed error "Start date must not be after end date." must be displayed on the page.
  - No structural changes to any existing section. Only the Jinja variables fed into them change when a filter is active.

## Files to change

- `app.py`
  - In the `profile()` view, read `date_from` and `date_to` from `request.args`.
  - Validate each value with `datetime.strptime(value, "%Y-%m-%d")`. On `ValueError`, treat that value as absent. After parsing, re-format with `strftime("%Y-%m-%d")` and use the re-formatted value for the query and the template, so unpadded input such as `2026-3-5` becomes `2026-03-05`.
  - If both are present and `date_from` is later than `date_to`, treat both as absent and flash "Start date must not be after end date."
  - Compute preset date ranges in a small function, for example `get_preset_ranges(today)`, that takes `today` as an argument (the route passes `date.today()`). The template never calculates dates.
  - The preset data passed to the template is a list of dicts with `label`, `date_from`, `date_to` and `active`.
  - Pass the validated `date_from`, `date_to` and the preset URL params back to the template so the filter bar can reflect the active state.
  - Pass `date_from` and `date_to` to each of the three query helpers.
- `database/queries.py`
  - `get_summary_stats(user_id, date_from=None, date_to=None)`
    - When both params are provided, add `AND date BETWEEN ? AND ?` to the expenses queries.
    - When only `date_from` is provided, add `AND date >= ?`. When only `date_to` is provided, add `AND date <= ?`.
    - The `user_id` condition always stays.
    - When absent, behave identically to Step 05 (unfiltered).
  - `get_recent_transactions(user_id, limit=10, date_from=None, date_to=None)`
    - Same pattern. Newest-first ordering and the limit remain unchanged.
  - `get_category_breakdown(user_id, date_from=None, date_to=None)`
    - Same pattern. The percentage recalculation logic remains unchanged.
- `templates/profile.html`: add the filter bar (see Templates above).
- `static/css/profile.css`: add styles for the filter bar and the active-preset button state, using CSS variables only. Step 04 creates this file; `profile.html` must link it with `url_for('static', filename='css/profile.css')`.

## Files to create

- `tests/test_date_filter.py`: tests described under "Tests to write".

## New dependencies

- None

## Rules for implementation

- No SQLAlchemy or ORMs. Raw `sqlite3` only, via `get_db()`.
- Parameterised queries only. Never string-format dates into SQL; use `?` placeholders even for date-range bounds.
- Date validation in `app.py`: use `datetime.strptime(value, "%Y-%m-%d")`, then re-format to `YYYY-MM-DD`. On `ValueError`, treat the param as absent.
- Empty strings (for example from submitting the blank custom form) count as absent.
- Preset links must be generated with `url_for("profile", date_from=..., date_to=...)`. Never use hardcoded URL strings in the template.
- When `date_from` and `date_to` are both absent, all three query helpers must behave identically to their Step 05 behaviour (unfiltered).
- If `date_from` is later than `date_to` after validation, treat both as absent and flash: "Start date must not be after end date."
- The "All Time" preset must pass no query params (a clean `/profile` URL).
- Preset date calculations must be computed in `app.py`, not in the template.
- Use CSS variables. Never hardcode hex values.
- No inline styles.
- All templates extend `base.html`.
- Use `url_for()` for every internal link. Never hardcode paths.

## Tests to write

- File: `tests/test_date_filter.py`. Tests use a temporary database via `SPENDLY_DB`.
- Preset function (called with a fixed `today`):
  - Mid-month date: "This Month" starts on the 1st.
  - 31 May: "Last 3 Months" starts 28 February (or 29 in a leap year).
  - 15 January: "Last 3 Months" starts 15 October of the previous year (year rollover).
  - 29 February in a leap year: "Last 6 Months" starts on the last valid day of that month.
- Filter behaviour:
  - Inclusive bounds: expenses dated exactly on `date_from` and `date_to` are included.
  - Only `date_from`, and only `date_to`, filter correctly.
  - `date_from` equal to `date_to` shows one day.
  - Reversed range: flash "Start date must not be after end date." and unfiltered data.
  - Malformed value (`not-a-date`) and empty strings: no error, ignored.
  - Unpadded `2026-3-5` is normalised and returns the same rows as `2026-03-05`.
  - The filtered stats, transactions and breakdown all agree with each other.
- Isolation: with two users, a filtered profile still shows only the signed-in user's expenses.

## Definition of done

- [ ] Visiting `/profile` with no query params returns the same data as Step 05 (unfiltered, all expenses)
- [ ] Clicking "This Month" filters all three sections to the current calendar month only
- [ ] Clicking "Last 3 Months" filters to expenses in the 3-month window ending today
- [ ] Clicking "Last 6 Months" filters to expenses in the 6-month window ending today
- [ ] Clicking "All Time" removes any active filter and shows all expenses
- [ ] Submitting a custom date range with valid `date_from` and `date_to` shows only expenses within that range in all three sections
- [ ] Submitting a range where `date_from` is later than `date_to` shows a flash error and falls back to the unfiltered view
- [ ] Submitting a malformed date string (for example `date_from=not-a-date`) does not crash the app and silently falls back to the unfiltered view
- [ ] The active preset button or custom-range fields visually indicate which filter is currently applied
- [ ] A range with only `date_from` or only `date_to` filters on that bound alone
- [ ] A single-day range (`date_from` equal to `date_to`) is accepted
- [ ] Unpadded dates are normalised to `YYYY-MM-DD` before use
- [ ] The custom fields are pre-filled and a "Showing ..." line states the period in effect
- [ ] Preset calculations live in a function that takes `today`, and `tests/test_date_filter.py` passes
- [ ] All amounts continue to display the ₹ symbol regardless of the active filter
- [ ] A user with no expenses in the selected range sees ₹0.00 total spent, 0 transactions and an empty category breakdown, with no errors
