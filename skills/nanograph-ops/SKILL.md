---
name: nanograph-ops
description: "Operational guide for managing nanograph embedded graph databases. Use this skill whenever the user works with nanograph files (.pg schemas, .gq queries, nanograph.toml, *.nano/ databases), asks about graph data operations, mutations, schema changes, data loading, CDC, embeddings, media, or any nanograph CLI command. Also trigger when you see nanograph-related files in the project or the user mentions graph databases or Lance-backed local graph storage in the context of this codebase."
---

# nanograph operations

## Summary

Operate nanograph databases safely and in a way that matches the current `1.2.x` CLI and storage model.

- schema-first workflow with `.pg` schemas and `.gq` queries
- targeted mutations instead of export-edit-reload loops
- `lint` for query validation, `migrate` for schema changes, `changes` for CDC
- Gemini multimodal embeddings for text, images, audio, video, and PDFs
- `NamespaceLineage` storage, lineage-native CDC, and `storage migrate` for legacy databases

This skill teaches the operational defaults that prevent the usual agent mistakes: stale-schema writes, fragile raw CLI invocations, and obsolete manifest or WAL assumptions.

## Before doing anything

Run from the directory where `nanograph.toml` lives. nanograph resolves config from the current working directory and does not walk parent directories.

Inspect the database before operating on it:

```bash
nanograph describe --format json
nanograph doctor
```

`describe --format json` gives you type names, properties, key properties, edge summaries, row counts, and `@description` / `@instruction` annotations. Do not guess the schema from JSONL or query output.

## Where to find guidance

Pick the reference that matches the task. Each file is focused so only the relevant context loads.

| Task | Reference |
|------|-----------|
| Setting up a project (`nanograph.toml`, git, `.env.nano`, agent output conventions) | [references/project-setup.md](references/project-setup.md) |
| Changing data (mutations, `@key`, aliases, JSONL format) | [references/data-changes.md](references/data-changes.md) |
| Authoring or evolving schema (`describe`, `doctor`, annotations, `lint`, `migrate`) | [references/schema.md](references/schema.md) |
| Embeddings and search (providers, traversal, backfill) | [references/embeddings.md](references/embeddings.md) |
| Day-to-day ops (post-change checklist, CDC, compact, storage generation, common mistakes) | [references/operations.md](references/operations.md) |
| Media, blobs, and multimodal Gemini embeddings | [references/blobs.md](references/blobs.md) |

For storage migration from legacy manifest/WAL databases to `NamespaceLineage`, use the `nanograph-lance` skill.

Full nanograph documentation: <https://nanograph.io/docs>.

## Non-negotiable rules

These are the mistakes agents make most often. The references explain the *why*; here is the short list.

- **Use mutation queries, not export-edit-reload.** Reingestion destroys CDC continuity and Lance history. See [data-changes.md](references/data-changes.md).
- **Put `@key` on every node type.** Without it, `update`, edge wiring, and `load --mode merge` all break.
- **Parameterize mutations.** Never hardcode slugs, dates, or one-off values.
- **Edge names in queries use camelCase.** `HasMentor` becomes `hasMentor`. JSONL keeps PascalCase.
- **Use `--format json` or `--format jsonl`.** Never parse `table` output.
- **Prefer aliases over raw `--query`/`--name`/`--param`.** Define them in `nanograph.toml` with an explicit `format`.
- **Never commit `.env.nano`.** It holds API keys.

## Post-change workflow

After editing `.pg` or `.gq` files, follow this sequence every time:

1. Edit `.pg` and/or `.gq`.
2. `nanograph lint --query queries.gq`
3. If the schema changed: `nanograph migrate` (preview with `--dry-run`).
4. If `@embed(...)` fields changed: `nanograph embed --only-null`.
5. Smoke test with `nanograph run <alias>`.

Do not skip or reorder these steps — stale-schema writes, missing embeddings, and broken runtime behavior all come from shortcuts here.

## Storage generation

nanograph `1.2.x` uses `NamespaceLineage` for new graphs. Committed graph state, CDC windows, and managed blob storage live on Lance-native rails instead of the old `graph.manifest.json` + `_wal.jsonl` layout.

If a database is still on legacy storage, do not patch the old files manually. Run:

```bash
nanograph storage migrate --db <db>.nano --target lineage-native
```

For the full migration flow (verification, rollback, checklist), use the `nanograph-lance` skill.
