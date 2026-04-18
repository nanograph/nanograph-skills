# Schema and query authoring

How to design, annotate, and evolve a nanograph schema — inspection commands, the annotations agents rely on, and the migrate workflow for schema changes.

## Contents

- [Start with `describe` and `doctor`](#start-with-describe-and-doctor)
- [Use `@description` and `@instruction`](#use-description-and-instruction)
- [Always typecheck after editing `.gq` or `.pg`](#always-typecheck-after-editing-gq-or-pg)
- [Migrate, don't rebuild](#migrate-dont-rebuild)

## Start with `describe` and `doctor`

Before writing queries, running mutations, or ingesting data, inspect the database:

```bash
nanograph describe --format json
nanograph describe --type Signal --format json
nanograph doctor
```

`describe --format json` returns type names, properties and their types, key properties, edge summaries, row counts, and `@description` / `@instruction` annotations. Agents that skip `describe` and try to infer the schema from output or JSONL make systematic mistakes.

`doctor` is the health check. Use `doctor --schema` to compare the DB against a desired schema file, and `doctor --verbose` to see per-dataset Lance storage format versions.

## Use `@description` and `@instruction`

These annotations do not change execution, but they give agents structured intent through `describe` and `run`:

```graphql
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

Write these annotations when creating the schema or query. They cost nothing and prevent agent misuse.

## Always typecheck after editing `.gq` or `.pg`

No query or schema edit is complete until `lint` passes:

```bash
nanograph lint --query queries.gq
```

This catches wrong property names, type mismatches, invalid traversals, missing parameters, and malformed mutations.

When the schema changed, also preview the migration plan:

```bash
nanograph migrate --dry-run
nanograph schema-diff --from old.pg --to new.pg
```

## Migrate, don't rebuild

When the schema changes, update the schema file and run `nanograph migrate`. Do not rebuild from scratch unless the seed is complete and current DB state does not matter.

```bash
# Right
nanograph migrate

# Wrong — destroys all mutations applied since last seed
nanograph init --schema new-schema.pg
nanograph load --data seed.jsonl --mode overwrite
```

Migration rules:

- Adding a nullable property is safe — auto-applied.
- Adding a node or edge type is safe — auto-applied.
- Renames require `@rename_from("old_name")`.
- Adding a non-nullable property to a populated type is blocked.
- Dropping properties requires `--auto-approve`.
- Adding enum values works immediately without migration.

If a migration fails and leaves a stale journal in `PREPARED` state, delete the journal before retrying.

If the project keeps the desired schema outside the DB directory, configure `schema.default_path` so `migrate`, `lint`, and `doctor --schema` all refer to the same file.
