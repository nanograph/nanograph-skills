---
title: Best Practices
slug: best-practices
---

# Best Practices for AI Agents

This guide is for agents and teams operating a nanograph database in a real project. The goal is simple: avoid data loss, avoid stale-schema mistakes, and give agents a safe, repeatable way to read and change graph data.

## Start Here

If you only remember a few rules, remember these:

- Always run nanograph commands from the directory where `nanograph.toml` lives.
- Use mutation queries for data changes. Do not export, edit JSONL, and re-load just to change a few records.
- Put `@key` on every node type you will ever update, merge, or connect with edges.
- After editing `.gq` or `.pg` files, run `nanograph check --query <file>`.
- Use `--json`, `--format json`, or `--format jsonl` for agent-readable output.

## Project Setup

### Run from the project directory

nanograph resolves `nanograph.toml` from the current working directory. It does not walk parent directories.

```bash
# Wrong — "database path is required" or missing-config behavior
cd /some/other/dir && nanograph describe

# Right — run from the project root
cd /path/to/project && nanograph describe
```

If the project uses `[db] default_path` and `[schema] default_path` in `nanograph.toml`, commands can omit `--db` and `--schema` only when you are in the right directory.

### Treat `nanograph.toml` as the operational backbone

`nanograph.toml` is the interface contract between the project and the agents operating on it. A good config removes fragile one-off CLI invocations and gives agents a short, predictable way to work.

A complete config usually includes:

```toml
[project]
name = "My Graph"
description = "What this graph is for — agents read this."
instruction = "Run from this directory. Use aliases for common operations."

[db]
default_path = "app.nano"

[schema]
default_path = "schema.pg"

[query]
roots = ["."]

[embedding]
provider = "openai"

[cli]
output_format = "table"

[query_aliases.search]
query = "queries.gq"
name = "semantic_search"
args = ["q"]
format = "json"

[query_aliases.add-task]
query = "queries.gq"
name = "add_task"
args = ["slug", "title", "status"]
format = "jsonl"
```

Without this config, you must pass `--db`, `--query`, `--name`, and `--format` on every command.

### Commit the right nanograph artifacts to git

These files should normally be committed:

| File | Commit? | Why |
|------|---------|-----|
| `nanograph.toml` | Yes | Shared project config, aliases, defaults |
| `schema.pg` | Yes | Schema is code |
| `queries.gq` | Yes | Query and mutation library |
| `seed.jsonl` | Yes | Reproducible bootstrap data |
| `*.nano/` | Yes, if small enough | Embedded database state is often part of the project |
| `.env.nano` | No | Secrets |

The database is not regenerable from seed.
Once agents start making mutations, the DB contains records, edges, CDC history, and embeddings that may not exist anywhere else.

Small databases under roughly `50 MB` per file are usually fine to commit. Large ones should be backed up outside git, but you should still commit the schema, queries, config, and seed files.

### Never commit `.env.nano`

`.env.nano` holds secrets like `OPENAI_API_KEY`. Add it to `.gitignore` before the first commit:

```gitignore
.env.nano
```

If it was already committed:

```bash
git rm --cached .env.nano
echo ".env.nano" >> .gitignore
git add .gitignore
git commit -m "stop tracking .env.nano"
```

Then rotate the exposed key immediately.

### Add nanograph rules to `CLAUDE.md` or `AGENTS.md`

Agents read `CLAUDE.md` and `AGENTS.md` at the repo root. If your project uses nanograph, tell them so explicitly.

A minimal section:

```markdown
## NanoGraph

This project uses a nanograph database at `app.nano`.

- Always run nanograph commands from the directory where `nanograph.toml` lives
- Use `nanograph describe --format json` before querying
- Use mutation queries for data changes — never export/edit/reimport JSONL
- Use `nanograph run <alias>` instead of constructing raw `--query` / `--name` / `--param` calls
- After editing `.gq` or `.pg`, run `nanograph check --query <file>`
- Never commit `.env.nano`
- Use `--format json` or `--format jsonl` for machine-readable output
```

Tailor it to your project: list aliases, slug conventions, mutable node types, and any required fields.

## Safe Data Changes

### Use mutations, not batch reingestion

This is the most common agent mistake. Do not export the full graph as JSONL, edit rows, and `load --mode overwrite` just to change a few records. That destroys CDC continuity, invalidates Lance dataset history, and is much slower than a targeted mutation.

