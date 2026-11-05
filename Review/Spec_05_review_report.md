# Spec Review Report: Step 05 Backend Routes for Profile Page

**Date:** 2026-11-04
**Source of truth (the prompt):** `Spec_05-backend-routes-for-profile-page_prompt.md`
**Document under review:** `.claude/specs/05-backend-routes-for-profile-page.md`

## Verdict

**The spec conforms to the prompt.** All 11 required sections are present, in order, with the exact names. Every requirement is carried over: the four helper functions and their return shapes, all 11 implementation rules, the 8-row unit test table, the route tests and all 8 Definition of done items. The spec adds nothing that changes the meaning.

The review found problems in the prompt that the spec faithfully reproduced. The two most important are that a money-rounding detail will probably make the 346.24 test fail, and that no test checks one user cannot see another user's expenses.

## Format rules from the prompt

| Rule | Result | Notes |
|---|---|---|
| Output only the Markdown spec | Pass | The only addition is a `# Spec 05: ...` title line |
| No code, no prose outside the spec sections | Pass | Only inline identifiers and the dict literal for the empty-stats case |
| Bullet lists throughout | Pass | |
| Table for the unit test cases | Pass | 8 rows with Function / Input / Expected output |
| Checkbox list for Definition of Done | Pass | 8 checkbox items |
| Exact section names, no numbers | Pass | Same names and order |

## Section-by-section comparison

| Section | Result | Notes |
|---|---|---|
| Overview | Match | All four points, including the three-subagent sentence |
| Depends on | Match | Steps 01 to 04 with the same explanations |
| Routes | Match | No new routes; `GET /profile` modified in place |
| Database changes | Match | No changes; same column list |
| Templates | Match | ₹ rule and "no structural changes" |
| Files to change | Match | `app.py` and `profile.html` |
| Files to create | Match | `database/queries.py` with all four helpers, return shapes, ordering and `pct` rule |
| New dependencies | Match | None |
| Rules for implementation | Match | All 11 rules present |
| Tests to write | Match | Test file name, 8-row table, all route tests; sub-headings "Unit tests" and "Route tests" added |
| Definition of done | Match | All 8 items present, lightly reworded |

## Differences from the prompt

1. A title heading at the top of the file.
2. Sub-headings (`Unit tests`, `Route tests`) under Tests to write. The prompt only labels them in text.

## Issues inherited from the prompt

### Likely to break the tests

1. **Money totals may not equal 346.24 exactly.**
   - Amounts are stored as REAL (Spec 01), and adding floats can give `346.24000000000007`.
   - The prompt has no rounding rule, so a strict equality test on `total_spent` can fail, and the page can show an odd figure.
   - *Suggestion:* round totals to 2 decimals in the helpers, and say the tests compare to 2 decimals.
2. **The "Demo User" test can't tell live data from hardcoded data.**
   - The Definition of done says the page must show real values "not the hardcoded strings", but the route test only checks that "Demo User" and "demo@spendly.com" appear.
   - If Step 04's hardcoded data used the same name and email, the test passes without any database query.
   - *Suggestion:* also log in as a newly registered user with a unique name and check that name appears.
3. **Newest-first ordering has no tie-break.** Two expenses on the same date can come back in either order. *Suggestion:* add a second sort key such as the id, descending.
4. **No isolation for tests.** The tests rely on the seed user and the real `spendly.db`. Spec 01 has no way to point at a different database (see the Spec 01 report).

### Missing coverage

5. **No test that users only see their own data.**
   - The Overview's key promise is that "every logged-in user sees their own expenses".
   - No test creates a second user and checks that user's profile excludes the first user's expenses.
6. **No test for the rounding rule.** The rule says the largest category absorbs the remainder so `pct` sums to 100. A case such as three equal categories (33, 33, 34) is the one that exercises it.
7. **No test for connection closing.** The rule requires each helper to close its connection, but nothing checks it.
8. **The seed amounts are undefined.** Specs 01 and 05 together depend on amounts that sum to 346.24, with 8 expenses across 7 categories and "Bills" as the top total (see the Spec 01 report).

### Ambiguities

9. **`get_user_by_id` can return `None`, but the route doesn't say what to do.** If a signed-in user's row is gone (for example the database was reset), the page would crash. *Suggestion:* clear the session and redirect to `/login`.
10. **Top-category ties are undefined.** If two categories have the same total, which one wins?
11. **Zero totals display.** The helper returns `0`, but the Definition of done expects `₹0.00` and `₹346.24`. The two-decimal format isn't specified anywhere.
12. **The 10-row limit has no follow-up.**
    - Totals and the breakdown cover every expense, but the table shows only the latest 10.
    - A user with 12 expenses sees a count of 12 and a list of 10, and no way to see the rest.
13. **Month names depend on the machine's locale.** "Month YYYY" is not safe if the locale changes.
14. **Date display format for transactions is not specified.**

### Structure

15. **"Three parallel subagents" doesn't fit.**
    - There are four helpers, but the sentence describes three concerns. `get_user_by_id` isn't covered.
    - All three write to the same new file, `database/queries.py`, so parallel edits risk conflicts.
    - It is also an instruction about how to build, not a requirement, and the Definition of done has nothing to verify it.
16. **Helpers are split across two files.** `create_user` and `get_user_by_email` live in `db.py`, but the new user lookup lives in `queries.py`. A reader has to look in both.
17. **The file lists disagree.** `tests/test_backend_connection.py` appears under Tests to write but not under Files to create.
18. **The prompt's file tree is out of date.** It omits files that now exist.

## Recommended actions

1. Add 2-decimal rounding for totals (issue 1) and a tie-break for ordering (issue 3), because both can fail tests.
2. Add an isolation test with a second user (issue 5), and a unique-user check on the profile page (issue 2).
3. Define the seed amounts in Spec 01 (issue 8), and decide what the route does when the user row is missing (issue 9).
4. Issues 10 to 18 can wait. Issue 15 is worth fixing if parallel agents are actually used.
