# Spec Review Report: Step 04 Profile Page

**Date:** 2026-11-04
**Source of truth (the prompt):** `Spec_04_Profilepage_prompt.md`
**Document under review:** `.claude/specs/04-profile.md`

## Verdict

**The spec conforms to the prompt, with two deliberate additions.** All 10 required sections are present, in order, with the exact names. The four template sections are a numbered list, as required. Every prompt requirement appears in the spec.

The two additions resolve conflicts inside the prompt itself. Reviewing them turned up a bigger problem: the progress-bar rule works for hardcoded data but cannot work once Step 05 supplies live percentages. This and several other gaps are listed below.

## Format rules from the prompt

| Rule | Result | Notes |
|---|---|---|
| Output only the Markdown spec | Pass | The only addition is a `# Spec 04: Profile Page` title line |
| No code, no prose outside the spec sections | Pass | Only inline identifiers appear |
| Bullet lists throughout | Pass | |
| Numbered list for the four template sections | Pass | Items 1 to 4 under Templates |
| Checkbox list for Definition of Done | Pass | 8 checkbox items |
| Exact section names, no numbers | Pass | Same names and order |

## Section-by-section comparison

| Section | Result | Notes |
|---|---|---|
| Overview | Match | All four points present |
| Depends on | Match | Steps 01, 02, 03 with the same explanations |
| Routes | Match | `GET /profile`, logged-in only, redirect to `/login` |
| Database changes | Match | No changes; no DB queries |
| Templates | Match | `profile.html` extending `base.html`; all four sections with their fields and minimum counts |
| Files to change | Match, plus 1 addition | `app.py` bullets match; `base.html` added (see below) |
| Files to create | Match | `templates/profile.html` |
| New dependencies | Match | None |
| Rules for implementation | Match, plus 1 addition | All 9 prompt rules present; one clarification added (see below) |
| Definition of done | Match | All 8 items present, lightly reworded |

## Additions beyond the prompt

1. **`templates/base.html` added to Files to change.**
   - The prompt's Definition of done requires the navbar to show the logged-in state (username and logout link), but its Files to change lists only `app.py`.
   - `base.html` always shows "Sign in / Get started", so it must change. The addition is correct.
2. **Progress-bar widths must use CSS classes, not `style` attributes.** This follows from the "no inline styles" rule. See issue 1 below for why it doesn't hold up.
3. A title heading at the top of the file.

## Issues inherited from the prompt

1. **The CSS-class rule for progress bars breaks in Step 05.**
   - Hardcoded percentages can map to a few fixed classes. Step 05's `pct` is any integer from 0 to 100, which would need up to 101 classes.
   - *Suggestion:* use the HTML `<progress>` element (value and max attributes, no inline style), or allow one named exception like the one Spec 09 grants for `display:inline`.
2. **The navbar needs a username, but "no DB queries" and the session hold only an id.**
   - Spec 03 stores only `session["user_id"]`.
   - Showing the name in the navbar needs either a database lookup in every request or storing the name in the session.
   - *Suggestion:* decide which, and update Spec 03 or this spec to match.
3. **No stylesheet is listed.**
   - Category badges and progress bars need CSS classes, but Files to create and Files to change name no CSS file.
   - Specs 06 and 09 later assume `static/css/profile.css` and `style.css`.
   - *Suggestion:* list `static/css/profile.css` as a file to create here.
4. **There are no CSS variables for the 7 category colours.** Badges need one colour per category. The rule forbids hex values, so the variables must exist somewhere, but nothing says where or what they're called.
5. **The data shapes aren't defined.**
   - Step 05 says the template needs "no structural changes", which only holds if the hardcoded dicts already use Step 05's keys: `name`, `email`, `member_since`; `total_spent`, `transaction_count`, `top_category`; `date`, `description`, `category`, `amount`; `name`, `amount`, `pct`.
   - *Suggestion:* list those key names in this spec.
6. **The currency symbol isn't mentioned.** Step 05 requires ₹, but this spec doesn't, so the hardcoded amounts could be written with another symbol.
7. **Smaller things left open.**
   - How the avatar initials are derived from the name.
   - The format of the member-since date (Step 05 uses "Month YYYY").
   - What the page shows with no transactions.
   - Mobile layout and table accessibility.
8. **The page is hard to reach.** Spec 03 sends users to the landing page after login, and the navbar has no Profile link, so users must type `/profile`. *Suggestion:* make the navbar username link to the profile page.
9. **The login check is repeated.** This is the same missing shared guard noted in the Spec 03 report.
10. **The prompt's file tree is out of date.** It omits files that now exist, such as `terms.html` and `privacy.html`.
11. **No tests section.** The prompt doesn't ask for one. Steps 05 to 09 each include tests.

## Recommended actions

1. Replace the progress-bar rule with something that works for dynamic values, such as `<progress>` (issue 1).
2. Decide how the navbar gets the username (issue 2).
3. Add `static/css/profile.css` and the category colour variables to the spec (issues 3 and 4).
4. Fix the data key names and the ₹ symbol (issues 5 and 6).
5. Issues 7 to 11 can wait.
