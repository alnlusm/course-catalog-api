# course-catalog-api

Backend of the course catalog: FastAPI, SQLAlchemy 2.1, PostgreSQL, and Alembic migrations (Labs 4–7 of "Advanced Web Technologies").

## How to run

You need Python 3.11 or newer and PostgreSQL 16 or newer.

Once, as the `postgres` superuser, create the role and the database:

```sql
CREATE ROLE catalog_app LOGIN PASSWORD 'change-me';
CREATE DATABASE course_catalog OWNER catalog_app;
```

Then, from the project folder:

```bash
python -m venv .venv
source .venv/bin/activate          # Windows (PowerShell): .venv\Scripts\Activate.ps1
pip install -r requirements.txt
cp .env.example .env               # Windows (PowerShell): Copy-Item .env.example .env
alembic upgrade head
psql -h localhost -U catalog_app -d course_catalog -v ON_ERROR_STOP=1 -f seed.sql
fastapi dev main.py
```

API documentation: http://localhost:8000/docs. The password `change-me` is for teaching; in a real project, passwords are never put into the repository.

Checks (with the server running for the last one):

```bash
python check_db.py       # connected to course_catalog as catalog_app, PostgreSQL ...
python check_models.py   # 6, 3 and 6 rows; names only in models / database: none
alembic check            # No new upgrade operations detected.
python try_errors.py     # 5 of 5 as expected
```

| File | Purpose |
| --- | --- |
| `config.py` | settings: `DATABASE_URL` and `SQL_ECHO` come from `.env` or environment variables |
| `db.py` | `Engine`, `Base` with `naming_convention`, `SessionLocal`, the `get_db` dependency |
| `models.py` | SQLAlchemy models `Course`, `Student`, `Enrollment` (the tables) |
| `schemas.py` | Pydantic schemas (the JSON format, camelCase) |
| `main.py` | FastAPI routes and the constraint-name → HTTP response mapping |
| `alembic/versions/` | migrations `0001_create_catalog_tables.py`, `0002_add_course_level.py` |
| `seed.sql` | initial data; clears the tables first (`TRUNCATE ... RESTART IDENTITY`) |
| `check_db.py`, `check_models.py`, `try_errors.py` | connection check, models-vs-database check, error-response check |

## What changed in the API

- **Data is stored in PostgreSQL.** Before, the courses were a list in `data.py` and the enrollments a list in memory, and everything was reset on every restart. Now likes, students and enrollments survive a server restart.
- **`POST /enrollments` takes `studentId` instead of `studentName`.** An enrollment references a student by number, so the student is created first with `POST /students` (`{"name", "email"}` → 201 with `id`). The response now has `id`, `courseId`, `studentId` and `enrolledAt`, which the database fills in.
- **New routes and a richer list.** `POST /students` creates a student; `GET /stats/courses` returns the number of enrollments of every course, including courses with 0; `GET /enrollments` now also returns `courseTitle` and `studentName` (a JOIN of three tables).
- **The `level` field.** Every course has a required level: `beginner`, `intermediate` or `advanced`. `GET /courses?level=beginner` filters by it and can be combined with `is_elective`, `sort`, `page` and `page_size`.
- **Database errors become clear responses.** A taken email or a repeated enrollment gives 409, an unknown course or student gives 404; the response is chosen by the name of the violated constraint. A too-short name is rejected by Pydantic with 422 before the database. An unknown violation stays a 500, because it means a bug in the code.

Routes: `GET /`, `GET /courses`, `GET /courses/{course_id}`, `POST /courses/{course_id}/like`, `POST /students`, `POST /enrollments`, `GET /enrollments`, `GET /stats/courses`.

## Migrations

| Command | What it does |
|---------|--------------|
| `alembic upgrade head` | applies all the missing migrations |
| `alembic downgrade -1` | rolls back the last migration |
| `alembic current` | shows the version of the database |
| `alembic history` | shows the chain of migrations |
| `alembic revision --autogenerate -m "text"` | writes a draft from models vs. database |
| `alembic check` | reports whether the models and the database have diverged |
| `alembic upgrade 0001:0002 --sql` | prints the SQL of a migration without running it |
| `alembic stamp head` | marks the database as up to date without running migrations |

The database from Lab 6 already had the tables, so migration `0001` was generated on an empty throwaway database (`scratch`), checked there with `alembic upgrade head` and `alembic check`, and the real database was only marked with `alembic stamp head` (a plain `upgrade` failed with `DuplicateTable: relation "courses" already exists`).

## Experiment log

