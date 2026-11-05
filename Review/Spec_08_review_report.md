# Spec Review Report: Step 08 Edit Expense

**Date:** 2026-11-04
**Source of truth (the prompt):** `Spec_08-edit-expense_prompt.md`
**Document under review:** `.claude/specs/08-edit-expense.md`

## Verdict

**The spec conforms to the prompt, with one deliberate change.** All 12 required sections are present, in order, with the exact names. Every requirement is carried over: both routes, both new helpers, the `id` change to `get_recent_transactions`, the four validation rules, the 5-row unit test table, the 13 route tests and all 10 Definition of done items.

The one change replaces hardcoded paths in the prompt with `url_for`, which the prompt's own rules require. The review found gaps in the prompt that the spec reproduced. The most serious: nothing says how a POST to someone else's expense produces a 404, and expenses outside the 10 newest rows can't be edited at all.

## Format rules from the prompt

| Rule | Result | Notes |
|---|---|---|
| Output only the Markdown spec | Pass | The only addition is a `# Spec 08: Edit Expense` title line |
| No code, no prose outside the spec sections | Pass, with a note | One inline decorator line is kept because the prompt contains it (see below) |
| Bullet lists throughout | Pass | |
| Table for the unit test cases | Pass | 5 rows with Function / Input / Expected output |
| Checkbox list for Definition of Done | Pass | 10 checkbox items |
| Exact section names, no numbers | Pass | Same names and order |

## Section-by-section comparison

| Section | Result | Notes |
|---|---|---|
| Overview | Match | All six points |
| Depends on | Match | Steps 01, 03, 05, 07 |
| Routes | Match | `GET` and `POST /expenses/<int:id>/edit`, logged-in only |
| Database changes | Match | No changes; same column list |
| Templates | Match, with one change | `edit_expense.html` with pre-filled fields and the error block; `profile.html` Actions column and Edit link |
| Files to change | Match | `queries.py` (both helpers and the `id` change), `app.py`, `profile.html` |
| Files to create | Match | `templates/edit_expense.html` |
| New dependencies | Match | None |
| Rules for implementation | Match | All 13 rules present, including both ownership guards |
| Tests to write | Match | `tests/test_edit_expense.py`, all 5 unit tests, all 13 route tests |
| Definition of done | Match | All 10 items present, lightly reworded |

## Differences from the prompt

1. **Hardcoded paths replaced with `url_for`.** The prompt specifies `action="/expenses/{{ expense.id }}/edit"` and an Edit link to `/expenses/{{ tx.id }}/edit`, plus a cancel link to `/profile`, while also saying "never hardcode paths". The spec uses `url_for('edit_expense', id=...)` and `url_for('profile')`. Correct resolution.
2. A title heading, and `Unit tests` and `Route tests` sub-headings.

**Note on the "no code" rule:** the prompt itself contains the decorator line `@app.route(..., methods=["GET", "POST"])`, and the spec keeps it as an inline snippet. It is the only code-like line in the spec, so the rule is followed in spirit.

## Issues inherited from the prompt

### Missing or unclear behaviour

1. **The 404 for a POST isn't specified.**
   - The route tests require a POST to another user's expense to return 404.
   - The prompt says only GET calls `get_expense_by_id`. The POST bullet says "validate, call `update_expense`", and `update_expense` is defined to do nothing (0 rows, no error) for the wrong user.
   - *Suggestion:* say that POST also loads the expense first and aborts with 404, or have `update_expense` return the number of rows changed and have the route check it.
2. **No order between validation and ownership.** A POST with bad data to someone else's expense should return 404, not a form with errors. No test covers it.
3. **Stale or deleted expense.** If an expense is deleted after the form opened, a POST silently updates nothing and still redirects as though it worked.
4. **Many expenses can't be edited.**
   - The Edit link lives in the profile table, which shows only the 10 newest rows (Spec 05).
   - Anything older, or anything outside the active date filter, has no Edit link, and there is no page listing all expenses.
5. **The updated row may vanish from the list.** After saving, the profile page shows the newest 10. Editing a date to an old one makes the expense drop out, which can fail "the updated values appear in the transaction list".
6. **Redirect loses context.** Saving or cancelling returns to the plain `/profile`, dropping any active date filter.
7. **The categories list has no source.**
   - The prompt passes a "categories list" from `app.py`, but Spec 07 hardcodes the 7 options inside `add_expense.html`.
   - *Suggestion:* one shared constant used by both forms and the validator.

### Weaknesses carried over from Step 07

8. **Validation is "identical to Step 07", so every gap in Spec 07 repeats here.** The Spec 07 report covers them: `inf`, `nan` and sub-cent amounts accepted, unpadded dates accepted, and the 200-character limit not enforced on the server.
9. **Duplicated validation.** Two copies of the same four rules will drift apart. *Suggestion:* a shared validation function.
10. **Date and amount pre-fill can fail.**
    - `<input type="date">` shows nothing if the stored value is not padded `YYYY-MM-DD`, which Spec 07's gap allows.
    - Floating-point amounts can print as `346.24000000000007`. Format to 2 decimals.
11. **A category that is no longer in the list** is silently replaced by the first option when the form opens, and saved that way.

### Test gaps

12. **No test for the Edit link on the profile page.** The last Definition of done item has no test, and neither does the "Actions" column.
13. **"Submitted values retained" isn't tested.** The route tests check only for an error message, the same gap as Spec 07.
14. **Other missing tests:**
    - A forged `user_id` form field must not change the owner.
    - `update_expense` with `description=None`.
    - Editing one expense leaves others unchanged.
    - "Other user's expense" needs a second real user to be a meaningful test.
15. **Shared test database.** Tests that change rows will break other test files if the database is shared (see the Spec 07 report, issue 1).
16. **The `id` change in `get_recent_transactions`** can break an earlier test that checks the exact keys of each row (flagged when the spec was written).

### Smaller points

17. No CSRF protection on the form, and no "last updated" column, so edits leave no trace.
18. Two people editing the same expense get "last save wins" with no warning.
19. A success message after saving isn't specified (same gap as Spec 07).
20. The route parameter is named `id`, which shadows Python's built-in.
21. The prompt's file tree is out of date.

## Recommended actions

1. Specify how POST returns 404 and what `update_expense` returns (issues 1 to 3).
2. Decide how users reach expenses beyond the newest 10 (issue 4), and where to land after saving (issues 5 and 6).
3. Share one category list and one validator with Step 07 (issues 7 and 9), and fix the Step 07 validation gaps once, in one place (issue 8).
4. Add tests for issues 12 to 14.
5. The rest can wait.
