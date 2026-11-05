
# Spec Review Report: Step 06 Date Filter on Profile

**Date:** 2026-11-04
**Source of truth (the prompt):** `Spec_06-date-filter-profile_prompt.md`
**Document under review:** `.claude/specs/06-date-filter-profile.md`

## Verdict

**The spec conforms to the prompt.** All 10 required sections are present, in order, with the exact names. Every requirement is carried over: the two query parameters, the filter bar and its four presets, the validation rules, the query-helper signatures, all 11 implementation rules and all 11 Definition of done items. The one addition (a bullet saying the flash error must be displayed) fills a gap in the prompt.

The review found real problems in the prompt that the spec reproduced. The most important: `strptime` accepts unpadded dates such as `2026-3-5`, which would break the string-based `BETWEEN` filter, and the error message wording is wrong for a single-day range.

## Format rules from the prompt

| Rule | Result | Notes |
|---|---|---|
| Output only the Markdown spec | Pass | The only addition is a `# Spec 06: ...` title line |
| No code, no prose outside the spec sections | Pass | Only inline identifiers appear |
| Bullet lists throughout | Pass | |
| Checkbox list for Definition of Done | Pass | 11 checkbox items |
| Exact section names, no numbers | Pass | Same names and order |

## Section-by-section comparison

| Section | Result | Notes |
|---|---|---|
| Overview | Match | All four points |
| Depends on | Match | Steps 01, 04 and 05 with the same explanations |
| Routes | Match | No new routes; `date_from` and `date_to` with inclusive bounds; fall back to "All Time" when absent or malformed |
| Database changes | Match | No changes; `BETWEEN` works on the date text |
| Templates | Match, plus 1 addition | Filter bar with presets, custom range form and active state; one extra bullet (see below) |
| Files to change | Match | `app.py`, `queries.py` (all three helpers with the same signatures), `profile.html`, `profile.css` |
| Files to create | Match | None |
| New dependencies | Match | None |
| Rules for implementation | Match | All 11 rules present |
| Definition of done | Match | All 11 items present, lightly reworded |

## Additions beyond the prompt

1. **"The flashed error 'Start date must be before end date.' must be displayed on the page."** The prompt says to flash the message but never says where it appears. This is a sensible fix.
2. A title heading at the top of the file.

## Issues inherited from the prompt

### Will produce wrong results

1. **Unpadded dates pass validation but break the filter.**
   - `datetime.strptime(value, "%Y-%m-%d")` accepts `2026-3-5` and `2026-03-5`.
   - The filter then compares that raw text with stored `2026-03-05` as a string, so `BETWEEN` gives wrong results.
   - The same weakness affects Steps 07 and 08: a hand-crafted POST could save an unpadded date, which would sort and filter incorrectly forever.
   - *Suggestion:* after parsing, re-format the date to `YYYY-MM-DD` and use that value for the query and for stored data.
2. **One valid date is silently ignored.**
   - "Both provided" is the only case that filters. If a user fills only the start date, the filter does nothing and shows no message.
   - This also clashes with the Overview's promise that users can "pick any range", and with the "inclusive lower bound" and "upper bound" wording.
   - *Suggestion:* either support one-sided ranges or flash a message saying both dates are needed.
3. **The error message is wrong for a one-day range.**
   - The check rejects only `date_from` later than `date_to`, so equal dates are allowed (a single-day filter). But the message says the start date "must be before" the end date.
   - *Suggestion:* reword it, for example "Start date must not be after end date."

### Ambiguous definitions

4. **Preset boundaries are not defined.**
   - "This Month": does the range end today or on the last day of the month? Seed data "across the current month" can include dates after today.
   - "Last 3 Months" and "Last 6 Months": the start date is not defined. For example, 31 March minus 3 months is not a valid date, and "3 months back" could mean the same day or the first of that month.
   - Python's standard library has no month arithmetic, so this needs a stated rule.
5. **How the active button is chosen is not stated.** The page must compare the current parameters to each preset's range. "All Time" is active only with no parameters. After a reversed range is rejected, the active state should fall back to "All Time".
6. **The custom form is never described as pre-filled.** The Definition of done says the fields should "indicate" the filter, but nothing says they show the active dates.
7. **Blank form submits create noisy URLs.** Submitting the custom form with empty fields produces `/profile?date_from=&date_to=`. Empty strings are treated as absent, which is fine, but the URL isn't clean.
8. **The page doesn't say which period is shown.** A label such as "Showing 1 Mar to 31 Mar" would help, especially with the 10-row limit on the table.

### Missing pieces

9. **No tests section.**
   - Unlike Steps 05 and 07 to 09, the prompt asks for no tests.
   - This step has the most date logic in the project (month boundaries, leap years, December to January, inclusive ends), and every Definition of done item has to be checked by hand.
   - *Suggestion:* add a test file, and put the preset calculation in a function that takes "today" as an argument so tests are not clock-dependent.
10. **The stylesheet isn't loaded anywhere.**
    - The prompt says `static/css/profile.css` "exists", but `base.html` links only `style.css`.
    - No spec says `profile.html` should link `profile.css`, or where it was created (Spec 04 lists no CSS file).
11. **No test for users only seeing their own filtered data.** The filter adds a query clause on each helper, so it is easy to drop the `user_id` condition by mistake.
12. **Smaller points:**
    - "Today" uses the server's date, not the user's.
    - Accessibility is not mentioned. Preset links should be marked as the current page when active, and the date inputs need labels.
    - The shape of the preset data passed to the template is not defined.

## Recommended actions

1. Normalise parsed dates and reject unpadded input (issue 1). This also fixes a latent bug in Steps 07 and 08.
2. Define the preset ranges and the one-sided range behaviour (issues 2 and 4), and fix the error wording (issue 3).
3. Add a test file with an injectable "today" (issue 9), and say where `profile.css` is loaded (issue 10).
4. The rest can wait.