| # | Experiment | What I saw | Cause | How I fixed it |
|---|------------|------------|-------|----------------|
| 1 | the `uq` template in `naming_convention` changed | `check_models.py`: `names only in models: ['enrollments_course_id_unique', 'students_email_unique']`, `names only in database: ['enrollments_course_id_student_id_key', 'students_email_key']` | the template builds the UNIQUE names, and they stopped matching the names PostgreSQL chose in Lab 6; Alembic would want to recreate the constraints, and the API mapping wouldn't find the names | restored `%(table_name)s_%(column_0_N_name)s_key`; the check prints `none` twice |
| 2 | `min_length` removed from `StudentIn`, name `N` | `POST /students` → 500; in the server log `IntegrityError: (psycopg.errors.CheckViolation) new row for relation "students" violates check constraint "students_name_check"` | without the Pydantic check the request reached the database; the CHECK protected the data, but `students_name_check` is not in `CONSTRAINT_ERRORS`, so the error was treated as a bug | restored `Field(min_length=2)`: Pydantic answers 422, the CHECK stays as insurance; `try_errors.py` → 5 of 5 |
| 3 | a typo in a constraint name in the dictionary (`students_email_uniq`) | a taken email → 500; the log shows the real name: `UniqueViolation ... unique constraint "students_email_key"` | the response is found by an exact name match, and the typo key never matches | restored the key `students_email_key`; 409 again |
| 4 | `title` without `NOT NULL` in the database, `alembic check` | `Detected NOT NULL on column 'courses.title'`, `FAILED: New upgrade operations detected: [('modify_nullable', ...)]`; with `varchar(100)`: `Detected type change from VARCHAR(length=100) to Text()`, `modify_type` | the database was changed by hand and no longer matches the models; `alembic check` compares them and fails with a non-zero exit code | `ALTER COLUMN title SET NOT NULL` and `TYPE text`; `No new upgrade operations detected.` |
| 5 | automatic migration `0002` on existing data | `alembic upgrade head` → `NotNullViolation: column "level" of relation "courses" contains null values`, `[SQL: ALTER TABLE courses ADD COLUMN level TEXT NOT NULL]`; `alembic current` stayed `0001` | a NOT NULL column without a default can't be added to a table with rows; autogenerate compares schemas, not data | rewrote `0002` in three steps: add the column with NULL, fill it with `UPDATE`, then `SET NOT NULL` |
| 6 | `import models` commented out in `env.py` | `alembic revision --autogenerate` → `Detected removed table 'courses'`, `'students'`, `'enrollments'`; the file had `op.drop_table(...)` for all three tables | without the import the models don't register in `Base.metadata`, it is empty, and Alembic thinks every table is extra | deleted `0003_oops.py` without running it, restored `import models`; `alembic check` is clean |
| 7 | a `phone` field in the model without a migration | `alembic check` → `Detected added column 'students.phone'`, `FAILED: New upgrade operations detected: [('add_column', None, 'students', ...)]` | the model got ahead of the database; on another machine the app would fail with `UndefinedColumn` | removed the field (in real work: write a migration and commit it with the model); check is clean |
| 8 | two migrations from one revision | `alembic heads` → `0003 (head)` and `0004 (head)`; `alembic upgrade head` → `FAILED: Multiple head revisions are present for given argument 'head'` | two branches started from `0002`, so the history has two heads and Alembic doesn't know which to apply | `alembic merge heads` created `0005` with two parents (`0003, 0004 -> 0005 (mergepoint)`); then deleted the three experimental files, one head `0002` is left |

The `upgrade → downgrade → upgrade` cycle for `0002` also passed: after `alembic downgrade -1` the query `SELECT id, level FROM courses` failed with `column "level" does not exist`, and after `alembic upgrade head` the levels were filled again by the rule (electives `beginner`, the rest `intermediate`).

## Decisions

- **The response is chosen by the constraint name, and the error is caught after `commit`.** SQLSTATE says only the kind of violation: `23505` is the same for `students_email_key` and `enrollments_course_id_student_id_key`, and the message text depends on the version and language. The constraint name (`exc.orig.diag.constraint_name`) says exactly which rule fired, and our names are fixed by `naming_convention`. Checking in advance with a `SELECT` ("is this email free?") has a race: between the check and the insert another request can take the email, and two queries cost more than one. So we just insert and handle the database's refusal.
- **A like is `UPDATE courses SET likes = likes + 1`.** The database adds one to the current value inside one command and locks the row while it does it. If the program read the number and wrote back `likes + 1`, two simultaneous requests would both read 11 and both write 12, and one like would be lost.
- **Migration `0002` is split into three steps.** The table already has rows, and a new `NOT NULL` column without a default can't be added: the rows have no value for it (experiment 5). So the column is added as nullable, the existing rows are filled by a rule (`beginner` for electives, `intermediate` for the rest), and only then `NOT NULL` is set. All three steps run in one transaction, so a failure leaves the database at `0001`.

## Migration 0002 self-check

| What to check | Result |
|---------------|--------|
| Goal and steps | yes: one change — the required `courses.level` column — in three clear steps (add nullable, fill, set NOT NULL) |
| Data | yes: the six existing courses get a level by a rule (electives `beginner`, the rest `intermediate`); `seed.sql` was updated with the same values |
| Renames and drops | yes: no renames and no drops in `upgrade`; only a new column is added |
| Rollback | yes: `downgrade` drops the column; that erases the level values, which is acceptable here because they are restored by the rule on the next upgrade, but on production it would lose levels chosen by hand |
| Verification | yes: the SQL from `--sql` was read (`ADD COLUMN level TEXT`, `UPDATE ... CASE`, `SET NOT NULL`), the `upgrade → downgrade → upgrade` cycle passed, `alembic check` is clean |
| History | yes: `alembic heads` shows one head `0002`; the migration is in the same commit as the model, schema, filter and `seed.sql` changes |
| Application | yes: `CourseOut` has the new `level` field, `GET /courses` got the `?level=` filter; for clients the response has one more field, codes didn't change |