```bash
# Wrong — full reingestion to update one field
nanograph export --format jsonl > dump.jsonl
# ... edit dump.jsonl ...
nanograph load --data dump.jsonl --mode overwrite

# Right — targeted mutation
nanograph run --query queries.gq --name update_status \
  --param slug=cli-acme --param status=healthy
```

Use each operation for what it is meant for:

| Goal | Command |
|------|---------|
| Add one record | `insert` mutation query |
| Update a field on an existing record | `update` mutation query |
| Remove a record | `delete` mutation query |
| Initial bulk data load | `nanograph load --mode overwrite` |
| Add a batch of new records | `nanograph load --mode append` |
| Reconcile external data into keyed records | `nanograph load --mode merge` |

Write reusable parameterized mutations in your `.gq` file when you design the schema, not later.

```
node Task {
    slug: String @key
    title: String
    status: enum(open, in_progress, done)
}

query add_task($slug: String, $title: String, $status: String) {
    insert Task { slug: $slug, title: $title, status: $status }
}

query update_task_status($slug: String, $status: String) {
    update Task set { status: $status } where slug = $slug
}

query remove_task($slug: String) {
    delete Task where slug = $slug
}
```

### Mutation constraints to know

- All mutations should be parameterized. Do not hardcode slugs, dates, or one-off values into the query body.
- `update` requires `@key` on the target node type.
- `now()` is available in filters, projections, and mutation assignments and resolves once per execution:
  ```
  query touch_client($slug: String) {
      update Client set { updatedAt: now() } where slug = $slug
  }
  ```
- Array params cannot be passed through `--param`.
- Edge inserts use `from:` and `to:` with endpoint `@key` values, not internal IDs:
  ```
  query link_signal($signal: String, $client: String) {
      insert SignalAffects { from: $signal, to: $client }
  }
  ```
- Edge names in queries use camelCase even when the schema defines them in PascalCase. `HasMentor` becomes `hasMentor`.

### Put `@key` on every node type

Every node type that participates in edges or will ever need updates should have a `@key`. In practice, every node type should have one.

```
node Client {
    slug: String @key
    name: String @unique
    status: enum(active, churned, prospect)
}
```

Without `@key`:

- `update` mutations cannot target a stable identity
- edge JSONL cannot resolve `from` / `to`
- `load --mode merge` has nothing to match on

Use stable human-readable slugs where possible. They make debugging, edge wiring, aliases, and CLI workflows much easier.

### Define aliases for every operation agents will use

Aliases are the best way to reduce agent mistakes. They turn a fragile multi-flag command into a short positional call.

```toml
[query_aliases.search]
query = "queries.gq"
name = "semantic_search"
args = ["q"]
format = "json"

[query_aliases.client]
query = "queries.gq"
name = "client_lookup"
args = ["slug"]
format = "json"

[query_aliases.add-task]
query = "queries.gq"
name = "add_task"
args = ["slug", "title", "status"]
format = "jsonl"

[query_aliases.status]
query = "queries.gq"
name = "update_task_status"
args = ["slug", "status"]
format = "jsonl"
```

```bash
nanograph run search "renewal risk"
nanograph run client cli-acme
nanograph run add-task task-42 "Draft proposal" open
nanograph run status task-42 done
```

Alias design rules:

- Always set `format`.
- Use short verb-like names.
- Put the natural scoping argument first.
- List the important aliases in `CLAUDE.md` or `AGENTS.md`.
- Do not create aliases for built-in commands like `describe` or `doctor`.

## Schema and Query Authoring

### Start with `describe` and `doctor`

Before an agent writes queries, runs mutations, or ingests data, it should inspect the database.

```bash
nanograph describe --format json
nanograph describe --type Signal --format json
nanograph doctor
```

`describe --format json` gives an agent everything it needs to operate correctly: type names, properties and their types, key properties, edge summaries, row counts, and `@description` / `@instruction` annotations. Agents that skip `describe` and try to infer the schema from output or JSONL make systematic mistakes.

`doctor` is the health check. Use `doctor --schema` to compare the current DB against the desired schema file, and `doctor --verbose` to see per-dataset Lance storage format versions.

### Use `@description` and `@instruction`

Annotations like `@description("...")` and `@instruction("...")` do not change execution, but they give agents structured intent through `describe` and `run`.

