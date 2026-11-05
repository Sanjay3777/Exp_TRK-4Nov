# Spec Review Report: Step 07 Add Expense

**Date:** 2026-11-04
**Source of truth (the prompt):** `Spec_07-add-expense_prompt.md`
**Document under review:** `.claude/specs/07-add-expense.md`

## Verdict

**The spec conforms to the prompt, with one deliberate change.** All 12 required sections are present, in order, with the exact names. Every requirement is carried over: the form fields, the four validation rules, the helper signature, the two template changes, the 2-row unit test table, the 10 route tests and all 10 Definition of done items.

The one change replaces three hardcoded paths in the prompt with `url_for` calls, which the prompt's own rules require. The review also found real gaps in the prompt that the spec reproduced. The most serious: the tests write to the same database that Spec 05's tests expect to be untouched, and `float()` accepts values like `inf` that would corrupt the totals.

## Format rules from the prompt

| Rule | Result | Notes |
|---|---|---|
| Output only the Markdown spec | Pass | The only addition is a `# Spec 07: Add Expense` title line |
| No code, no prose outside the spec sections | Pass | Only inline identifiers appear |
| Bullet lists throughout | Pass | |
| Table for the unit test cases | Pass | 2 rows with Function / Input / Expected output |
| Checkbox list for Definition of Done | Pass | 10 checkbox items |
| Exact section names, no numbers | Pass | Same names and order |

## Section-by-section comparison

| Section | Result | Notes |
|---|---|---|
| Overview | Match | All five points |
| Depends on | Match | Steps 01, 03, and 04/05 (written as "Steps 04 and 05") |
| Routes | Match | `GET` and `POST /expenses/add`, logged-in only |
| Database changes | Match | No changes; same column list |
| Templates | Match, with one change | `add_expense.html` with the four fields, buttons and error block; `profile.html` and `base.html` changes |
| Files to change | Match | `app.py`, `queries.py`, `profile.html`, `base.html` |
| Files to create | Match | `templates/add_expense.html` |
| New dependencies | Match | None |
| Rules for implementation | Match | All 13 rules present, including the 4 validation rules |
| Tests to write | Match | `tests/test_add_expense.py`, both unit tests, all 10 route tests |
| Definition of done | Match | All 10 items present, lightly reworded |

## Differences from the prompt

1. **Hardcoded paths replaced with `url_for`.** The prompt says `action="/expenses/add"` and a cancel link to `/profile`, and also says "never hardcode paths". The spec uses `url_for('add_expense')` and `url_for('profile')`. Correct resolution.
2. The profile "Add Expense" link is also marked "(via `url_for`)".
3. A title heading, and `Unit tests` and `Route tests` sub-headings.

## Issues inherited from the prompt

### Can cause wrong data or failing tests

1. **The tests would change the database that Spec 05 relies on.**
   - Spec 05's tests expect exactly 8 expenses for the seed user, totalling 346.24.
   - Spec 07's tests insert new rows for the "test user" into the same `spendly.db`.
   - Run in the wrong order, or twice, the Spec 05 tests fail.
   - *Suggestion:* give every test a fresh temporary database (see the Spec 01 report, issue 4).
2. **`float()` accepts values that corrupt the data.**
   - `float("inf")` passes a "greater than 0" check, and `float("nan")` can pass depending on how the check is written. `float("1_000")` also parses as 1000.0.
   - An infinite amount makes every total infinite.
   - *Suggestion:* require a finite number, add an upper limit, and round to 2 decimals.
3. **Precision doesn't match the form.** The input has `step="0.01"` and `min="0.01"`, but the server accepts `0.001`, which displays as ₹0.00. Browser limits can be bypassed.
4. **Unpadded dates pass the date check.** The same `strptime` weakness as in the Spec 06 report: `2026-3-5` is accepted and stored as text, which breaks sorting and the date filter. Re-format the parsed date before saving.
5. **The 200-character limit isn't enforced on the server.** It only exists as the form's `maxlength`, and no test covers it.

### Test coverage gaps

6. **The saved row's owner isn't tested.** Nothing checks that `user_id` comes from the session. A forged `user_id` form field should be ignored.
7. **"Previously entered values retained" is never tested.** The Definition of done requires it, but the route tests check only that an error message appears. That includes the selected category and the date.
8. **Other tests missing:**
   - Negative amount.
   - Whitespace-only description becoming NULL.
   - An unauthenticated POST must not insert a row.
   - The new expense appearing on the profile page after the redirect.
   - A foreign key failure for an unknown `user_id` (Spec 01 requires one).
9. **"The new expense appears in the transaction list" can stop being true.** The profile table shows only the 10 newest rows, so an old-dated expense disappears once the user has 10 or more newer ones.

### Ambiguities and missing pieces

10. **No success feedback.** After saving, the user lands on the profile page with no "Expense added" message, and the profile page has no place to show one.
11. **Error reporting is undefined.** With several problems, does the page show the first error or all of them? Rule order implies the first, but it isn't stated.
12. **Category list has no single source.** Seven names are repeated in the add form, the validator, and (later) the edit form, with no shared constant and no database check.
13. **Edge dates:** nothing says whether future dates are allowed, or sets a lower limit.
14. **`insert_expense` return value isn't stated.** Its sibling `create_user` returns the new id.
15. **The amount label doesn't mention ₹.** The currency rule is listed, but the form is the one place a user types an amount.
16. **No CSRF protection** on the form.
17. **Status code on failure is 200.** Technically 400 is more correct for a rejected form, but the tests require 200.

## Recommended actions

1. Isolate the test database (issue 1), because it can silently break Step 05's tests.
2. Tighten amount and date validation (issues 2 to 4, and 13).
3. Add tests for issues 6, 7 and 8.
4. Add a success flash and a shared category list (issues 10 and 12).
5. The rest can wait.
