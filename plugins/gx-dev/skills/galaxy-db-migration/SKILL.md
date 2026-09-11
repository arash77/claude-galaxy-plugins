---
name: galaxy-db-migration
description: >
  Galaxy database migration with Alembic - create schema changes (add table/column),
  upgrade/downgrade database versions, check migration status, troubleshoot errors.
  Use for: SQLAlchemy model changes, database schema modifications, Alembic revisions,
  migration version conflicts, lib/galaxy/model changes.
argument-hint: "[create|upgrade|downgrade|status|troubleshoot]"
---

Persona: You are a senior Galaxy database developer working with Alembic migrations.

Arguments:
- $ARGUMENTS - Optional task specifier: "create", "upgrade", "downgrade", "status", "troubleshoot"
  Examples: "", "create", "upgrade", "status"

Parse $ARGUMENTS to determine which guidance to provide.

---

## Quick Reference: Galaxy Database Migrations

Galaxy uses **Alembic** for database schema migrations with **two branches**:
- **gxy** - Galaxy model (main application database) - `lib/galaxy/model/migrations/alembic/versions_gxy/`
- **tsi** - Tool shed install model (rarely used) - `lib/galaxy/model/migrations/alembic/versions_tsi/`

### Three Scripts Available

1. **`manage_db.sh`** - Admin script for production (upgrade, downgrade, init)
2. **`scripts/db_dev.sh`** - Dev script with full Alembic features (includes `revision` command)
3. **`scripts/run_alembic.sh`** - Advanced wrapper for direct Alembic CLI access

---

## If $ARGUMENTS is empty: Display Task Menu

Present this menu to the user:

**Available tasks:**

1. **create** - Create a new migration revision
2. **upgrade** - Upgrade database to latest version
3. **downgrade** - Downgrade database to previous version
4. **status** - Check current database version vs codebase
5. **troubleshoot** - Diagnose migration errors

**Quick commands:**
- `./scripts/db_dev.sh dbversion` - Show current DB version
- `./scripts/db_dev.sh version` - Show head revision in codebase
- `./scripts/db_dev.sh history --indicate-current` - Show migration history with current position

---

## If $ARGUMENTS is "create": Guide Through Creating Migration

Follow this workflow:

### Step 1: Update the Model

Ask the user if they have:
1. Updated SQLAlchemy models in `lib/galaxy/model/__init__.py`
2. Added tests to `test/unit/data/model/mapping/test_*model_mapping.py`

If not, remind them these are prerequisites before creating a migration.

### Step 2: Create Revision File

Run:
```bash
./scripts/db_dev.sh revision -m "brief_description_of_change"
```

This creates a new file in `lib/galaxy/model/migrations/alembic/versions_gxy/` with format:
`<revision_id>_<message>.py`

### Step 3: Fill Out Migration

Open the newly created file. You'll need to implement:

**Import common utilities:**
```python
import sqlalchemy as sa
from galaxy.model.custom_types import JSONType, TrimmedString
from galaxy.model.migrations.util import (
    create_table, drop_table,
    add_column, drop_column, alter_column,
    create_index, drop_index,
    create_foreign_key, create_unique_constraint, drop_constraint,
    table_exists, column_exists, index_exists,
    transaction,
)
```

**Available utility functions:**
- `create_table(table_name, *columns)` - Create new table
- `drop_table(table_name)` - Drop table
- `add_column(table_name, column)` - Add column
- `drop_column(table_name, column_name)` - Drop column
- `alter_column(table_name, column_name, **kw)` - Modify column
- `create_index(index_name, table_name, columns, **kw)` - Create index
- `drop_index(index_name, table_name)` - Drop index
- `create_foreign_key(constraint_name, table_name, columns, referent_table, referent_columns)` - Create FK
- `create_unique_constraint(constraint_name, table_name, columns)` - Create unique constraint
- `drop_constraint(constraint_name, table_name)` - Drop constraint
- `transaction()` - Context manager for transaction wrapping

**Check functions (for conditional migrations):**
- `table_exists(table_name, default)` - Check if table exists
- `column_exists(table_name, column_name, default)` - Check if column exists
- `index_exists(index_name, table_name, default)` - Check if index exists
- `foreign_key_exists(constraint_name, table_name, default)` - Check if FK exists
- `unique_constraint_exists(constraint_name, table_name, default)` - Check if constraint exists

**Implement upgrade() and downgrade():**
```python
def upgrade():
    with transaction():
        # Your migration code here
        pass

def downgrade():
    with transaction():
        # Reverse the migration
        pass
```

### Step 4: Review Example

Suggest reading the most recent migration for reference:
```bash
# Find most recent migration
ls -t lib/galaxy/model/migrations/alembic/versions_gxy/*.py | head -1
```

Then read it to see current patterns (e.g., `04cda22c48a9_add_job_direct_credentials_table.py`).

### Step 5: Run Migration

```bash
./manage_db.sh upgrade
```

### Step 6: Verify

Check that:
1. Migration runs without errors
2. Database schema matches model
3. Tests pass: `./run_tests.sh -unit test/unit/data/model/mapping/test_*model_mapping.py`

---

## If $ARGUMENTS is "upgrade": Guide Through Upgrading

**Standard upgrade to latest:**
```bash
./manage_db.sh upgrade
```

This upgrades both gxy and tsi branches to head.

