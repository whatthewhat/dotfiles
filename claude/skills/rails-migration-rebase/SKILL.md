---
name: rails-migration-rebase
description: Resolves conflicting Rails database migrations when rebasing to master/main. Use this when asked to fix migration conflicts, rebase with migration conflicts, or when db/structure.sql or db/schema.rb has merge conflicts.
---

# Rails Migration Rebase

When rebasing a branch with Rails migrations onto master/main, timestamp conflicts in migration files can cause `db/structure.sql` or `db/schema.rb` merge conflicts. This skill provides a safe process to resolve these conflicts.

## Process Overview

1. Identify migrations **added** in the current branch (not in master/main)
2. Clean up orphaned schema_migrations entries
3. Rollback those migrations
4. Stash or commit any pending changes
5. Rebase onto master/main
6. Rename migration files with new timestamps (so they run last)
7. Run migrations to regenerate the schema file
8. Verify no master migrations were accidentally deleted

## Step-by-Step Instructions

### Step 1: Identify Branch Migrations

Find migrations **added** in your branch (not just changed — use `--diff-filter=A`):

```bash
git diff master --diff-filter=A --name-only -- db/migrate/ | grep '\.rb$'
```

**Important:** Do NOT use `git diff master --name-only` without `--diff-filter=A`. That shows all changed migrations (including ones modified or deleted from master), which will confuse branch-only migrations with master migrations.

Save these filenames - you'll need them later.

### Step 2: Check Current Migration Status and Clean Up Orphans

```bash
rails db:migrate:status
```

Note which of your branch migrations are "up" (applied).

**Check for orphaned entries** — migrations marked as "up" with `NO FILE` will block rollback with `ActiveRecord::UnknownMigrationVersionError`. Remove them first:

```bash
rails runner "ActiveRecord::Base.connection.execute(\"DELETE FROM schema_migrations WHERE version = 'ORPHANED_VERSION'\")"
```

### Step 3: Rollback Branch Migrations

For each migration added in your branch (in reverse chronological order), rollback:

```bash
rails db:rollback STEP=N
```

Where N is the number of **branch-only** migrations to rollback. Alternatively, rollback specific versions:

```bash
rails db:migrate:down VERSION=YYYYMMDDHHMMSS
```

Also clean up old branch migration versions from schema_migrations so they don't conflict after renaming:

```bash
rails runner "ActiveRecord::Base.connection.execute(\"DELETE FROM schema_migrations WHERE version = 'OLD_BRANCH_VERSION'\")"
```

Verify all branch migrations show as "down":

```bash
rails db:migrate:status
```

### Step 4: Stash Changes and Rebase

```bash
git add -A
git stash
git fetch origin
git rebase origin/master  # or origin/main
```

If there are conflicts in structure.sql/schema.rb, accept the incoming (master) version:

```bash
git checkout --theirs db/structure.sql  # or db/schema.rb
git add db/structure.sql
GIT_EDITOR=true git rebase --continue
```

**Note:** Use `GIT_EDITOR=true` to prevent the editor from opening during rebase continue.

### Step 5: Restore Your Changes

```bash
git stash pop
```

### Step 6: Rename Migration Files

Generate new timestamps for your migration files so they run after master's latest migration.

First, check master's latest migration timestamp:

```bash
ls -1 db/migrate/ | tail -5
```

Then rename each branch migration with a new timestamp after master's latest:

```bash
mv db/migrate/OLD_TIMESTAMP_migration_name.rb db/migrate/NEW_TIMESTAMP_migration_name.rb
```

Generate sequential timestamps starting from now:

```bash
date +%Y%m%d%H%M%S  # Current timestamp
```

For multiple migrations, increment the seconds to maintain order. The migration class name inside the file does NOT need to match the filename timestamp.

### Step 7: Run Migrations

Before running, check if any master migrations that were already applied to your DB are missing from `schema_migrations` (this can happen if you cleaned up entries in Step 2/3 that belonged to master). If `db:migrate` fails with `PG::DuplicateColumn` or similar errors on a **master** migration, mark it as already applied:

```bash
rails runner "ActiveRecord::Base.connection.execute(\"INSERT INTO schema_migrations (version) VALUES ('MASTER_VERSION')\")"
```

Then run migrations:

```bash
rails db:migrate
```

This regenerates `db/structure.sql` or `db/schema.rb` cleanly.

### Step 8: Verify and Commit

Verify no master migrations were accidentally deleted:

```bash
git diff origin/master --name-only --diff-filter=D -- db/migrate/
```

This should return empty. If any master migrations are listed, restore them:

```bash
git checkout origin/master -- db/migrate/MISSING_MIGRATION.rb
```

Then stage and commit:

```bash
git add db/migrate/ db/structure.sql  # or db/schema.rb
git commit --amend  # or create a new commit
```

## Important Notes

- **Never rename migrations that have been merged to master/main** — only rename migrations unique to your branch
- If using `db/structure.sql`, ensure your local database matches master's schema before running new migrations
- If working in Docker, prefix commands appropriately (e.g., `docker compose exec spring rails db:migrate`)

## Quick Reference Script

```bash
# 1. Get list of branch-only migrations (added, not just changed)
BRANCH_MIGRATIONS=$(git diff master --diff-filter=A --name-only -- db/migrate/)

# 2. Clean orphaned schema_migrations entries (NO FILE in db:migrate:status)
# rails runner "ActiveRecord::Base.connection.execute(\"DELETE FROM schema_migrations WHERE version = 'ORPHAN'\")"

# 3. Rollback branch migrations
rails db:rollback STEP=N

# 4. Rebase
git add -A && git stash
git fetch origin && git rebase origin/master
git checkout --theirs db/structure.sql && git add db/structure.sql
GIT_EDITOR=true git rebase --continue
git stash pop

# 5. Rename migrations with new timestamps and run
rails db:migrate

# 6. Verify no master migrations deleted, then commit
git diff origin/master --name-only --diff-filter=D -- db/migrate/
git add db/migrate/ db/structure.sql
git commit --amend
```

## Troubleshooting

### UnknownMigrationVersionError During Rollback

If `db:rollback` fails with `ActiveRecord::UnknownMigrationVersionError`, a migration is in `schema_migrations` but has no file on disk. Remove the orphaned entry:

```bash
rails runner "ActiveRecord::Base.connection.execute(\"DELETE FROM schema_migrations WHERE version = 'ORPHAN_VERSION'\")"
```

### DuplicateColumn/DuplicateTable During Migrate

If `db:migrate` fails because a master migration tries to add columns/tables that already exist, the migration was already applied but its `schema_migrations` entry is missing. Insert it:

```bash
rails runner "ActiveRecord::Base.connection.execute(\"INSERT INTO schema_migrations (version) VALUES ('MASTER_VERSION')\")"
```

### Schema Conflicts After Rebase

If `db/structure.sql` still conflicts after rebase, reset it from master and re-run all migrations:

```bash
git checkout origin/master -- db/structure.sql
rails db:migrate
```

### Migration Already Exists Error

If Rails complains a migration version already exists, you may have a timestamp collision. Use a different timestamp.
