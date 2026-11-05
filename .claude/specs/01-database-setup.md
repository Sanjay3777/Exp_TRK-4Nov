# Spec 01: Database Setup

## 1. Overview

This step replaces the stub in `database/db.py` with a working SQLite implementation. It creates the database file, defines the `users` and `expenses` tables, and seeds a demo account with sample data.

It is the foundational step of Spendly. Registration and login (auth), the profile page, and every expense feature (add, edit, delete, filter) read from or write to this database, so none of them can be built until it exists.

## 2. Depends on

None. This is the first step and has no prerequisites.

## 3. Routes

No new routes. The existing placeholder routes in `app.py` remain unchanged.

## 4. Database Schema

### Table A: `users`

| Column | SQLite Type | Constraints |
|---|---|---|
| id | INTEGER | PRIMARY KEY, AUTOINCREMENT |
| name | TEXT | NOT NULL |
| email | TEXT | NOT NULL, UNIQUE |
| password_hash | TEXT | NOT NULL |
| created_at | TEXT | NOT NULL, DEFAULT current timestamp, stored in SQLite's default format `YYYY-MM-DD HH:MM:SS` (UTC) |

### Table B: `expenses`

| Column | SQLite Type | Constraints |
|---|---|---|
| id | INTEGER | PRIMARY KEY, AUTOINCREMENT |
| user_id | INTEGER | NOT NULL, FOREIGN KEY references users(id) |
| amount | REAL | NOT NULL |
| category | TEXT | NOT NULL (must be one of the fixed categories in Section 10) |
| date | TEXT | NOT NULL, format must be YYYY-MM-DD |
| description | TEXT | Nullable |
| created_at | TEXT | NOT NULL, DEFAULT current timestamp |

Notes:
- `expenses.date` must always be stored as a `YYYY-MM-DD` string so that date-range filtering and sorting work correctly with plain string comparison.
- `expenses.user_id` links every expense to exactly one user.

## 5. Functions to Implement (`database/db.py`)

### `get_db()`
- Opens the SQLite database file `spendly.db` in the project root by default.
- The path can be overridden with the `SPENDLY_DB` environment variable, so tests can use a separate database and never touch the real one.
- Sets `row_factory` to `sqlite3.Row` so columns can be read by name.
- Enables `PRAGMA foreign_keys = ON` on the connection.
- Returns the open connection.

### `init_db()`
- Creates both tables (`users`, `expenses`) using `CREATE TABLE IF NOT EXISTS`.
- Safe to call multiple times without error or data loss.

### `seed_db()`
- Checks whether the `users` table already has any rows. If it does, returns early without inserting anything.
- Otherwise inserts one demo user:
  - Name: Demo User
  - Email: demo@spendly.com
  - Password: demo123, stored as a hash generated with werkzeug
- Inserts the 8 sample expenses below, linked to the demo user:
  - They cover all 7 categories listed in Section 10.
  - Dates use the `YYYY-MM-DD` format and fall in the current month.
  - No date may be later than today. Day-of-month values greater than today's day are replaced by today's day.
- The user and all 8 expenses are inserted in a single transaction. If any insert fails, nothing is kept, so the next run can seed again.

Sample expenses (amounts are fixed; they total 346.24, and Bills is the highest category total at 120.00):

| Day of month | Category | Amount | Description |
|---|---|---|---|
| 1 | Bills | 120.00 | Electricity bill |
| 3 | Food | 42.50 | Groceries |
| 5 | Transport | 25.00 | Metro card top-up |
| 8 | Health | 35.74 | Pharmacy |
| 12 | Entertainment | 30.00 | Movie night |
| 15 | Shopping | 58.00 | New shirt |
| 18 | Other | 10.00 | Miscellaneous |
| 20 | Food | 25.00 | Lunch with friends |

## 6. Changes to `app.py`

- Import `get_db`, `init_db` and `seed_db` from `database.db`.
- Call `init_db()` and `seed_db()` inside `app.app_context()` on startup.
- The database must be ready before any route is served.

## 7. Files to Change

- `database/db.py`
- `app.py`
- `.gitignore` (add `spendly.db` so the local database is never committed)

## 8. Files to Create

None.

## 9. Dependencies

- No new pip packages.
- Use `sqlite3` (Python standard library).
- Use `werkzeug.security` (already installed and listed in `requirements.txt`).

## 10. Categories (Fixed List)

These are the only valid values for `expenses.category`:

- Food
- Transport
- Bills
- Health
- Entertainment
- Shopping
- Other

## 11. Rules for Implementation

- No ORM and no SQLAlchemy.
- Parameterized queries only. Never use string formatting to build SQL.
- `PRAGMA foreign_keys = ON` on every connection.
- Store `amount` as REAL, not INTEGER.
- Hash passwords with `generate_password_hash` from `werkzeug.security`.
- `seed_db()` must be idempotent (safe to call multiple times).
- Dates must always be `YYYY-MM-DD`.

## 12. Expected Behavior

### Function behavior
- **`get_db()`**: returns a new connection to `spendly.db` in the project root. Rows can be accessed by column name, and foreign key enforcement is active.
- **`init_db()`**: after the call, both tables exist with the schema in Section 4. Calling it again changes nothing and raises no error.
- **`seed_db()`**: on an empty database, creates the demo user and the 8 expenses from Section 5 atomically. On a database that already has users, does nothing.

### Database-level enforcement
- `users.email` must be unique.
- `expenses.user_id` must refer to an existing user.
- Required columns (`NOT NULL`) must not accept missing values.
- Default timestamps are filled in automatically when `created_at` is not provided.

## 13. Error Handling Expectations

- **Duplicate email insert**: raises a UNIQUE constraint failure.
- **Expense with an invalid `user_id`**: raises a foreign key constraint failure.
- **Invalid queries**: raise clear, unmodified errors so problems are easy to debug. Errors must not be silently swallowed.

## 14. Definition of Done

- [ ] Database file is created on app startup
- [ ] Both tables exist with the correct schema and constraints
- [ ] Demo user exists with a hashed password
- [ ] 8 sample expenses exist across all 7 categories, totalling 346.24 with Bills as the top category
- [ ] No seeded expense is dated after today
- [ ] Seeding is atomic (user and expenses are inserted together or not at all)
- [ ] `SPENDLY_DB` overrides the database path
- [ ] `spendly.db` is listed in `.gitignore`
- [ ] No duplicate seed data on repeated runs
- [ ] App starts without errors
- [ ] Foreign key enforcement works
- [ ] All queries use parameterized SQL
