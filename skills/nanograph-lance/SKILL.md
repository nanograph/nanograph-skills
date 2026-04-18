---
name: nanograph-lance
description: "Safely migrate a legacy nanograph database onto the current NamespaceLineage storage generation. Use this skill when the CLI says a database uses legacy storage, when a user asks about storage migration, manifest or WAL changes, or when a project needs the current lineage-native CDC rail. This is a one-shot storage migration with clear verification and rollback steps."
---

# nanograph storage migration

A one-shot migration of a legacy nanograph database onto the current `NamespaceLineage` storage generation. Preserves the current visible graph state, resets storage-era history, and leaves a verifiable backup — with clear rollback steps if anything looks wrong.

- Run `nanograph storage migrate --db <db>.nano --target lineage-native`
- Keep the current committed graph state; drop old manifest/WAL-era history
- Verify row counts, queries, embeddings, and media before deleting the backup
- For schema changes in a healthy database, use `nanograph migrate` instead — this skill is storage-only

For new graphs, nanograph 1.2.x defaults to `NamespaceLineage`. That storage generation is Lance 4 namespace-backed, uses lineage-native CDC for inserts and updates, keeps delete tombstones in `__graph_deletes`, and stores committed graph state in Lance-backed internal tables such as `__graph_snapshot` and `__graph_tx`.

## When to use this

Use this skill when any of these are true:

- the CLI refuses to open a database normally and prints `nanograph storage migrate --db ... --target lineage-native`
- the database still uses the old manifest or WAL-era storage layout
- the project wants the current default storage generation for new work

Do not use this for routine schema edits or ordinary Lance maintenance.

## Recommended target

Use:

```bash
nanograph storage migrate --db <db>.nano --target lineage-native
```

`lance-v4` still exists as an intermediate compatibility target, but `lineage-native` is the recommended steady state.

## What migration preserves

Migration preserves the currently committed visible state:

- nodes
- edges
- property values
- schema files
- indexes rebuilt for migrated tables
- remote media URIs that were already logical references

## Consequences of migration

Migration is intentionally not history-preserving.

- The migrated graph starts a fresh CDC epoch.
- `graph_version` restarts from `1`.
- Old dataset version history is not preserved.
- Old event-order assumptions such as WAL sequencing do not carry forward.
- Tools that read or patch `graph.manifest.json` or `_wal.jsonl` must stop doing that for migrated graphs.
- Managed imported media is rewritten into `__blob_store` and exposed as `lanceblob://sha256/...` URIs.

That is the tradeoff: keep the current graph contents, drop the old storage-era history.

## Backup behavior

The migrator moves the original database aside before building the new one.

- active path after success: `<db>.nano`
- automatic backup path: `<db>.nano.v3-backup`

Migration refuses to run if the backup path already exists.

## Migration steps

Substitute `<db>` with the actual database name, for example `omni`, `starwars`, or `app`.

### Step 1: pre-flight

Record the current state:

```bash
nanograph doctor --db <db>.nano --verbose
nanograph describe --db <db>.nano --json > /tmp/pre-migration-describe.json
```

If the project uses `nanograph.toml` with `db.default_path`, you can omit `--db` when running from the correct directory.

### Step 2: run the migration

```bash
nanograph storage migrate --db <db>.nano --target lineage-native
```

On success, the command reports:

- `db_path`
- `backup_path`
- `graph_version`
- `node_tables`
- `edge_tables`
- `media_uris_rewritten`

`media_uris_rewritten` counts imported managed media rows that were rewritten into `__blob_store`.

### Step 3: verify the migrated database

```bash
nanograph version --db <db>.nano
nanograph doctor --db <db>.nano --verbose
nanograph describe --db <db>.nano --json > /tmp/post-migration-describe.json
nanograph changes --db <db>.nano --format json
```

Validate:

- row counts per node and edge type match pre-migration
- expected queries still work
- embeddings behave correctly
- managed media now resolves through `lanceblob://sha256/...` for imported assets
- `changes` returns lineage-native rows using `graph_version`

Then run smoke queries for the actual project:

```bash
nanograph run --db <db>.nano --query queries.gq --name <query> --param key=value
```

### Step 4: keep the backup until the new graph is trusted

Do not delete `<db>.nano.v3-backup` immediately. Keep it until:

- smoke queries pass
- downstream automation no longer depends on old manifest or WAL files
- search, embeddings, and media-backed rows behave correctly

## Rollback

If post-migration verification fails, restore the backup:

```bash
mv <db>.nano <db>.nano.failed
mv <db>.nano.v3-backup <db>.nano
```

If the migration command itself fails, nanograph tries to restore the original database automatically.

## Operator notes

- New graphs use `NamespaceLineage` by default.
- Public CDC is read through `nanograph changes`.
- Order is deterministic by `graph_version`, type, and row identity, not old `seq_in_tx`.
- `cleanup --retain-tx-versions N` is the main retention control for new graphs.
- Managed imported media is database-managed through `__blob_store`, not the old filesystem media-root model.

## Checklist

- [ ] `doctor --verbose` and `describe --json` captured pre-migration state
- [ ] `storage migrate --target lineage-native` completed successfully
- [ ] backup exists at `<db>.nano.v3-backup`
- [ ] row counts match before and after migration
- [ ] smoke queries pass
- [ ] `changes` returns lineage-native rows
- [ ] imported managed media resolves correctly
- [ ] backup retained until the new graph is trusted
