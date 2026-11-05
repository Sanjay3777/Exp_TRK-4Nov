# Spec Review Report: Step 02 Registration

**Date:** 2026-11-04
**Source of truth (the prompt):** `Spec_02_Registration.prompt.md`
**Document under review:** `.claude/specs/02-registration.md`

## Verdict

**The spec conforms to the prompt.** All 10 required sections are present, in order, with the exact names. Every requirement from the prompt appears in the spec, and nothing in the spec contradicts it.

The spec adds three bullets that the prompt does not contain. One of them (the confirm password field) fixes a real inconsistency in the prompt. The remaining findings are gaps in the prompt that the spec inherited, the most important being a missing minimum-password-length check.

## Format rules from the prompt

| Rule | Result | Notes |
|---|---|---|
| Output only the Markdown spec | Pass | The only addition is a `# Spec 02: Registration` title line |
| No code, no prose outside the spec sections | Pass | Only inline identifiers appear |
| Bullet lists throughout, no tables | Pass | No tables used |
| Checkbox list for Definition of done | Pass | 7 checkbox items |
| Section names exactly as listed, no numbers | Pass | Same names and order |

## Section-by-section comparison

| Section | Result | Notes |
|---|---|---|
| Overview | Match | All four points present: GET and POST, the four fields, flash and redirect to `/login`, entry point for authenticated features |
| Depends on | Match | Step 01 (`users` table, `get_db()`) |
| Routes | Match | `GET /register` and `POST /register`, both public |
| Database changes | Match | No new tables; `create_user(name, email, password)` with all four behaviours, including `sqlite3.IntegrityError` on a duplicate email |
| Templates | Match, plus 2 additions | All four prompt bullets present (see "Additions" below) |
| Files to change | Match, plus 1 addition | Same three files; the `register.html` line also mentions the confirm field |
| Files to create | Match | None |
| New dependencies | Match | None; `werkzeug.security`, `flash`, `redirect`, `url_for` |
| Rules for implementation | Match | All 11 rules present, including the 3-step validation order and `abort(405)` |
| Definition of done | Match | All 7 items present, lightly reworded |

## Additions beyond the prompt

1. **"Add a confirm password input"** (Templates, and repeated in Files to change).
   - Justified. The prompt requires a `confirm_password` field and a mismatch check, but its Templates list only says to add `name` attributes, and the existing `register.html` has no such input.
   - It slightly conflicts with "keep all existing visual design unchanged", because the form gets one more field. That is unavoidable.
2. **"Replace the current single `error` message block"**: clarifies how to handle the existing `error` variable in the template. Consistent with the prompt.
3. **"Reuse the existing auth classes"**: a clarification that supports the "visual design unchanged" rule.
4. A title heading at the top of the file.

## Issues inherited from the prompt

1. **Password length is promised but not enforced.**
   - The existing template's password placeholder reads "Min. 8 characters", but the validation list has only three checks (non-empty, match, unique).
   - A user can register with a 1-character password. *Suggestion:* add a minimum-length check, or change the placeholder.
2. **Email handling is under-specified.**
   - There is no step to trim whitespace or normalise case, so `A@x.com` and `a@x.com` count as different accounts.
   - "Non-empty" also doesn't say whether a field of only spaces is empty.
   - *Suggestion:* strip all inputs and lower-case the email before the checks and the insert.
3. **Prompt and template disagree.**
   - The prompt says `register.html` has "no form action or method yet", but the file already has `method="POST"` and a hardcoded `action="/register"`.
   - The spec's instruction (use `url_for('register')`) is still correct.
4. **The success message has no place to appear.**
   - The spec flashes a success message and redirects to `/login`, but Spec 03 only requires `login.html` to show flashed *error* messages.
   - *Suggestion:* make Spec 03 require showing all flashed messages, with distinct success and error styling.
5. **Re-render behaviour is incomplete.**
   - It doesn't say whether the entered name and email are kept on failure. Spec 07 does this for add-expense, so the register form is inconsistent.
   - It doesn't give an HTTP status for failures (200 or 400).
   - The error text for an empty field is not defined ("a validation error"), unlike "Email already registered" and "Passwords do not match".
6. **`abort(405)` is redundant.** Flask already returns 405 for methods not listed on the route. It only does something if the route accepts more methods than GET and POST.
7. **Hardcoded `secret_key`.** The prompt allows this "for now", but nothing marks it as something to replace. If it is committed and deployed, anyone can forge session cookies. *Suggestion:* read it from an environment variable with a dev fallback.
8. **No CSRF protection.** The form is a plain POST with no token.
9. **Connection handling is unspecified.** `create_user` doesn't say who commits and closes the connection. Spec 05 requires its helpers to close theirs, so the two are inconsistent.
10. **No tests section.** The prompt doesn't ask for one, but Specs 05 to 09 each do. Registration has the validation order and duplicate-email behaviour that are easy to test.
11. **Logged-in users can open `/register`.** Not addressed, though login doesn't exist until Step 03.

## Recommended actions

1. Decide on the minimum password length (issue 1) and the email normalisation rule (issue 2), and add both to the validation order.
2. Update Spec 03 so `login.html` displays flashed success messages (issue 4).
3. Pick a connection-closing convention for all helpers (issue 9).
4. The rest can wait, but issue 7 should be fixed before any deployment.