```
node Signal
  @description("Observed customer or market signals that can surface opportunities.")
  @instruction("Use Signal.slug for trace queries. summary is the retrieval field for search.")
{
    slug: String @key @description("Stable signal identifier for trace and search queries.")
    summary: String @description("Primary textual content for lexical and semantic retrieval.")
    urgency: enum(low, medium, high, critical)
}

query semantic_search($q: String)
  @description("Rank signals by semantic similarity.")
  @instruction("Use for broad conceptual search, not exact spelling.")
{
    match { $s: Signal }
    return { $s.slug, nearest($s.embedding, $q) as score }
    order { nearest($s.embedding, $q) }
    limit 5
}
```

Write these annotations when you create the schema or query. They are cheap and they prevent agent misuse.

### Always typecheck after editing `.gq` or `.pg`

No query or schema edit is complete until `check` passes.

```bash
nanograph check --query queries.gq
```

This catches:

- wrong property names
- type mismatches
- invalid traversals
- missing parameters
- malformed mutations

When the schema changed, also preview the migration plan:

```bash
nanograph migrate --dry-run
nanograph schema-diff --from old.pg --to new.pg
```

### Migrate, don't rebuild

When the schema changes, update the schema and run `nanograph migrate`. Do not rebuild from scratch unless you truly have a complete seed and do not care about current DB state.

```bash
# Right
nanograph migrate

# Wrong
nanograph init --schema new-schema.pg
nanograph load --data seed.jsonl --mode overwrite
```

Useful migration rules:

- Adding a nullable property is safe.
- Adding a node or edge type is safe.
- Renames require `@rename_from("old_name")`.
- Adding a non-nullable property to a populated type is blocked.
- Dropping properties requires explicit confirmation.

If a migration fails and leaves a stale journal in `PREPARED`, delete the journal before retrying.

If your project keeps a desired schema outside the DB directory, configure `schema.default_path` so `migrate`, `check`, and `doctor --schema` all refer to the same intended schema.

## Search and Embeddings

### Choose an embedding provider

nanograph supports OpenAI and Gemini embedding providers. Set the provider in `nanograph.toml`:

```toml
# OpenAI (default)
[embedding]
provider = "openai"

# Gemini
[embedding]
provider = "gemini"
```

Then put the matching API key in `.env.nano`:

```bash
# For OpenAI
OPENAI_API_KEY=sk-...

# For Gemini
GEMINI_API_KEY=...
```

The default models are `text-embedding-3-small` (OpenAI, native 1536 dims) and `gemini-embedding-2-preview` (Gemini, native 3072 dims). Both support dimensionality reduction — nanograph passes the schema's `Vector(dim)` to the API automatically.

If no explicit provider is set, nanograph auto-detects: it picks Gemini when `GEMINI_API_KEY` is present and `OPENAI_API_KEY` is not, and defaults to OpenAI otherwise.

Switching providers does not require re-creating the database, but existing embeddings were generated by the old model. Re-embed after switching:

```bash
nanograph embed                          # recompute all embeddings with the new provider
```

### Scope with graph traversal, then rank

nanograph's strength is combining graph structure with search. Avoid global search followed by application-side filtering.

```
// Wrong — search globally, then filter later
query bad_search($q: String) {
    match { $s: Signal }
    return { $s.slug, $s.summary, nearest($s.embedding, $q) as score }
    order { nearest($s.embedding, $q) }
    limit 20
}

// Right — constrain first, then rank
query client_signals($client: String, $q: String) {
    match {
        $c: Client { slug: $client }
        $s signalAffects $c
    }
    return { $s.slug, $s.summary, nearest($s.embedding, $q) as score }
    order { nearest($s.embedding, $q) }
    limit 5
}
```

Traverse first, then rank inside the relevant subgraph.

### Backfill embeddings after `@embed(...)` changes

When you add or change an `@embed(source_prop)` field, existing rows do not automatically gain vectors. Backfill them explicitly:

```bash
# Fill only missing vectors
nanograph embed --only-null

# Recompute all embeddings
nanograph embed

# Restrict to one type
nanograph embed --type Signal --only-null

# Preview without writing
nanograph embed --dry-run
```

If the field has `@index`, rebuild after embedding:

```bash
nanograph embed --only-null --reindex
```

## Operational Workflow

### Follow the post-change workflow

Schema and query edits should always follow the same sequence:

1. Edit `.pg` and/or `.gq`.
2. Typecheck with `nanograph check --query queries.gq`.
3. If the schema changed, run `nanograph migrate` after previewing with `--dry-run`.
4. If `@embed(...)` fields changed, run `nanograph embed --only-null`.
5. Run a smoke query or mutation alias.

