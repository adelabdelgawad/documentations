
# Alembic Database Migration Setup and Usage Guide

This guide provides a step-by-step process for setting up and managing database schema migrations with **Alembic** in a project using **SQLModel**.

---

## 1. Install System Dependencies (Linux Only)

On Linux systems, install the required system packages before setting up Alembic to ensure compatibility with PostgreSQL and Python development tools:

```bash
sudo apt install libpq-dev gcc python3-dev
```

---

## 2. Initialize Alembic

Navigate to your project’s `db` module (or wherever your database logic resides) and initialize Alembic:

```bash
uv run alembic init alembic
```

This creates an `alembic` directory containing the environment and configuration files necessary for migration management.

---

## 3. Configure Database Connection

Open the **`alembic.ini`** file and update the `sqlalchemy.url` parameter with your database connection string. For example:

```ini
sqlalchemy.url = mysql+pymysql://username:password@localhost/dbname
```

Ensure this connection string matches your target database.

---

## 4. Configure Alembic Environment

Modify **`alembic/env.py`** to integrate with SQLModel and enable schema autogeneration:

* Import your models module at the top:

  ```python
  import model  # adjust import path as needed
  ```

* Update the `target_metadata` variable:

  ```python
  target_metadata = model.metadata
  ```

This ensures Alembic is aware of your SQLModel-defined tables.

Additionally, update **`script.py.mako`** so generated migration files automatically include `sqlmodel` types:

```python
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
import sqlmodel.sql.sqltypes
${imports if imports else ""}
```

---

## 5. Create the Initial Migration

1. Mark the database as up-to-date with the current schema:

   ```bash
   uv run alembic stamp head
   ```

2. Generate the initial migration script based on your existing models:

   ```bash
   uv run alembic revision --autogenerate -m "Initial schema"
   ```

---

## 6. Apply Migrations

To apply migrations and bring the database schema up to date, run:

```bash
uv run alembic upgrade head
```

---

## 7. Manage Schema Changes

When models change, follow this workflow:

1. Generate a new migration:

   ```bash
   uv run alembic revision --autogenerate -m "Describe your changes here"
   ```

2. Apply the migration:

   ```bash
   uv run alembic upgrade head
   ```

This process ensures your database schema stays synchronized with your evolving SQLModel definitions.

---

## 8. Useful Alembic Commands

* **View migration history**:

  ```bash
  uv run alembic history --verbose
  ```

* **Downgrade to a specific version**:

  ```bash
  uv run alembic downgrade <revision_id>
  ```

* **Revert one migration step**:

  ```bash
  uv run alembic downgrade -1
  ```

---

## Notes

* If errors occur regarding the database being out of sync, run `alembic stamp head` to align Alembic’s versioning with the current database state.
* Always review generated migration scripts before applying them to ensure correctness.

---

✅ Following this structured workflow ensures smooth database schema version control with **Alembic** and **SQLModel** integration.

---

