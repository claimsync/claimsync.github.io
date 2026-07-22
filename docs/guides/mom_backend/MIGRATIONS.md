# Migration Runner — End-to-End Reference

## What It Does

Every time the server starts, it automatically checks every tenant database for
unapplied SQL migration files and runs them in order. No manual `psql` needed.
No Alembic. A lightweight `schema_migrations` table in each DB tracks what has
already been applied so each file runs exactly once, even across restarts.

It also **protects applied files from being edited** — when a migration runs
successfully, a fingerprint of that file is saved to the DB. On every startup
the runner re-checks the fingerprint. Same fingerprint → file untouched → all
good. Different fingerprint → file was edited → red error, new migrations
blocked until it is fixed.

**How to fix it:** either revert the file to its original content and restart,
or if you genuinely need a schema change — leave the file alone and create a
new `NNNN_alter_...sql` file with the change instead.

---

## Files Involved

| File | Role |
|------|------|
| `webapi/migrations/*.sql` | The actual SQL files to run |
| `webapi/db/migration_runner.py` | All runner logic |
| `webapi/main.py` → `_bg_init()` | Startup hook that calls the runner |

---

## Step-by-Step Flow

### 1. Server starts (`python app.py`)

`webapi/main.py` creates a background thread (`_bg_init`) immediately on startup.
The FastAPI server is already accepting requests while this thread runs.

```
app.py
 └─ uvicorn starts webapi.main:app
     └─ lifespan() fires
         └─ threading.Thread(_bg_init).start()   ← non-blocking
             ├─ init_tenant_engines()             ← connect to all tenant DBs
             ├─ run_migrations_all_tenants()      ← ← ← migration runner
             └─ start_scheduler()
```

---

### 2. `run_migrations_all_tenants()` — outer loop

```
webapi/db/migration_runner.py  →  run_migrations_all_tenants()
```

1. Calls `get_all_tenant_engines()` from `session.py` to get all initialized
   tenant engines (e.g. `sandbox_staging_db`, `pace_semi`, `longevity_db`).
2. Logs a START banner with tenant count.
3. Loops over every tenant and calls `run_migrations(engine, db_name)`.
4. If one tenant fails, it logs the error and **continues** to the next tenant
   — one broken DB never blocks the others.
5. Logs a DONE banner with total count applied and elapsed time.

**Log output:**
```
[Migration] ───────────────────────────────────────────────────────────
[Migration] START — checking 4 tenant(s)
[Migration] ───────────────────────────────────────────────────────────
... (per-tenant output)
[Migration] ───────────────────────────────────────────────────────────
[Migration] DONE — 25 new migration(s) applied across 4 tenant(s) in 102.5s
[Migration] ───────────────────────────────────────────────────────────
```

---

### 3. `run_migrations(engine, db_name)` — per tenant

```
webapi/db/migration_runner.py  →  run_migrations()
```

For each tenant:

1. **Discover files** — calls `_discover_migrations()` (see step 4).
2. **Open a connection** to the tenant DB.
3. **Ensure tracking table** — runs `CREATE TABLE IF NOT EXISTS schema_migrations`
   and `ALTER TABLE schema_migrations ADD COLUMN IF NOT EXISTS checksum TEXT`.
   Both are silent no-ops if already present.
4. **Read applied set** — `SELECT filename, checksum FROM schema_migrations`.
5. **Verify integrity** — calls `_verify_checksums()` (see step 5). If any
   applied file has been modified, **all pending migrations are blocked** for
   this tenant and the function returns early.
6. **Compute pending** — files discovered but not in the applied set, in order.
7. If nothing pending → logs "all up to date" and returns.
8. If pending → logs each filename with `→`, then calls `_apply_one()` for each.
9. Logs "done — N migration(s) applied".

---

### 4. `_discover_migrations()` — file discovery + naming check

```
webapi/db/migration_runner.py  →  _discover_migrations()
```

1. Globs `webapi/migrations/*.sql` and sorts lexicographically.
   Lexicographic order = numeric order because filenames are zero-padded:
   `0001_...` < `0002_...` < ... < `0025_...`.

2. Validates each filename against the regex:
   ```
   ^\d{4}_(create|add|alter|drop|rename)_[a-z0-9_]+\.sql$
   ```

3. Files that match → added to the valid list (will be executed).

4. Files that **don't** match → logged as a **red full-block warning** and
   **silently skipped** — they are never executed.

**Non-standard filename warning (shows in RED in terminal):**
```
[Migration] █████████████████████████████████████████████████████
[Migration] ⛔  NON-STANDARD MIGRATION FILE(S) — IGNORED
[Migration]    These files will NOT be executed.
[Migration]    Rename to:  NNNN_verb_description.sql
[Migration]    verb must be one of: create | add | alter | drop | rename
[Migration]    ✗  bad_filename.sql
[Migration] █████████████████████████████████████████████████████
```