Skipping or reordering these steps is how agents end up with stale-schema errors, missing embeddings, and broken runtime behavior.

### Compact and cleanup periodically

After many mutations, Lance datasets accumulate deletion markers and fragmented files.

```bash
nanograph compact
nanograph cleanup --retain-tx-versions 50
nanograph doctor
```

Do not compact after every single mutation. Batch work and compact periodically.

### Keep nanograph up to date

nanograph is under active development. Updating regularly matters, especially for migration, CDC, embeddings, and CLI behavior.

```bash
nanograph version
brew upgrade nanograph
```

When something behaves unexpectedly, check the CLI version first.

### Lance storage format upgrade (v2 → v2.2)

nanograph 1.x uses Lance v3, which writes datasets in Lance storage format v2.2. Databases created with earlier nanograph versions use Lance format v2. Both formats are readable, but v2.2 is required for new Lance features.

Check which format your datasets use:

```bash
nanograph doctor --verbose
```

To upgrade existing datasets from v2 to v2.2, use the dedicated migration command. This rewrites Lance data files in place — back up your `*.nano/` directory first.

## Agent Output and Automation

### Always use structured output

`table` output is for humans. Agents should use machine-readable output.

For `nanograph run`:

```bash
nanograph run search "renewal risk" --format json
nanograph run search "renewal risk" --format jsonl
```

For other commands:

```bash
nanograph describe --json
nanograph migrate --dry-run --json
nanograph doctor --json
```

Set `format = "json"` or `format = "jsonl"` in aliases so agents do not need to remember output flags.

### Prefer aliases over raw flag-heavy invocations

This is especially important for automation and agent loops:

```bash
# Fragile
nanograph run --query queries.gq --name update_task_status \
  --param slug=task-123 --param status=done --format jsonl

# Better
nanograph run status task-123 done
```

Use aliases for reads, mutations, and common lookups.

## Data Formats

### JSONL: nodes vs edges

Getting the JSONL shape wrong is a common ingestion mistake.

Nodes use `"type"` and `"data"`:

```json
{"type": "Client", "data": {"slug": "cli-acme", "name": "Acme Corp", "status": "active"}}
```

Edges use `"edge"`, `"from"`, and `"to"`:

```json
{"edge": "SignalAffects", "from": "sig-renewal", "to": "cli-acme"}
```

Common mistakes:

- using `type` for edge rows
- using `src` / `dst` instead of `from` / `to`
- using internal IDs instead of endpoint `@key` values
- forgetting that JSONL type and edge names use schema PascalCase exactly

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Full reingestion to change one record | Use an `update` mutation query |
| Running from the wrong directory | `cd` to where `nanograph.toml` lives |
| Missing `@key` on node types | Add `slug: String @key` or another stable key |
| PascalCase edge names in queries | Use camelCase in queries |
| Using `type` for edge JSONL rows | Use `edge` with `from` / `to` |
| Rebuilding the DB instead of migrating | Use `nanograph migrate` |
| Not committing schema, queries, and config | Commit `.pg`, `.gq`, `nanograph.toml`, and seed files |
| Committing `.env.nano` | Add it to `.gitignore` before the first commit |
| Parsing `table` output | Use JSON or JSONL |
| No `@description` / `@instruction` | Add annotations so agents can self-orient |
| No aliases or missing alias `format` | Alias every important agent operation |
| Guessing the schema instead of calling `describe` | Run `nanograph describe --format json` first |
| Global search plus post-filtering | Traverse first, then rank |
| Editing `.gq` or `.pg` without typechecking | Run `nanograph check --query <file>` after every edit |
| Hardcoded values in mutations | Parameterize everything |
| Missing embeddings after `@embed(...)` changes | Run `nanograph embed --only-null` |
| Skipping the post-change workflow | Edit -> check -> migrate -> embed -> smoke test |
| Running an outdated CLI | `brew upgrade nanograph` regularly |
| No nanograph guidance in `CLAUDE.md` or `AGENTS.md` | Add a short operational section |

## See Also

- [CLI Reference](cli-reference.md)
- [Schema Language Reference](schema.md)
- [Query Language Reference](queries.md)
- [Search Guide](search.md)
- [Project Config](config.md)
- [Lance Migration](lance-migration.md)
- [Blobs and Multimodal Embeddings](blobs.md)
