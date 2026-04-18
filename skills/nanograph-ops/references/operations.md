# Operations

Day-to-day operational rhythm: the post-change workflow, change-data inspection, maintenance, storage generation, and a cross-topic cheat-sheet of common mistakes.

## Contents

- [Post-change checklist](#post-change-checklist)
- [Inspect CDC with `changes`](#inspect-cdc-with-changes)
- [Compact and cleanup periodically](#compact-and-cleanup-periodically)
- [Keep nanograph up to date](#keep-nanograph-up-to-date)
- [Storage generation](#storage-generation)
- [Common mistakes](#common-mistakes)

## Post-change checklist

Schema and query edits should follow this sequence:

1. Edit `.pg` and/or `.gq`.
2. Lint: `nanograph lint --query queries.gq`.
3. If the schema changed: `nanograph migrate` (preview with `--dry-run` first).
4. If `@embed(...)` fields changed: `nanograph embed --only-null`.
5. Smoke test: `nanograph run <alias>`.

Skipping or reordering these steps leads to stale-schema errors, missing embeddings, and broken runtime behavior.

## Inspect CDC with `changes`

Use `nanograph changes` to inspect committed CDC from the CLI:

```bash
nanograph changes --format json
nanograph changes --since 42 --format jsonl
nanograph changes --from 100 --to 120 --format json
```

For new graphs, nanograph defaults to `NamespaceLineage`. In that mode:

- CDC windows are defined by `graph_version`
- inserts and updates are reconstructed from Lance lineage
- deletes come from `__graph_deletes`
- ordering is deterministic, but there is no old `seq_in_tx` contract

Main returned fields: `graph_version`, `tx_id`, `committed_at`, `change_kind`, `entity_kind`, `type_name`, `table_id`, `rowid`, `entity_id`, `logical_key`, `row`, `previous_graph_version`.

Use `--no-embeddings` when returned row payloads are too large.

## Compact and cleanup periodically

After many mutations, Lance datasets accumulate deletion markers and fragmented files:

```bash
nanograph compact
nanograph cleanup --retain-tx-versions 50
nanograph doctor
```

Do not compact after every single mutation. Batch work and compact periodically. `cleanup --retain-tx-versions N` is the main retention control for new graphs.

## Keep nanograph up to date

nanograph is under active development. Updating regularly matters for migration, CDC, embeddings, and CLI behavior:

```bash
nanograph version
brew upgrade nanograph
```

When something behaves unexpectedly, check the CLI version first.

## Storage generation

nanograph `1.2.x` uses `NamespaceLineage` for new graphs. Committed graph state, CDC windows, and managed blobs live on Lance-native rails instead of the old `graph.manifest.json` + `_wal.jsonl` layout.

If a database is still on legacy storage, do not try to patch the old files by hand. Run:

```bash
nanograph storage migrate --db <db>.nano --target lineage-native
```

See the [Lance migration docs](https://nanograph.io/docs) or the `nanograph-lance` skill for what the migration preserves, what it resets, and how rollback works.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Full reingestion to change one record | Use an `update` mutation query |
| Running from the wrong directory | `cd` to where `nanograph.toml` lives |
| Missing `@key` on node types | Add `slug: String @key` or another stable key |
| PascalCase edge names in queries | Use camelCase: `HasMentor` → `hasMentor` |
| Using `type` for edge JSONL rows | Use `edge` with `from` / `to` |
| Rebuilding the DB instead of migrating | Use `nanograph migrate` |
| Not committing schema, queries, config | Commit `.pg`, `.gq`, `nanograph.toml`, seed `.jsonl` |
| Committing `.env.nano` | Add to `.gitignore` before first commit |
| Parsing `table` output | Use `--format json` or `--format jsonl` |
| No `@description` / `@instruction` | Add annotations so agents can self-orient |
| No aliases or missing alias `format` | Alias every important agent operation |
| Guessing schema instead of calling `describe` | Run `nanograph describe --format json` first |
| Global search plus post-filtering | Traverse first, then rank |
| Editing `.gq`/`.pg` without linting | Run `nanograph lint` after every edit |
| Hardcoded values in mutations | Parameterize everything |
| Missing embeddings after `@embed` changes | Run `nanograph embed --only-null` |
| Skipping post-change workflow | Edit → lint → migrate → embed → smoke test |
| Running an outdated CLI | `brew upgrade nanograph` regularly |
| No nanograph rules in `CLAUDE.md` | Add a short operational section |

## See also

Full nanograph documentation lives at <https://nanograph.io/docs>.