**Valid naming examples:**
```
0001_create_user_sessions_table.sql             ✓
0013_add_is_active_to_participants.sql          ✓
0018_alter_clinical_recommendation_to_jsonb.sql ✓

add_column.sql          ✗  missing sequence number
0001_update_users.sql   ✗  "update" is not an allowed verb
0001_Add_Column.sql     ✗  uppercase not allowed
```

---

### 5. `_verify_checksums()` — file integrity check

```
webapi/db/migration_runner.py  →  _verify_checksums()
```

Runs **before** any new migration is applied.

For every file already recorded in `schema_migrations`, the runner:
1. Re-reads the file from disk.
2. Takes a new fingerprint of it.
3. Compares it to the fingerprint saved in the DB when the file first ran.

**Same fingerprint → file untouched → continue normally.**

**Different fingerprint → file was edited after it ran:**

```
[Migration] █████████████████████████████████████████████████████
[Migration] ⛔  TAMPERED MIGRATION FILE(S) DETECTED — sandbox_staging_db
[Migration]    These files were modified AFTER being applied to the DB.
[Migration]    Running new migrations is BLOCKED until this is resolved.
[Migration]    To fix: revert the file to its original content,
[Migration]    or create a NEW migration file for the schema change.
[Migration]    ✗  0003_create_medication_optimization_triage_table.sql
[Migration] █████████████████████████████████████████████████████
```

- New migrations for that tenant are blocked. Other tenants are unaffected.
- Files with no saved fingerprint (applied before this feature existed) are **automatically backfilled** on the next startup — the runner reads each file, computes its fingerprint, and fills in the missing value. After one restart all rows are covered and tamper detection is fully active.

**To fix:**
- Revert the file to its original content → restart → runner proceeds normally.
- Or create a new `NNNN_alter_...sql` file for the intended change instead.

---

### 6. `_apply_one(conn, filename)` — execute one file

```
webapi/db/migration_runner.py  →  _apply_one()
```

1. Reads the `.sql` file from disk.
2. Passes the raw SQL to `_split_statements()` (see step 7).
3. Executes each statement individually via `conn.execute(text(stmt))`.
4. On exception, extracts the PostgreSQL SQLSTATE code:
   - **psycopg2**: reads `exc.orig.pgcode`
   - **pg8000**: reads `exc.orig.args[0]['C']`
5. Handles three outcomes per statement:

   | SQLSTATE | Meaning | Action |
   |----------|---------|--------|
   | `42P07`, `42701`, `42710`, `42P06`, `42723` | Object already exists | Log `⚠`, rollback that statement, continue |
   | `42501` | Insufficient privilege | Log clear error, rollback, raise (stops this tenant) |
   | Anything else | Real error | Re-raise (stops this tenant) |

6. After all statements succeed → takes a fingerprint of the file and writes to
   `schema_migrations`:
   ```sql
   INSERT INTO schema_migrations (filename, checksum) VALUES (:f, :c)
   ```
7. Commits and logs `✓ filename  (Nms)`.

**The "already exists" handling** is critical for tenants where migrations were
applied manually before the runner existed. Those statements are skipped and the
file is still recorded as applied.

---

### 7. `_split_statements(sql)` — SQL-aware statement splitter

```
webapi/db/migration_runner.py  →  _split_statements()
```

**Why this exists:** a naive `sql.split(";")` breaks `CREATE FUNCTION` blocks
because PostgreSQL dollar-quoted function bodies (`$$...$$`) contain semicolons.

The parser scans the SQL character by character and tracks quoting context:

| Context | Entered by | Exited by | Semicolons inside |
|---------|------------|-----------|-------------------|
| Normal | — | — | Split point |
| Single-quoted string | `'` | closing `'` (not `''`) | Ignored |
| Dollar-quoted block | `$$` or `$tag$` | matching `$$` / `$tag$` | Ignored |
| Single-line comment | `--` | end of line | Ignored |
| Block comment | `/*` | `*/` | Ignored |

Only a `;` in **normal context** triggers a statement split.

**Example — function with semicolons inside `$$`:**
```sql
CREATE TABLE foo (id INT);              -- statement 1

CREATE FUNCTION bar() RETURNS void AS $$
DECLARE x INT;                          -- semicolon INSIDE $$, ignored
BEGIN
  UPDATE foo SET id = 1;               -- semicolon INSIDE $$, ignored
END;                                   -- semicolon INSIDE $$, ignored
$$ LANGUAGE plpgsql;                   -- THIS semicolon splits → statement 2

CREATE INDEX idx ON foo (id);          -- statement 3
```
Result: 3 statements, not 6.

