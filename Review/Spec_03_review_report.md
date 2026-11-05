# Spec Review Report: Step 03 Login and Logout

**Date:** 2026-11-04
**Source of truth (the prompt):** `Spec_03_login_logout_prompt.md`
**Document under review:** `.claude/specs/03-login_logout.md`

## Verdict

**The spec conforms to the prompt.** All 10 required sections are present, in order, with the exact names. Every requirement from the prompt is in the spec, and nothing in it contradicts the prompt.

The spec makes one deliberate change: the register link is written as `url_for('register')` instead of `/register`. The remaining findings are gaps in the prompt that the spec inherited. The most important is that nothing says how routes get protected, and that the registration success message has nowhere to appear.

## Format rules from the prompt

| Rule | Result | Notes |
|---|---|---|
| Output only the Markdown spec | Pass | The only addition is a `# Spec 03: Login and Logout` title line |
| No code, no prose outside the spec sections | Pass | Only inline identifiers appear |
| Bullet lists throughout | Pass | |
| Checkbox list for Definition of Done | Pass | 7 checkbox items |
| Exact section names, no numbers | Pass | Same names and order |

## Section-by-section comparison

| Section | Result | Notes |
|---|---|---|
| Overview | Match | Verify credentials, store `session["user_id"]`, redirect to landing; `/logout` clears the session and redirects to `/`; guests versus logged-in users; prerequisite for expense features |
| Depends on | Match | Step 01 and Step 02, with the same explanations |
| Routes | Match | `GET /login`, `POST /login`, `GET /logout`, all public |
| Database changes | Match | `get_user_by_email(email)` with all three behaviours, including "must live in `database/db.py`" |
| Templates | Match, with one change | All four prompt bullets present; the register link uses `url_for` |
| Files to change | Match | Same three files |
| Files to create | Match | None |
| New dependencies | Match | None; `check_password_hash` already available |
| Rules for implementation | Match | All 11 rules present, including the generic error text and "do not reveal which field was wrong" |
| Definition of done | Match | All 7 items present, lightly reworded |

## Differences from the prompt

1. **Register link written with `url_for('register')`.** The prompt says "a link to /register" but also says "never hardcode paths". The spec resolves the conflict in favour of `url_for`. This is a correct call.
2. A title heading at the top of the file.

## Issues inherited from the prompt

1. **The registration success message has no place to appear.**
   - Spec 02 flashes a success message and redirects to `/login`.
   - This spec only requires `login.html` to show flashed *error* messages.
   - *Suggestion:* require the template to show every flashed message, with different styling for success and error.
2. **Sessions need a `secret_key`, but the dependency isn't listed.** Spec 02 sets `app.secret_key`. Without it `session["user_id"]` can't be stored. "Depends on" should say so.
3. **No shared way to protect routes.**
   - This step is "a prerequisite for all expense features", yet it provides no login-required guard.
   - Specs 04, 07, 08 and 09 each repeat the same `session.get("user_id")` check by hand.
   - There's also no way to send a user back to the page they wanted after logging in.
   - *Suggestion:* add one reusable guard here.
4. **The session isn't cleared on login.** Only `logout()` calls `session.clear()`. Clearing any existing session before storing `user_id` avoids reusing stale session data.
5. **Email handling matches Spec 02's gap.** If registration lower-cases and trims the email, login must do the same, otherwise `Demo@Spendly.com` fails to log in. Neither spec defines it.
6. **A wrong email can be told apart by timing.** An unknown email skips the password hash check, so it responds faster than a wrong password. Running a dummy hash check closes this. Low priority.
7. **No handling for blank fields or an already-logged-in visitor.** The prompt relies on `required` in the browser. A logged-in user can also open `/login` again.
8. **Logout is a GET route.** The prompt requires it. Any page can log a user out with an image tag. Spec 09 requires POST for delete, so the specs are inconsistent. Low risk, but worth deciding.
9. **No visible logged-in state after login.** Users are redirected to the landing page, and the navbar always shows "Sign in / Get started" until Step 04 changes it. There is also no "logged in" or "logged out" message.
10. **The prompt's file tree is out of date.** It says `login.html` has "no form action or method yet", but the file already has a POST form with a hardcoded action `/login`, an `{% if error %}` block, and a register link via `url_for`. The spec's "add a POST form" is partly already done, so the real work is switching to `url_for` and flashed messages.
11. **Definition of done relies on Step 01's demo user.** It uses `demo@spendly.com` / `demo123`, which comes from `seed_db()`, but "Depends on" mentions only that the users table exists.
12. **No brute-force protection and no tests section.** The prompt doesn't ask for either. Specs 05 to 09 do include tests.
13. **Connection handling is unspecified for `get_user_by_email`**, the same gap as `create_user` in Spec 02.

## Recommended actions

1. Update `login.html` and this spec to show all flashed messages (issue 1).
2. Add a shared login-required guard (issue 3), and list `secret_key` and the seeded demo user under "Depends on" (issues 2 and 11).
3. Agree one email-normalisation rule across Specs 02 and 03 (issue 5).
4. The rest can wait. Issues 4 and 8 are cheap fixes.
