# Code Review: Spendly (Expense_Tracker_Basic)

**Date:** 2026-11-04
**Scope:** `app.py`, `database/db.py`, all templates, `static/js/main.js`, `static/css/style.css`, `requirements.txt`, git state.
**Not revipush the ewed:** the nine `Spec_*` prompt files. No tests were run, because the repo has none.

## Verdict

This is a front-end scaffold. The pages render, but nothing works end to end yet: there are no accounts, no database and no expense logic. There is one functional bug (the forms can't submit) and several issues to fix before the real features are built.

## High severity

1. **Register and login forms can't submit.**
   `templates/register.html:20` and `templates/login.html:20` POST to `/register` and `/login`, but the routes in `app.py:16-23` accept GET only. Submitting returns **405 Method Not Allowed**.
   *Fix:* add `methods=["GET", "POST"]` and the handlers when Specs 02 and 03 are built.

2. **Legal pages describe features that don't exist.**
   `privacy.html` and `terms.html` state that passwords are hashed, accounts and all their data can be deleted, a session cookie is used, and expenses are visible only to their owner. None of this is implemented, and the specs don't plan a delete-account feature. The pages also give `support@spendly.com` (looks invented) and a "Last updated: November 3, 2026" date.
   *Fix:* build the features, or soften the wording.

3. **Privacy page contradicts the video modal.**
   The policy says data is shared only when the law requires it and has no third-party section. The landing-page modal loads a YouTube iframe, which contacts Google.
   *Fix:* use `youtube-nocookie.com` and add a "Third-party services" section to the policy.

## Medium severity

4. **Delete is a GET route.** `app.py:60` (`/expenses/<id>/delete`) can be triggered by a link, a prefetch or an `<img>` tag.
   *Fix:* make it POST-only with CSRF protection.

5. **No security groundwork yet.**
   - No `SECRET_KEY`, which sessions and login will need.
   - No CSRF protection on forms.
   - `debug=True` is hard-coded at `app.py:66`; read it from an environment variable, since the debugger allows code execution if exposed.

6. **Hard-coded URLs instead of `url_for`.** The footer links (`base.html:38-39`, `landing.html` footer) use `/terms` and `/privacy`, and the form actions use `/register` and `/login`. The rest of the templates use `url_for`.

7. **Navbar ignores login state.** `base.html:22-23` always shows "Sign in / Get started". Once login exists it needs Profile and Logout.

8. **No tests.** `pytest` and `pytest-flask` are in `requirements.txt`, but there is no `tests/` directory. A test that every public route returns 200 would have caught several of these issues.

9. **`database/db.py` contains only comments.** Nothing imports it. This is expected for now, but the app cannot store anything yet.

## Low severity and polish

10. **`app.py:12-13`: template name split across two lines** (`"landing.htm" "l"`). It works by accident. Change it to `"landing.html"`.
11. **Redundant footer.** `landing.html` overrides the footer block with markup identical to `base.html`'s. The override can be removed.
12. **Hero theme drift.** The new hero uses hard-coded colours (`#1aa88a`, `#1a2340`, white background) and a sans-serif title. The rest of the site uses the CSS variables (forest green, DM Serif Display, warm paper).
13. **Possibly dead CSS.** `.btn-ghost` was used only by the old hero. Not confirmed.
14. **Modal accessibility.** Focus is not trapped inside the dialog, and autoplay with sound may be blocked by browsers. It does close with Escape and returns focus to the trigger.
15. **Unsupported marketing claim.** "Join thousands of people who track their expenses with Spendly" (landing CTA).
16. **No `.gitattributes`.** Every git command warns that LF will be replaced by CRLF. Add `* text=auto`.
17. **README is two lines.** Add setup and run instructions (`pip install -r requirements.txt`, `python app.py`, port 5001).
18. **Repo hygiene.**
    - Local `origin` points to `Expense-tracker99`, but the latest push went to `Exp_TRK-4Nov`.
    - The earlier `ExpenseTracker` push was never completed, and that repo has unrelated history.
    - The video-modal changes are uncommitted.
    - Decide which remote is the real one and set `origin` to it.

## What's in good shape

- Template inheritance is clean (`base.html` with blocks) and the Jinja is simple.
- CSS uses variables and a consistent structure; the `legal-*` rules are reused across both legal pages.
- The video modal is vanilla JS only. It stops the video on every close path and lazy-loads the iframe.
- Register and login templates use proper `label`/`for` pairs, `required` attributes and correct input types.
- `.gitignore` covers the database file, `__pycache__` and virtual environments.

## Suggested order

1. Fix #1 and #10 (a few minutes).
2. Fix #2 and #3, since they affect what you publish.
3. Add `SECRET_KEY`, CSRF and POST-only delete (#4, #5) before building Specs 02-09.
4. Add a basic route smoke test (#8).