**Upgrade to specific release:**
```bash
./manage_db.sh upgrade 22.05
# or
./manage_db.sh upgrade release_22.05
```

**Upgrade only gxy branch:**
```bash
./scripts/run_alembic.sh upgrade gxy@head
```

**Upgrade by relative steps:**
```bash
./scripts/run_alembic.sh upgrade gxy@+1  # One revision forward
```

**Check status before upgrading:**
```bash
./scripts/db_dev.sh dbversion    # Current version
./scripts/db_dev.sh version      # Head version in codebase
```

**Important notes:**
- Always backup database before upgrading
- Shut down all Galaxy processes during migration to avoid deadlocks
- First-time Alembic upgrade: run without revision argument to initialize

---

## If $ARGUMENTS is "downgrade": Guide Through Downgrading

**Downgrade by one revision:**
```bash
./manage_db.sh downgrade <current_revision_id>-1
```

**Downgrade to specific revision:**
```bash
./manage_db.sh downgrade <revision_id>
```

**Downgrade to specific release:**
```bash
./manage_db.sh downgrade 22.01
# or
./manage_db.sh downgrade release_22.01
```

**Downgrade gxy branch only:**
```bash
./scripts/run_alembic.sh downgrade gxy@-1  # One revision back
```

**Downgrade to base (empty database):**
```bash
./scripts/run_alembic.sh downgrade gxy@base
```

**Check current position first:**
```bash
./scripts/db_dev.sh history --indicate-current
```

**Important notes:**
- Always backup database before downgrading
- Oldest release: 22.01
- Downgrading to 22.01 requires SQLAlchemy Migrate version 180

---

## If $ARGUMENTS is "status": Show Status Commands

**Check current database version:**
```bash
./scripts/db_dev.sh dbversion
```

Output shows current revision(s) with `(head)` marker if up-to-date.

**Check head revision in codebase:**
```bash
./scripts/db_dev.sh version
```

Shows latest revision IDs for both branches.

**View migration history:**
```bash
./scripts/db_dev.sh history --indicate-current
```

Shows chronological list with `(current)` and `(head)` markers.

**Show specific revision details:**
```bash
./scripts/db_dev.sh show <revision_id>
```

**Compare database vs codebase:**

If `dbversion` shows different revision than `version`, database needs upgrade/downgrade.

---

## If $ARGUMENTS is "troubleshoot": Provide Troubleshooting Guidance

### Problem: Deadlock detected

**Cause:** Migration requires exclusive access to database objects while Galaxy is running.

**Solution:**
1. Shut down all Galaxy processes (web servers, job handlers, workflow schedulers)
2. Run migration again
3. Restart Galaxy after successful migration

### Problem: migrations.IncorrectVersionError

**Cause:** Database not at expected SQLAlchemy Migrate version before Alembic upgrade.

**Solution:**
1. Backup database
2. Check `migrate_version` table - should be version 180
3. If < 180: Checkout 22.01 branch, run old `manage_db.sh upgrade`
4. If = 181 (rare): Downgrade to 180 using old manage_db.sh
5. Switch back to current branch
6. Run `./manage_db.sh upgrade`

### Problem: Database version mismatch on startup

**Error:** "Database is at revision X but codebase expects revision Y"

**Solution:**
1. Check which is ahead:
   ```bash
   ./scripts/db_dev.sh dbversion
   ./scripts/db_dev.sh version
   ```
2. If database behind: `./manage_db.sh upgrade`
3. If database ahead: Either upgrade codebase or downgrade database

### Problem: Migration fails with "table already exists"

**Cause:** Migration not idempotent or database in unexpected state.

**Solution:**
1. Check if table/column already exists in database
2. Use check functions in migration:
   ```python
   from galaxy.model.migrations.util import table_exists

   def upgrade():
       if not table_exists("my_table", False):
           create_table("my_table", ...)
   ```
3. Consider using `--repair` flag if implementing manual fixes

### Problem: Cannot find revision file

**Cause:** Migration file not in expected directory.

**Solution:**
- Ensure file is in `lib/galaxy/model/migrations/alembic/versions_gxy/`
- Check file naming: `<revision_id>_<message>.py`
- Verify imports and module structure

### Problem: Foreign key constraint violation

**Cause:** Migration tries to add FK but referential integrity violated.

**Solution:**
1. Clean up orphaned rows before adding constraint
2. Add data migration in upgrade() before schema change
3. Use `with transaction():` to ensure atomicity

---

## Additional Resources

**Key files to reference:**
- Models: `lib/galaxy/model/__init__.py`
- Utilities: `lib/galaxy/model/migrations/util.py`
- Tests: `test/unit/data/model/mapping/test_*model_mapping.py`
- Recent examples: `lib/galaxy/model/migrations/alembic/versions_gxy/` (check latest files)

**External documentation:**
- Alembic tutorial: https://alembic.sqlalchemy.org/en/latest/tutorial.html
- Alembic operations: https://alembic.sqlalchemy.org/en/latest/ops.html
- Galaxy admin docs: `doc/source/admin/db_migration.md`

**Common patterns to follow:**
- Always wrap operations in `with transaction():`
- Use Galaxy util functions instead of raw Alembic ops
- Implement both upgrade() and downgrade()
- Test migrations on dev database before committing
- Use descriptive revision messages
