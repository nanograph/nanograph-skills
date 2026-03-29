---
name: nanograph-ops
description: "Operational guide for managing nanograph embedded graph databases. Use this skill whenever the user works with nanograph files (.pg schemas, .gq queries, nanograph.toml, *.nano/ databases), asks about graph data operations, mutations, schema changes, data loading, querying, search, or any nanograph CLI command. Also trigger when you see nanograph-related files in the project or the user mentions graph databases, property graphs, or Lance datasets in the context of this codebase."
metadata:
  summary: "Operate nanograph databases — schema design, mutations, querying, search, blobs, and maintenance."
---

# NanoGraph Operations

This skill teaches you how to operate a nanograph database correctly. nanograph is an embedded local-first typed property graph DB — think SQLite for graphs. The common mistakes agents make are well-documented and entirely preventable if you follow these rules.

## Before you do anything

Run from the directory where `nanograph.toml` lives. nanograph resolves config from the current working directory and does not walk parent directories. If you get "database path is required", you're in the wrong directory.

Inspect the database before operating on it:

```bash
nanograph describe --format json
nanograph doctor
```

`describe --format json` gives you type names, properties, key properties, edge summaries, row counts, and `@description`/`@instruction` annotations. This is everything you need to construct correct queries and mutations. Do not guess the schema from JSONL files or query output.

## Data changes: mutations, not reingestion

This is the most important rule. When you need to change a few records, use mutation queries. Do not export the graph as JSONL, edit it, and `load --mode overwrite`. Reingestion destroys CDC history, invalidates Lance dataset versions, and is far slower than a targeted mutation.

```bash
# Right
nanograph run --query queries.gq --name update_status --param slug=cli-acme --param status=healthy

# Wrong
nanograph export --format jsonl > dump.jsonl && edit dump.jsonl && nanograph load --data dump.jsonl --mode overwrite
```

Use each operation for its intended purpose:

| Goal | Command |
|------|---------|
| Add one record | `insert` mutation query |
| Update a field | `update` mutation query (needs `@key`) |
| Remove a record | `delete` mutation query (cascades edges) |
| Initial bulk load | `nanograph load --mode overwrite` |
| Add batch of new records | `nanograph load --mode append` |
| Reconcile external data | `nanograph load --mode merge` (needs `@key`) |

## Writing mutations

All mutations must be parameterized. Never hardcode slugs, dates, or values — hardcoded mutations are single-use throwaway code.

```
// Right — reusable
query update_status($slug: String, $status: String) {
    update Client set { status: $status } where slug = $slug
}

// Wrong — single-use, will clutter the .gq file
query advance_stage() {
    update Opportunity set { stage: "won" } where slug = "opp-stripe-migration"
}
```

Design mutations alongside the schema. When you create a node type, immediately write its `insert`, `update`, and `delete` mutations plus edge-linking mutations. This gives a safe, tested interface from day one.

Key constraints:
- `update` requires `@key` on the target node type
- Use `now()` for timestamps in mutations — it resolves to current UTC once per execution
- Array params cannot be passed via `--param` — hardcode list values in the query body
- Edge inserts use `from:`/`to:` with `@key` values: `insert SignalAffects { from: $signal, to: $client }`
- Edge names in queries use camelCase: schema `HasMentor` becomes `hasMentor` in queries

## Schema design rules

Put `@key` on every node type. Without it, `update` mutations won't work, edge JSONL can't resolve `from`/`to`, and `load --mode merge` has nothing to match on. Use stable human-readable slugs over UUIDs.

```
node Client {
    slug: String @key
    name: String @unique
    status: enum(active, churned, prospect)
}
```

Add `@description("...")` to types, properties, and queries. Add `@instruction("...")` to types and queries (not properties). These surface in `describe --format json` and `run` output, giving agents structured context about what things mean and how to use them.

## Aliases

Define aliases in `nanograph.toml` for every operation agents will use. Aliases turn fragile multi-flag commands into short positional calls.

```toml
[query_aliases.search]
query = "queries.gq"
name = "semantic_search"
args = ["q"]
format = "json"

[query_aliases.status]
query = "queries.gq"
name = "update_task_status"
args = ["slug", "status"]
format = "jsonl"
```

```bash
nanograph run search "renewal risk"
nanograph run status task-42 done
```

Always set `format` in the alias. Use `json` for reads, `jsonl` for mutations. Use short verb-like names. Put the scoping argument first.

## Output

Agents should use `--format json`, `--format jsonl`, or `--format kv`, never parse `table` output. For non-run commands, use `--json`:

```bash
nanograph describe --json
nanograph migrate --dry-run --json
nanograph doctor --json
```

For `nanograph run`, `json` wraps output in `{metadata, rows}` and `jsonl` emits a metadata header record first — both carry query `@description` and `@instruction`. Use `--quiet` to suppress human-readable output while keeping machine formats.

## Post-change workflow

After editing `.pg` or `.gq` files, follow this sequence every time:

1. Edit `.pg` and/or `.gq`
2. `nanograph check --query queries.gq`
3. If schema changed: `nanograph migrate` (preview with `--dry-run` first)
4. If `@embed(...)` fields added/changed: `nanograph embed --only-null`
5. Smoke test: `nanograph run <alias>`

Do not skip or reorder these steps.

## Schema migration

When the schema changes, use `nanograph migrate`. Do not rebuild the database from scratch — that destroys all mutations applied since the last seed.

- Adding a nullable property: safe, auto-applied
- Adding a new type: safe, auto-applied
- Renames: use `@rename_from("old_name")`
- Adding non-nullable property to populated type: blocked — make it nullable
- Dropping properties: requires `--auto-approve`
- Adding enum values: works immediately without migration

