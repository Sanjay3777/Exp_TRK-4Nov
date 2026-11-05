# Spec Review Report: Step 01 Database Setup

**Date:** 2026-11-04
**Source of truth (the prompt):** `Spec_01_database_setup_prompt.md`
**Document under review:** `.claude/specs/01-database-setup.md`

## Verdict

**The spec conforms to the prompt.** All 14 required sections are present, correctly numbered and named, and every requirement in the prompt is covered. There are no missing requirements and no contradictions.

The weaknesses are about what the prompt itself leaves out. Several later specs (02 to 09) depend on details that neither the prompt nor the spec defines. These are listed under "Gaps that affect later steps".

## Format rules from the prompt

| Rule | Result | Notes |
|---|---|---|
| Output only the Markdown spec | Pass | The only addition is a `# Spec 01: Database Setup` title line |
| No code, no prose outside the spec sections | Pass | Only inline identifiers (function and column names) appear |
| Use tables for schema definitions | Pass | Both tables use a Column / SQLite Type / Constraints layout |
| Use checkbox lists for Definition of Done | Pass | Section 14 is a checkbox list |
| Number all sections exactly as listed | Pass | Sections 1 to 14, in order, with the same names |

## Section-by-section comparison

| # | Section | Result | Notes |
|---|---|---|---|
| 1 | Overview | Match | Foundational role stated; auth, profile and expense features named as dependants |
| 2 | Depends on | Match | States "None", first step, no prerequisites |
| 3 | Routes | Match | No new routes; placeholders unchanged |
| 4 | Database Schema | Match | All columns of `users` and `expenses` present with type and constraints; `YYYY-MM-DD` noted for `expenses.date`. Spec adds a Notes block explaining why |
| 5 | Functions to Implement | Match | `get_db()`, `init_db()`, `seed_db()` all specified as in the prompt, including demo user details, 8 expenses, 7 categories, dates within the current month |
| 6 | Changes to app.py | Match | Imports, `app.app_context()` startup call, and "ready before any route" all present |
| 7 | Files to Change | Match | `database/db.py` and `app.py` only |
| 8 | Files to Create | Match | None |
| 9 | Dependencies | Match | No new packages; `sqlite3` and `werkzeug.security` |
| 10 | Categories | Match | All 7 listed as bullets, marked as the only valid values |
| 11 | Rules for Implementation | Match | All 7 rules present |
| 12 | Expected Behavior | Match | Function behaviour and database-level enforcement both covered |
| 13 | Error Handling | Match | Duplicate email, invalid `user_id`, and clear errors all covered |
| 14 | Definition of Done | Match | All 8 items present, only lightly reworded |

## Additions beyond the prompt

These do not contradict the prompt, but they are not in it:

- A title heading at the top of the file.
- A Notes block under the schema (string comparison for dates; each expense belongs to one user).
- Sub-headings per function in Sections 5 and 12.
- "Errors must not be silently swallowed" in Section 13.
- "Default timestamps are filled in automatically" in Section 12.

## Gaps that affect later steps

These are not spec errors, because the prompt did not ask for them. They are things the later specs assume.

1. **Seed amounts are undefined.**
   - Spec 05's tests and Definition of done expect a total of 346.24, 8 transactions and "Bills" as the top category.
   - Spec 01 gives no amounts, so a correct Step 01 implementation can still fail those tests.
   - *Suggestion:* add a sample-data table (date, category, amount, description) whose amounts sum to 346.24, with Bills the largest category total.
2. **`created_at` format is not defined.**
   - Spec 05 needs `member_since` as "Month YYYY", parsed from `users.created_at`.
   - Spec 01 says only "current timestamp".
   - *Suggestion:* state the format (SQLite's default is `YYYY-MM-DD HH:MM:SS`).
3. **Seed dates depend on the day the app first runs.**
   - "Spread across the current month" can put expenses on dates after today.
   - Spec 05's newest-first checks and Spec 06's date-filter tests then depend on when the seed ran.
   - *Suggestion:* require seed dates on or before today, or fix them relative to the first of the month.
4. **The database path can't be overridden.**
   - Spec 01 hard-codes `spendly.db` in the project root.
   - Specs 05 to 09 add tests that insert and delete rows, so they would run against the real database.
   - *Suggestion:* allow the path to be overridden, for example by an environment variable or an app config value.
5. **The category list is not enforced in the database.**
   - Section 10 says only 7 values are valid, but Section 12 lists no category check, so the database accepts any text.
   - Later steps validate in Python, so this is only a risk for direct inserts.
   - *Suggestion:* decide whether a database-level check is wanted.
6. **`spendly.db` is not in `.gitignore`.**
   - The current `.gitignore` lists `expense_tracker.db`.
   - Section 7 allows changes to only `db.py` and `app.py`, so the spec can't fix this.
   - *Suggestion:* add `.gitignore` to the files-to-change list, or add `spendly.db` to it separately.
7. **The seed is not atomic.** If the demo user is inserted and the expense inserts then fail, the next run skips seeding because a user exists. *Suggestion:* require the user and all 8 expenses to be inserted in a single transaction.
8. **Deleting users is unspecified.** There is no delete behaviour for expenses when a user is removed. The Terms and Privacy pages promise account deletion. *Suggestion:* state whether the foreign key cascades.

## Notes on the prompt itself

- Its project tree is out of date. It lacks `terms.html`, `privacy.html` and the `Spec_*` files that now exist.
- Section 6 asks for `get_db` to be imported into `app.py`, but nothing in `app.py` uses it at this step, so it will be an unused import.

## Recommended actions

1. Fix gaps 1 to 3 in Spec 01, since Spec 05 depends on them.
2. Decide on gaps 4 and 6 before writing the Step 05 tests.
3. Gaps 5, 7 and 8 can wait, but gap 7 is a cheap fix.
