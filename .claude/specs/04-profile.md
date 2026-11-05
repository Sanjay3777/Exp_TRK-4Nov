# Spec 04: Profile Page

## Overview

- This step replaces the `/profile` stub with a fully designed profile page that shows static, hardcoded data.
- The goal is to establish the complete UI layout before any real DB queries are wired up in Step 05.
- Four sections are built: a user info card, summary stats, a transaction history table, and a category breakdown.
- Building the UI first lets the team validate the design in isolation and ensures the templates are ready for the backend-connection step.

## Depends on

- Step 01: Database setup (the schema must exist)
- Step 02: Registration (user accounts must be creatable)
- Step 03: Login and Logout (the session must be set; `/profile` must be a protected route; the `login_required` decorator and `session["user_name"]` must exist)

## Routes

- `GET /profile`: render the profile page. Logged-in only. Redirect to `/login` if not authenticated.

## Database changes

- No database changes.
- The existing `users` and `expenses` tables are sufficient.
- No DB queries in this step. All data is hardcoded.

## Templates

- Create `templates/profile.html`: a full profile page extending `base.html`.
- It must contain four sections:
  1. User info card: avatar initials, name, email and member-since date (all hardcoded values).
     - Avatar initials are the first letter of the first and last word of the name, upper-cased (a single-word name gives one letter).
     - Member-since is formatted as "Month YYYY" (for example "January 2026").
  2. Summary stats row: total spent, number of transactions and top category (all hardcoded values).
  3. Transaction history table: a list of recent expenses with date, description, category badge and amount (at least 3 hardcoded rows).
  4. Category breakdown: per-category totals as progress-bar rows (at least 3 hardcoded categories). Each row uses the HTML `<progress>` element (`value` and `max="100"`), never a `style` attribute.
- Every amount is shown with the ₹ symbol (Indian Rupee), never £ or $.
- When there are no transactions, the table and breakdown show a short empty-state message instead of empty rows.
- The table is wrapped so it scrolls horizontally on narrow screens, and uses `<th scope="col">` headers.
- The template reads exactly these context keys, so Step 05 needs no structural change:
  - `user`: `name`, `email`, `member_since`
  - `stats`: `total_spent`, `transaction_count`, `top_category`
  - `expenses`: list of dicts with `date`, `description`, `category`, `amount`
  - `categories`: list of dicts with `name`, `amount`, `pct` (integer 0 to 100)

## Files to change

- `app.py`: replace the `/profile` stub with a real view function that:
  - Is protected with the shared `login_required` decorator from Step 03.
  - Passes hardcoded context variables (`user`, `stats`, `expenses`, `categories`, with the keys above) to `profile.html`.
- `templates/base.html`: update the navbar to show the logged-in state when a user is logged in, and the existing sign-in links when not. This is required by the last Definition of done item.
  - Logged in: the username (read from `session["user_name"]`, no DB query) as a link to `url_for('profile')`, plus a logout link.

## Files to create

- `templates/profile.html`
- `static/css/profile.css`: styles for the profile page, including badge classes and progress-bar styling. Loaded from `profile.html` with `url_for('static', ...)`.

## New dependencies

- None

## Rules for implementation

- No SQLAlchemy or ORMs. Use raw `sqlite3` via `get_db()` if any DB call is ever needed.
- Parameterised queries only. Never string-format SQL.
- Authentication guard: use the shared `login_required` decorator from Step 03. Do not repeat the session check by hand.
- All data passed to the template must be hardcoded Python dicts and lists in `app.py`. No DB queries in this step.
- Use CSS variables. Never hardcode hex values.
- No inline styles anywhere in `profile.html`.
  - Progress-bar widths are set with the `<progress>` element's `value` attribute, not `style` attributes.
- Define one CSS variable per category colour in `static/css/profile.css` under `:root`: `--cat-food`, `--cat-transport`, `--cat-bills`, `--cat-health`, `--cat-entertainment`, `--cat-shopping`, `--cat-other`. Badge classes are `badge-food`, `badge-transport`, and so on, one per category.
- All templates must extend `base.html`.
- Category badges must use a CSS class, not inline colour styles.
- Use `url_for()` for every internal link. Never hardcode paths.

## Definition of done

- [ ] Visiting `/profile` without being logged in redirects to `/login`
- [ ] Visiting `/profile` while logged in returns HTTP 200
- [ ] The page displays a user info card with a name and email
- [ ] The page displays at least three summary stat values (for example total spent, transaction count, top category)
- [ ] The page displays a transaction history table with at least three hardcoded rows
- [ ] The page displays a category breakdown section with at least three categories
- [ ] The navbar shows the logged-in state (username linking to `/profile`, and a logout link)
- [ ] All amounts are shown with the ₹ symbol
- [ ] Category breakdown bars use `<progress>`, and `profile.html` contains no `style` attributes
- [ ] `static/css/profile.css` defines the seven category colour variables and badge classes
- [ ] The page shows an empty-state message when given no expenses
- [ ] No hex colour values appear in `profile.html`, only CSS variables
