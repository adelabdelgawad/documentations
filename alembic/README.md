
# Alembic Database Migration Setup and Usage Guide

## 1. Install Required System Dependencies (Linux Only)

If running on a Linux operating system, before installing Alembic and related tools, execute the following command to install necessary system libraries and headers:

```
sudo apt install libpq-dev gcc python3-dev
```

These packages ensure successful compilation and integration with PostgreSQL and Python development tools.

---

## 2. Initialize Alembic in the Project

Navigate to your `db` module directory (the folder where your database-related code resides) and initialize Alembic by running:

```
alembic init alembic
```

This command creates the Alembic environment and configuration files under an `alembic` directory inside your `db` module.

---

## 3. Configure Database Connection

Open the `alembic.ini` file (created during initialization) and update the `sqlalchemy.url` parameter to match your database connection string. For example:

```
sqlalchemy.url = mysql+pymysql://username:password@localhost/dbname
```

Make sure this URL points correctly to your target database.

---

## 4. Update Alembic Environment Script

Edit the `alembic/env.py` file to integrate with your SQLModel metadata for autogeneration to work properly:

- Import SQLModel at the top:

  ```
  from sqlmodel import SQLModel
  import model.py 
  ```
(or however your models are organized) so that SQLModel.metadata includes all these tables.

- Modify the `target_metadata` assignment to:

  ```
  target_metadata = SQLModel.metadata
  ```

This setup connects Alembic’s autogenerate feature with your current SQLModel definitions.

Edit the `script.py.mako` file to ensure when  running alembic revision ...., the migration scripts will automatically have the import import sqlmodel.sql.sqltypes.

```
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
import import sqlmodel.sql.sqltypes
${imports if imports else ""}
```

---

## 5. Create the Initial Migration

For the first migration to save your current database schema, run:

```
alembic stamp head
```

This marks your database as up to date with the current migration history (without applying any changes).

Next, generate a migration script reflecting your current models:

```
alembic revision --autogenerate -m "Initial schema"
```

This creates the initial migration file representing your existing schema.

---

## 6. Apply Migrations to the Database

To apply migrations (initial or subsequent) to your database, use:

```
alembic upgrade head
```

This ensures the database schema matches your migration scripts.

---

## 7. Handling Model Changes and Creating New Migrations

Whenever you make changes to your SQLModel models and want to migrate the database accordingly:

1. Generate a new migration script that captures the schema changes:

   ```
   alembic revision --autogenerate -m "Describe your changes here"
   ```

2. Apply the new migration to the database:

   ```
   alembic upgrade head
   ```

This workflow keeps your migrations and database schema synchronized as your models evolve.

---

This structured approach ensures smooth version control of your database schema with Alembic and SQLModel integration.

If any errors occur about the target database being out of date, remember to stamp the database first or upgrade pending migrations before creating new revision scripts.
