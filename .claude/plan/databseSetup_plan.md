# Step 1 — Database Setup

## Context

Spendly (the Flask expense tracker in this repo) is being built incrementally as a series of
numbered steps. `database/db.py` is currently just a 5-line comment stub, and every route that
would need persistent data (`register`, `login`, `expenses/*`) is either unimplemented or
GET-only. This step lays the data-layer foundation everything else depends on: it implements
`get_db()`, `init_db()`, and `seed_db()` in `database/db.py` per
`.claude/specs/databseSetup.md`, and wires `init_db()`/`seed_db()` into `app.py`'s startup so the
schema and demo data exist before any route runs. No routes are added or changed in this step —
that's explicitly out of scope per the spec.

## Approach

### 1. `database/db.py`

Replace the comment-only stub with:

```python
import os
import sqlite3
from datetime import date

from werkzeug.security import generate_password_hash

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
DB_PATH = os.path.join(BASE_DIR, "expense_tracker.db")

CATEGORIES = ["Food", "Transport", "Bills", "Health", "Entertainment", "Shopping", "Other"]

CREATE_USERS_TABLE = """
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    created_at TEXT DEFAULT (datetime('now'))
)
"""

CREATE_EXPENSES_TABLE = """
CREATE TABLE IF NOT EXISTS expenses (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    amount REAL NOT NULL,
    category TEXT NOT NULL,
    date TEXT NOT NULL,
    description TEXT,
    created_at TEXT DEFAULT (datetime('now')),
    FOREIGN KEY (user_id) REFERENCES users (id)
)
"""


def get_db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    return conn


def init_db():
    conn = get_db()
    conn.execute(CREATE_USERS_TABLE)
    conn.execute(CREATE_EXPENSES_TABLE)
    conn.commit()
    conn.close()


def seed_db():
    conn = get_db()
    existing = conn.execute("SELECT COUNT(*) FROM users").fetchone()[0]
    if existing > 0:
        conn.close()
        return

    password_hash = generate_password_hash("demo123")
    cursor = conn.execute(
        "INSERT INTO users (name, email, password_hash) VALUES (?, ?, ?)",
        ("Demo User", "demo@spendly.com", password_hash),
    )
    user_id = cursor.lastrowid

    today = date.today()
    # (day-of-month, amount, category, description)
    sample_expenses = [
        (1, 12.50, "Food", "Coffee and bagel"),
        (2, 45.00, "Transport", "Monthly train pass top-up"),
        (3, 89.99, "Bills", "Electricity bill"),
        (5, 25.00, "Health", "Pharmacy — vitamins"),
        (7, 15.00, "Entertainment", "Movie ticket"),
        (9, 60.00, "Shopping", "New running shoes"),
        (11, 30.00, "Food", "Groceries"),
        (13, 20.00, "Other", "Miscellaneous"),
    ]

    for day, amount, category, description in sample_expenses:
        expense_date = today.replace(day=min(day, 28)).isoformat()
        conn.execute(
            """
            INSERT INTO expenses (user_id, amount, category, date, description)
            VALUES (?, ?, ?, ?, ?)
            """,
            (user_id, amount, category, expense_date, description),
        )

    conn.commit()
    conn.close()
```

Design notes:
- **DB location**: `expense_tracker.db` at the project root, computed via `os.path.abspath(__file__)`
  two `dirname()` calls up from `database/db.py` — resolves correctly regardless of the process's
  CWD. This matches the pre-existing (unprefixed) `.gitignore` entry `expense_tracker.db`, so no
  `.gitignore` change is needed.
- **`get_db()`**: opens a fresh connection per call (no `flask.g` caching) — simplest option that
  satisfies the spec literally and avoids introducing request-context coupling this step doesn't
  ask for. `init_db()`/`seed_db()` each open and close their own connection.
- **`DEFAULT (datetime('now'))`**: parenthesized, since SQLite requires a non-literal `DEFAULT`
  expression to be wrapped in parens.
- **Seed dates**: computed from `date.today().replace(day=...)` (capped at 28) rather than
  hardcoded literals, so seed data always lands in "the current month" whenever the app is first
  run, and never overflows a short month.
- All 7 fixed categories appear at least once across the 8 rows (Food repeats once, the most
  realistic real-world repeat).

### 2. `app.py`

Two additive insertions only — every existing line (all 5 real routes, all 5 stub routes,
comments, blank-line spacing) stays byte-for-byte unchanged, per spec §3/§6.

```python
from flask import Flask, render_template
from database.db import get_db, init_db, seed_db

app = Flask(__name__)

with app.app_context():
    init_db()
    seed_db()


# ------------------------------------------------------------------ #
# Routes                                                              #
# ------------------------------------------------------------------ #

@app.route("/")
def landing():
    ...  # unchanged from here down
```

`get_db` is imported (even though no route calls it yet) because spec §6 explicitly lists it —
future steps (login/register/expenses) will use it.

## Verification

Run from the project root with the venv interpreter (`venv/bin/python3`, since this venv has no
`activate` script):

1. **Init + seed + read back**:
   ```bash
   venv/bin/python3 -c "
   from database.db import init_db, seed_db, get_db
   init_db(); seed_db()
   conn = get_db()
   print(conn.execute('SELECT * FROM users').fetchall())
   print(conn.execute('SELECT * FROM expenses').fetchall())
   conn.close()
   "
   ```
   Expect 1 user row (hashed password, not plaintext `demo123`) and 8 expense rows across the 7
   categories.

2. **Idempotency** — call `seed_db()` twice, confirm counts stay at 1 user / 8 expenses (no
   duplication).

3. **App still starts cleanly on port 5001**:
   ```bash
   venv/bin/python3 app.py &
   sleep 1
   curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:5001/
   kill %1
   ```
   Expect `200`, no traceback, and `expense_tracker.db` now present at the project root.

4. **FK enforcement sanity check** — attempt to insert an expense with `user_id=99999`; expect
   `sqlite3.IntegrityError: FOREIGN KEY constraint failed` (confirms the PRAGMA is actually active
   on the connection, not just set-and-ignored).

5. **Unique email sanity check** — attempt to insert a second user with `demo@spendly.com`;
   expect `sqlite3.IntegrityError: UNIQUE constraint failed: users.email`.

6. **`git status --porcelain expense_tracker.db`** — expect no output (file stays git-ignored via
   the existing `.gitignore` entry).

These six checks cover every item in the spec's Definition of Done without needing a test suite
(the spec doesn't request one, and none exists yet in the repo).