Preview with `nanograph schema-diff --from old.pg --to new.pg` or `nanograph migrate --dry-run`.

If migration fails and leaves a stale journal (`*.migration.journal.json` in PREPARED state), delete the journal before retrying.

## Embedding providers

nanograph supports OpenAI and Gemini embedding providers. Set in `nanograph.toml`:

```toml
[embedding]
provider = "openai"   # or "gemini" or "mock"
```

Put the API key in `.env.nano`:

```bash
OPENAI_API_KEY=sk-...
# or
GEMINI_API_KEY=...
```

Defaults: `text-embedding-3-small` (OpenAI, 1536 dims) and `gemini-embedding-2-preview` (Gemini, 3072 dims). Both support dimensionality reduction — nanograph passes the schema's `Vector(dim)` to the API automatically.

If no provider is set, nanograph auto-detects: Gemini when only `GEMINI_API_KEY` is present, OpenAI otherwise.

After switching providers, re-embed everything (not `--only-null`) because old vectors came from a different model:

```bash
nanograph embed --reindex
```

## Search

nanograph supports lexical search, semantic search, and hybrid ranking — all in the query language.

### Text search predicates

Use these in the `match` block as filters:

| Predicate | Behavior |
|-----------|----------|
| `search($c.bio, $q)` | Token match — all query tokens must be present |
| `fuzzy($c.bio, $q)` | Typo-tolerant approximate match |
| `match_text($c.bio, $q)` | Phrase / contiguous token match |

### Semantic search

`nearest($c.embedding, $q)` returns cosine distance (lower = closer). Requires `limit`.

### Ranked and hybrid search

Use scoring functions in `return` and `order`:

- `bm25($c.bio, $q)` — lexical relevance score (higher = better, use `desc`)
- `nearest($c.embedding, $q)` — cosine distance (lower = closer, ascending default)
- `rrf(nearest(...), bm25(...))` — hybrid fusion (higher = better, use `desc`)

```
query hybrid_search($q: String) {
    match { $c: Character }
    return {
        $c.slug,
        rrf(nearest($c.embedding, $q), bm25($c.bio, $q)) as score
    }
    order { rrf(nearest($c.embedding, $q), bm25($c.bio, $q)) desc }
    limit 10
}
```

### Graph-constrained search

Scope with traversal first, then rank — this is nanograph's core strength:

```
query client_signals($client: String, $q: String) {
    match {
        $c: Client { slug: $client }
        $s signalAffects $c
    }
    return { $s.slug, nearest($s.embedding, $q) as score }
    order { nearest($s.embedding, $q) }
    limit 5
}
```

After adding `@embed(...)` fields, backfill vectors: `nanograph embed --only-null`. Use `--reindex` if the field has `@index`.

## Blobs and media

nanograph stores media as normal nodes with external URIs — not as blob bytes inside `.nano/`. This keeps the database small and Git-friendly while supporting text-to-image search.

### Schema pattern

```
node PhotoAsset {
    slug: String @key
    uri: String @media_uri(mime)
    mime: String
    embedding: Vector(768)? @embed(uri) @index
}

edge HasPhoto: Product -> PhotoAsset
```

`@media_uri(mime)` marks the field as an external media URI and names the sibling mime property. `@embed(uri)` on a `@media_uri` field triggers multimodal (image) embedding, not text embedding.

### Loading media

Supported input formats in JSONL:

| Format | Behavior |
|--------|----------|
| `@file:path/to/image.jpg` | Imports into media root, stores `file://` URI |
| `@base64:...` | Decodes and imports into media root |
| `@uri:https://example.com/img.jpg` | Stores URI directly, no copy |
| `@uri:file:///path/img.jpg` | Stores URI directly |

```json
{"type": "PhotoAsset", "data": {"slug": "space", "uri": "@file:photos/space.jpg"}}
```

### Multimodal provider

Gemini is the practical multimodal provider today. OpenAI embeddings are text-only in nanograph.

```toml
[embedding]
provider = "gemini"
```

Current limits: `image/png` and `image/jpeg` only, max 20 MB inline, 6 images per batch. `s3://` URIs are not directly embeddable — use `@file:` import, presigned HTTPS URLs, or import first.

For detailed patterns and provider migration, see `references/blobs.md`.

## JSONL format

Nodes use `"type"` + `"data"`. Edges use `"edge"` + `"from"` + `"to"`:

```json
{"type": "Client", "data": {"slug": "cli-acme", "name": "Acme Corp"}}
{"edge": "SignalAffects", "from": "sig-renewal", "to": "cli-acme"}
```

Do not use `type` for edges. Do not use `src`/`dst`. `from`/`to` values are `@key` values, not internal IDs. Type and edge names in JSONL match schema PascalCase exactly.

## Git and secrets

Commit `nanograph.toml`, `schema.pg`, `queries.gq`, seed `.jsonl`, and `*.nano/` (if under ~50 MB). The database is not regenerable from seed once mutations have been applied.

Never commit `.env.nano` — it holds secrets. Add it to `.gitignore` before the first commit.

## Maintenance

After many mutations, compact periodically — not after every mutation:

```bash
nanograph compact
nanograph cleanup --retain-tx-versions 50
nanograph doctor
```

Keep the CLI up to date: `brew upgrade nanograph`. Check version first when something behaves unexpectedly.

## Lance storage format

nanograph 1.x links against Lance v3, which writes new datasets in storage format 2.2. Databases created with earlier versions use format 2.0 — both remain readable. Format 2.2 provides better compression for text, enums, dates, and vector columns. Check with `nanograph doctor --verbose`. There is a dedicated migration path for upgrading 2.0 to 2.2 — back up first.

## Deep reference

For detailed examples, edge cases, and the full common-mistakes table, read `references/best-practices.md`.