---

## The `schema_migrations` Tracking Table

Created automatically in every tenant DB on first run. Never needs manual setup.

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
    filename   TEXT PRIMARY KEY,
    applied_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    checksum   TEXT
);
```

| Column | Purpose |
|--------|---------|
| `filename` | The migration filename — primary key, unique per DB |
| `applied_at` | Timestamp when it was applied |
| `checksum` | Fingerprint of the file content saved at apply time — used to detect if the file is edited later |

**To see what has been applied:**
```sql
-- Most recently applied first
SELECT filename, applied_at, checksum
FROM schema_migrations
ORDER BY applied_at DESC;

-- In migration order
SELECT filename, applied_at
FROM schema_migrations
ORDER BY filename;
```

**To check what is pending** (not yet applied):
compare the output of the query above against the files in `webapi/migrations/`.
Any file in the folder that does NOT appear in the table will run on next startup.

---

## Error Handling Summary

| Scenario | Behavior |
|----------|----------|
| File in `migrations/` has non-standard name | Red warning block, file skipped entirely |
| Applied file modified after being applied | Red tamper alert, all pending migrations blocked for that tenant |
| File listed in tracking table but deleted from disk | Skipped silently (no checksum to verify) |
| Statement fails with "already exists" (42P07 etc.) | Statement skipped, file still marked applied |
| DB user lacks permission (42501) | Clear error logged, that tenant skipped, others continue |
| Any other DB error | Error logged, that tenant skipped, others continue |
| All tenants fail | Server still starts normally — migration errors are non-fatal |

---

## How to Add a New Migration

1. Name the file: `NNNN_verb_description.sql`
   - `NNNN` = next number in sequence (check the highest existing file)
   - `verb` = `create` | `add` | `alter` | `drop` | `rename`
   - `description` = lowercase words separated by `_`

2. Write idempotent SQL — always use:
   - `CREATE TABLE IF NOT EXISTS`
   - `ADD COLUMN IF NOT EXISTS`
   - `CREATE INDEX IF NOT EXISTS`
   - `CREATE OR REPLACE FUNCTION`
   - `DO $$ BEGIN ... EXCEPTION WHEN duplicate_column THEN NULL; END $$` for
     cases where `IF NOT EXISTS` isn't available

3. Drop the file into `webapi/migrations/` and restart the server.
   The runner picks it up automatically.

4. **Never edit an already-applied file.** If you need a schema change, create a
   new migration file. Editing an applied file triggers the tamper detection
   and blocks all pending migrations.

**Example:**
```sql
-- 0027_add_discharge_date_to_participants.sql
ALTER TABLE participants
    ADD COLUMN IF NOT EXISTS discharge_date DATE;
```

---

## DB Permissions Issues

If a tenant DB user doesn't own a table or schema, migrations that touch that
object will fail with SQLSTATE `42501`. The runner logs:

```
[Migration]   ✗ 0001_create_user_sessions_table.sql: insufficient DB privileges —
   the DB user does not own this object.
   Ask your DBA to grant ownership or run this migration manually.
```

**Fix options:**
```sql
-- Option A: transfer ownership to the app user
ALTER TABLE user_sessions OWNER TO sandbox_staging_user;

-- Option B: grant schema create permission
GRANT CREATE ON SCHEMA medication_optimization TO longevity_user;
```

Then restart the server — the runner will retry the pending migration.

---

## Complete Log Reference

| Log line | What it means |
|----------|---------------|
| `START — checking N tenant(s)` | Runner is beginning |
| `sandbox_db: all up to date (26 migration(s) already applied)` | Nothing to do |
| `sandbox_db: 3 new migration(s) to apply (23 already applied, 26 total)` | Will apply 3 files |
| `  → 0024_add_missing_columns.sql` | About to apply this file |
| `  ✓ 0024_add_missing_columns.sql  (42ms)` | Applied successfully, checksum stored |
| `  ⚠  0001_create_...: statement skipped (already exists, pgcode=42P07)` | Object existed, continued |
| `  ✗ 0001_create_...: insufficient DB privileges` | Permissions issue, needs DBA |
| `sandbox_db: done — 3 migration(s) applied` | Tenant finished |
| `Skipping sandbox_db due to error above` | Tenant failed, moved to next |
| `DONE — 50 new migration(s) applied across 4 tenant(s) in 102.5s` | All tenants finished |
| `DONE — all tenants up to date (4 tenant(s) checked in 0.04s)` | Nothing was pending |
| Red `⛔  NON-STANDARD MIGRATION FILE(S)` block | File in folder has wrong name format |
| Red `⛔  TAMPERED MIGRATION FILE(S) DETECTED` block | Applied file was modified — pending migrations blocked |
