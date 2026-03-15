---
name: nanograph-lance
description: "Safely migrate a nanograph database from Lance storage format v2 to v2.2. Use this skill when the user needs to upgrade their nanograph database storage format, when `nanograph doctor --verbose` shows Lance v2 datasets, when the user mentions Lance format migration or storage upgrade, or when upgrading from nanograph pre-1.0 to 1.0+. This is a destructive multi-step operation that requires careful validation — the skill ensures nothing is lost."
---

# Lance Storage Format Migration (v2 → v2.2)

nanograph 1.0 uses Lance v3, which writes datasets in Lance storage format v2.2. Databases created with earlier nanograph versions use format v2. Both formats are readable, but v2.2 is required for new Lance features and better performance.

This skill guides you through a safe export-rebuild-verify-swap migration. The process creates a fresh database alongside the old one — the old database is never modified until the new one is fully verified.

## When to use this

Run `nanograph doctor --verbose` and look at the storage format column. If any datasets show `v2`, migration is recommended. If all show `v2.2`, no migration is needed.

## What you will lose

Be upfront about this before starting:

- **CDC history** — the transaction log and CDC event log do not survive export/reimport. The new database starts with a fresh transaction history.
- **Dataset versions** — Lance version history is reset. Time-travel to old versions is no longer possible.
- **Computed embeddings** — vectors from `@embed(...)` fields are stripped during export (to avoid format issues) and must be regenerated. This costs API calls if using a real embedding provider.

Records, edges, schema, and all user data are preserved.

## Pre-flight

Before starting, verify the current state and back up:

```bash
# 1. Check current format — confirm migration is actually needed
nanograph doctor --verbose

# 2. Record current row counts for later verification
nanograph describe --json > /tmp/pre-migration-describe.json

# 3. Back up the entire database directory
cp -r <db>.nano <db>.nano.backup
```

Do not skip the backup. If anything goes wrong, `cp -r <db>.nano.backup <db>.nano` restores the original.

## Migration steps

Adapt `<db>` to the actual database name (e.g., `omni`, `starwars`, `app`). If the project has a `nanograph.toml` with `db.default_path`, the `--db` flags can be omitted for most commands.

### Step 1: Export

Export the full graph without embeddings. Embedding vectors are stripped because they may have format-specific storage details, and will be regenerated cleanly in step 5.

```bash
nanograph export --db <db>.nano --format jsonl --no-embeddings > <db>-export.jsonl
```

### Step 2: Validate the export

Count nodes and edges in the export and compare against the live database. Do not proceed if counts don't match.

```bash
# Count exported nodes (lines with "type" key)
grep -c '"type"' <db>-export.jsonl

# Count exported edges (lines with "edge" key)
grep -c '"edge"' <db>-export.jsonl

# Compare against nanograph describe output
nanograph describe --db <db>.nano
```

If the schema has been updated since the last load (e.g., new enum values were added), some exported records may contain values that are invalid under the current schema. Check the export against the schema before proceeding:

```bash
nanograph check --query queries.gq
```

If you find invalid enum values or other schema mismatches in the export, fix them with targeted jq or sed before loading. Document what you fixed.

### Step 3: Init a new database

Create a fresh database from the current schema. Use a temporary name — never overwrite the original until verification is complete.

```bash
nanograph init --db <db>-v2.nano --schema <schema_path>
```

Use the schema from `nanograph.toml`'s `schema.default_path`, or from `<db>.nano/schema.pg` if the project doesn't keep a separate schema file.

### Step 4: Load the exported data

```bash
nanograph load --db <db>-v2.nano --data <db>-export.jsonl --mode overwrite
```

If the load fails with schema validation errors, fix the export data (step 2) and retry. Do not use `--mode append` — this is a clean rebuild.

### Step 5: Regenerate embeddings

If the schema has any `@embed(...)` properties, regenerate vectors:

```bash
nanograph embed --db <db>-v2.nano
```

This calls the embedding provider configured in `nanograph.toml` or `.env.nano`. If using `provider = "mock"`, this is instant. If using `provider = "openai"`, this makes API calls and may take time depending on data volume.

If only some types have `@embed`, you can scope: `nanograph embed --db <db>-v2.nano --type Signal`.

### Step 6: Verify the new database

This is the most important step. Do not skip any check.

```bash
# Confirm storage format is v2.2
nanograph doctor --db <db>-v2.nano --verbose

# Compare row counts against pre-migration snapshot
nanograph describe --db <db>-v2.nano --json > /tmp/post-migration-describe.json

# Run integrity check
nanograph doctor --db <db>-v2.nano
```

Compare the row counts from `/tmp/pre-migration-describe.json` and `/tmp/post-migration-describe.json`. Every node and edge type should have the same count.

Then run smoke queries — especially any that touch search, traversal, and aggregation:

```bash
# If the project has aliases
nanograph run search "test query" --db <db>-v2.nano

# Or use explicit query form
nanograph run --db <db>-v2.nano --query queries.gq --name <some_query> --param key=value
```

If any check fails, stop. The old database is untouched — investigate the failure before proceeding.

### Step 7: Swap databases

Only after all verification passes:

```bash
# Move old database aside (keep it until you're confident)
mv <db>.nano <db>.nano.old

# Rename new database to the expected name
mv <db>-v2.nano <db>.nano
```

If `nanograph.toml` has `db.default_path`, no config change is needed — the path hasn't changed. If you used a different path, update `nanograph.toml`:

```toml
[db]
default_path = "<db>.nano"
```

### Step 8: Final verification

```bash
nanograph doctor --verbose
nanograph describe
```

Confirm everything works with the swapped database. Run a few queries.

### Step 9: Clean up

Once you're confident the migration succeeded:

```bash
# Remove the backup and old database
rm -rf <db>.nano.backup
rm -rf <db>.nano.old

# Remove the export file
rm <db>-export.jsonl
rm /tmp/pre-migration-describe.json /tmp/post-migration-describe.json
```

Do not rush this step. Keep the backup for at least a day. If the migration happened in a git-tracked project and the database was committed, you can always recover from git history.

## Rollback

If anything goes wrong at any point before step 7:

```bash
# Nothing has changed — the old database is still in place
# Just clean up the failed attempt
rm -rf <db>-v2.nano
rm <db>-export.jsonl
```

If something goes wrong after step 7 (swap):

```bash
# Restore from backup
rm -rf <db>.nano
mv <db>.nano.old <db>.nano
# or from the earlier backup
cp -r <db>.nano.backup <db>.nano
```

## Checklist

Use this to track progress:

- [ ] `doctor --verbose` confirms v2 format
- [ ] Backup created (`<db>.nano.backup`)
- [ ] Pre-migration row counts saved
- [ ] Export completed (`--no-embeddings`)
- [ ] Export counts match live database
- [ ] New database initialized from current schema
- [ ] Data loaded into new database
- [ ] Embeddings regenerated (if applicable)
- [ ] New database shows v2.2 in `doctor --verbose`
- [ ] Row counts match between old and new
- [ ] `doctor` passes on new database
- [ ] Smoke queries return expected results
- [ ] Databases swapped
- [ ] Final `doctor` and `describe` pass
- [ ] Old database and backup cleaned up
