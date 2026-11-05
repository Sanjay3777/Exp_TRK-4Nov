# Spec Review Report: Step 09 Delete Expense

**Date:** 2026-11-04
**Source of truth (the prompt):** `Spec_09-delete-expense_prompt.md`
**Document under review:** `.claude/specs/09-delete-expense.md`

## Verdict

**The spec conforms to the prompt, with two deliberate changes.** All 12 required sections are present, in order, with the exact names. Every requirement is carried over: the POST-only route with 405 on GET, `delete_expense` with its ownership guard, the delete form with its two permitted inline attributes, the `.btn-delete` style, all 13 implementation rules, the 3-row unit test table, the 5 route tests and all 7 Definition of done items.

The two changes fix places where the prompt contradicts itself. The review also found gaps the spec reproduced. The most important: the confirmation step disappears if JavaScript is off, and several Definition of done items can't be tested automatically.

## Format rules from the prompt

| Rule | Result | Notes |
|---|---|---|
| Output only the Markdown spec | Pass | The only addition is a `# Spec 09: Delete Expense` title line |
| No code, no prose outside the spec sections | Pass, with a note | HTML attributes and the decorator line appear inline because the prompt requires them verbatim |
| Bullet lists throughout | Pass | |
| Table for the unit test cases | Pass | 3 rows with Function / Input / Expected output |
| Checkbox list for Definition of Done | Pass | 7 checkbox items |
| Exact section names, no numbers | Pass | Same names and order |

## Section-by-section comparison

| Section | Result | Notes |
|---|---|---|
| Overview | Match, with one change | All six points (see below) |
| Depends on | Match | Steps 01, 03, 05, 08 |
| Routes | Match | POST-only, logged-in only, GET returns 405 |
| Database changes | Match | No changes |
| Templates | Match, with one change | Delete form with `display:inline`, `onsubmit` confirm, `btn-delete` button; Edit link stays |
| Files to change | Match | `queries.py`, `app.py` (with the exact decorator), `profile.html`, `style.css` |
| Files to create | Match | None |
| New dependencies | Match | None |
| Rules for implementation | Match | All 13 rules present |
| Tests to write | Match | `tests/test_delete_expense.py`, 3 unit tests, 5 route tests |
| Definition of done | Match | All 7 items present, lightly reworded |

## Differences from the prompt

1. **The confirm handler is described consistently.** The prompt's Overview says the `confirm()` dialog is in "the button's onclick handler", but its Templates section puts it in `onsubmit` on the form. The spec uses `onsubmit` in both places. Correct, because an `onsubmit` check also covers Enter-key submission.
2. **Form action uses `url_for('delete_expense', id=tx.id)`** instead of the hardcoded `/expenses/{{ tx.id }}/delete`, to follow the prompt's "never hardcode paths" rule.
3. A title heading, and `Unit tests` and `Route tests` sub-headings.

## Issues inherited from the prompt

### Security and safety

1. **Confirmation disappears without JavaScript.** The confirm dialog is the only guard against accidental deletion. With scripts disabled or blocked, the form submits immediately and the expense is gone.
2. **No CSRF protection on a destructive action.**
   - A page on another site could submit a hidden form to `/expenses/<id>/delete`, and ids are small sequential numbers.
   - Flask doesn't set a `SameSite` attribute on the session cookie by default. Current browsers treat that as Lax, which blocks most cross-site POSTs, but this is not guaranteed everywhere.
   - *Suggestion:* add a CSRF token to every form, or at least set `SESSION_COOKIE_SAMESITE` explicitly.
3. **The inline handler is fragile.** The confirm message is a static string today and is safe. If someone later adds the description or amount to the message, HTML escaping alone will not make it safe inside a JavaScript string, which would open a script-injection hole. The inline attribute also rules out a strict Content-Security-Policy. *Suggestion:* use a data attribute and a listener in `main.js`.
4. **Deleting is permanent with no undo and no confirmation message.** "Permanently" is stated, but there is no "Expense deleted" message afterward (the same gap as Specs 07 and 08).

### Unclear behaviour

5. **Redundant lookup.** `delete_expense` is already scoped to the owner, yet the route calls `get_expense_by_id` first just to decide on a 404. That costs an extra query and leaves a small gap between the check and the delete. Returning the number of rows deleted would do both jobs. The prompt also doesn't say what `delete_expense` returns.
6. **Connection cleanup.** The helper "commits and closes before returning". It doesn't say what happens on an error, so a failed delete could leave the connection open. A `try/finally` is the usual fix.
7. **The redirect loses the date filter.** After deleting, the user lands on plain `/profile`, even if they were viewing a filtered range.
8. **GET returns 405 for everyone, including logged-out users.** The route test says "any user", so this is intended, but it differs from the other routes, which redirect logged-out users to `/login`.

### Test gaps

9. **Two Definition of done items can't be checked by the listed tests.** "Confirming in the browser dialog removes the expense" and "cancelling leaves it intact" need a real browser. Route tests can't run `confirm()`.
10. **The profile page isn't tested.** Nothing checks that each row shows a Delete form with the right action, which is the last Definition of done item.
11. **Missing safety tests:**
    - Deleting one expense leaves the user's other expenses and other users' expenses unchanged.
    - An unauthenticated POST does not delete anything.
    - A non-numeric id returns 404 through routing.
12. **Shared test database.** The delete tests remove rows that earlier tests (Spec 05's 8 seed expenses) depend on. See the Spec 07 report, issue 1.

### Smaller points

13. **A danger colour already exists.** `style.css` already defines `--danger` and `--danger-light`, so no new variable is needed. The prompt says "e.g. a red-toned CSS variable".
14. **Button styling.** A native button needs its default border, font and background reset to match the Edit link beside it. Only `.btn-delete` is specified.
15. **Accessibility.** Every row has the same "Delete" label, so screen readers can't tell the buttons apart. An accessible label naming the expense would fix it.
16. **Where the style goes.** `.btn-delete` is added to `style.css`, while the rest of the profile table styling lives in `profile.css`, which no spec loads (see the Spec 06 report).
17. The prompt's file tree is out of date.

## Recommended actions

1. Decide how to handle CSRF and the no-JavaScript case (issues 1 and 2) before deployment.
2. Make `delete_expense` return the deleted-row count and drop the extra lookup, with cleanup in `try/finally` (issues 5 and 6).
3. Add the missing tests, and plan one browser-level test for the confirm dialog (issues 9 to 11).
4. Use the existing `--danger` variable (issue 13). The rest can wait.
