---
name: nanograph-lance
description: "Safely migrate a nanograph database from Lance storage format v2 to v2.2. Use this skill when the user needs to upgrade their nanograph database storage format, when `nanograph doctor --verbose` shows Lance v2 datasets, when the user mentions Lance format migration or storage upgrade, or when upgrading from nanograph pre-1.0 to 1.0+. This is a destructive multi-step operation that requires careful validation — the skill ensures nothing is lost."
metadata:
  summary: "Safely migrate a nanograph database from Lance storage format v2 to v2.2."
---

# Lance Storage Format Migration (v2 → v2.2)

nanograph 1.x links against Lance v3, which writes new datasets in Lance storage format 2.2. Databases created with earlier nanograph versions use format 2.0. Both remain readable and operational in nanograph 1.x.

Note: "Lance v3" refers to the Lance SDK/library version that nanograph links against, while "format 2.0" and "format 2.2" refer to the on-disk storage format that the SDK reads and writes.

This skill guides you through a safe export-rebuild-verify-swap migration. The process creates a fresh database alongside the old one — the old database is never modified until the new one is fully verified.

## Why upgrade to 2.2

**Better compression** — nanograph datasets are mostly text, enums, dates, graph relationship columns, and vector columns. Format 2.2 improves compression across all of these, resulting in less disk usage, cheaper backups, and less I/O during open, scan, compact, and cleanup operations.

**Safe gradual adoption** — nanograph 1.x can create new databases on 2.2, continue opening and operating existing 2.0 databases, and does not force an in-place file-format migration to upgrade the CLI. You can upgrade nanograph immediately and decide later whether to migrate existing databases.

## When to use this

Run `nanograph doctor --verbose` and look at the storage format column. If any datasets show `2.0`, migration is recommended. If all show `2.2`, no migration is needed.

## What the migration preserves and what it does not

**Preserved:** All node and edge records, schema, property values, and graph structure.

**Not preserved:**

- **CDC history** — the transaction log (`_tx_catalog.jsonl`) and CDC event log (`_cdc_log.jsonl`) do not survive export/reimport. The new database starts with a fresh transaction history.
- **Dataset versions** — Lance version history is reset. Time-travel to previous dataset versions is no longer possible.
- **Computed embeddings** — vectors from `@embed(...)` fields are stripped during export and regenerated afterward. If using a real embedding provider (`provider = "openai"` or `provider = "gemini"`), this makes API calls.

## Migration steps

Substitute `<db>` with the actual database name (e.g., `omni`, `starwars`, `app`). If the project has a `nanograph.toml` with `db.default_path`, the `--db` flags can be omitted for most commands.

### Step 1: Pre-flight

Verify the current state and back up:

```bash
# Confirm migration is needed
nanograph doctor --verbose

# Record current row counts for later verification
nanograph describe --json > /tmp/pre-migration-describe.json

# Back up the entire database directory
cp -r <db>.nano <db>.nano.backup
```

Do not skip the backup. If anything goes wrong, `cp -r <db>.nano.backup <db>.nano` restores the original.

### Step 2: Export

Export the full graph without embeddings. Embedding vectors are stripped because they will be regenerated cleanly in step 6.

```bash
nanograph export --db <db>.nano --format jsonl --no-embeddings > <db>-export.jsonl
```

### Step 3: Validate the export

Compare node and edge counts against the live database:

```bash
grep -c '"type"' <db>-export.jsonl    # node count
grep -c '"edge"' <db>-export.jsonl    # edge count
nanograph describe --db <db>.nano     # compare
```

If the schema has been updated since the last load, some exported records may contain values that are invalid under the current schema (e.g., old enum values). Check the export against the schema and fix any mismatches with targeted `jq` or `sed` before loading. Document what was fixed.

### Step 4: Create a new database

Initialize from the current schema. Use a temporary name — never overwrite the original until verification is complete.

```bash
nanograph init --db <db>-v2.nano --schema <schema_path>
```

Use the schema from `nanograph.toml`'s `schema.default_path`, or from `<db>.nano/schema.pg` if the project doesn't keep a separate schema file.

### Step 5: Load the exported data

```bash
nanograph load --db <db>-v2.nano --data <db>-export.jsonl --mode overwrite
```

If the load fails with schema validation errors, fix the export (step 3) and retry.

### Step 6: Regenerate embeddings

If the schema has any `@embed(...)` properties, regenerate vectors:

```bash
nanograph embed --db <db>-v2.nano
```

This calls the embedding provider configured in `nanograph.toml` or `.env.nano`. With `provider = "mock"`, this is instant. With `provider = "openai"` or `provider = "gemini"`, this makes API calls proportional to data volume. Scope to a single type with `--type <NodeType>` if needed.

### Step 7: Verify the new database

This is the most important step. Do not skip any check.

```bash
# Confirm storage format is 2.2
nanograph doctor --db <db>-v2.nano --verbose

# Compare row counts against pre-migration snapshot
nanograph describe --db <db>-v2.nano --json > /tmp/post-migration-describe.json

# Run integrity check
nanograph doctor --db <db>-v2.nano
```

Compare the row counts from `/tmp/pre-migration-describe.json` and `/tmp/post-migration-describe.json`. Every node and edge type should have the same count.

Then run smoke queries — especially any that touch search, traversal, and aggregation:

```bash
nanograph run search "test query" --db <db>-v2.nano
nanograph run --db <db>-v2.nano --query queries.gq --name <query> --param key=value
```

If any check fails, stop. The old database is untouched — investigate the failure before proceeding.

### Step 8: Swap databases

Only after all verification passes:

```bash
mv <db>.nano <db>.nano.old
mv <db>-v2.nano <db>.nano
```

If `nanograph.toml` uses `db.default_path`, no config change is needed — the path has not changed.

### Step 9: Final verification

```bash
nanograph doctor --verbose
nanograph describe
```

Confirm everything works with the swapped database. Run a few queries.

### Step 10: Clean up

Once confident the migration succeeded:

```bash
rm -rf <db>.nano.backup
rm -rf <db>.nano.old
rm <db>-export.jsonl
rm /tmp/pre-migration-describe.json /tmp/post-migration-describe.json
```

Keep the backup for at least a day before deleting. If the migration happened in a git-tracked project and the database was committed, you can always recover from git history.

## Rollback

**Before step 8** (swap): the old database is still in place. Clean up the failed attempt:

```bash
rm -rf <db>-v2.nano
rm <db>-export.jsonl
```

**After step 8** (swap): restore from backup:

```bash
rm -rf <db>.nano
mv <db>.nano.old <db>.nano
```

## Checklist

- [ ] `doctor --verbose` confirms current storage format (`2.0` or `2.2`)
- [ ] Backup created (`<db>.nano.backup`)
- [ ] Pre-migration row counts saved
- [ ] Export completed with `--no-embeddings`
- [ ] Export counts match live database
- [ ] New database initialized from current schema
- [ ] Data loaded into new database
- [ ] Embeddings regenerated (if applicable)
- [ ] New database shows 2.2 in `doctor --verbose`
- [ ] Row counts match between old and new
- [ ] `doctor` passes on new database
- [ ] Smoke queries return expected results
- [ ] Databases swapped
- [ ] Final `doctor` and `describe` pass
- [ ] Backup and old database cleaned up
