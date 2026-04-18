# CLAUDE.md — spike-intel

Scoped guidance for the `templates/spike-intel/` starter. Repo-wide conventions live in `../../CLAUDE.md`; nanograph operational conventions live in the `nanograph-ops` skill.

## What This Is

A nanograph schema + seed modeling AI/ML industry intelligence using the SPIKE framework. Schema, seed data, and queries only — no application code.

## Key Files

- `schema.pg` — Executable nanograph schema. Source of truth.
- `README.md` — Reference seed description, schema essentials, quick start.
- `seed.md` / `seed.jsonl` — Seed dataset (human-readable / loadable).
- `queries/*.gq` — Read and mutation queries.
- `nanograph.toml` — Project config with query aliases.
- `nanograph-intel-bootstrap/` — Co-located agent skill for bootstrapping new-domain SPIKE graphs.

## Domain Model

**SPIKE Nodes:** Signal, Element, Pattern, Insight, KnowHow
**Supportive:** Company, SourceEntity, Expert, InformationArtifact, Chunk

**Core analytical loop:** Signals form or contradict Patterns. Patterns drive or rely on other Patterns. Everything else supports this loop or maps the domain.

**Design choices to preserve:**

- Flat `kind` enums on Element and Pattern — no interfaces or subtypes
- ElementKind: `product, technology, framework, concept, ops`
- PatternKind: `challenge, disruption, dynamic`
- Domain is an enum property on Signal/Element, not a node
- Edges follow `VerbTargetType` naming (e.g. `FormsPattern`, `DevelopedByCompany`)
- Embeddings only on Chunk: `Vector(3072) @embed(text)`
- Chunk has a synthetic `slug @key` (nanograph requires `@key` on every node)

## Validation

```bash
nanograph lint --schema schema.pg --query queries/signals.gq
```

Lint after every `.pg` or `.gq` edit.

## When Editing

- Use `@rename_from(...)` on property/type renames for migration support
- Keep `README.md` in sync with `schema.pg`
- Prefer semantic edge names over generic ones (`Enables` not `RelatedTo`)
- Use the narrowest type that fits (enums over strings, `DateTime` over `String`)
- Required vs optional is deliberate — don't add `?` without reason

## Porting Notes (vs Omnigraph original)

- `@embed("text")` → `@embed(text)` (unquoted is nanograph-canonical; both parse)
- `@card(0..1)` / `@card(1..1)` — **nanograph does not support edge cardinality annotations.** The parser tolerates any `@name("string")` on an edge but enforces nothing. Constraints like "at most one source publisher" or "exactly one Chunk→Artifact" are now application-level (enforce in code/queries that emit edges).
- `@unique(src)` inside edge body is unsupported
- `Chunk` gained a `slug @key` — nanograph requires `@key` on every node
- `omnigraph.yaml` → `nanograph.toml`; `aliases:` → `[query_aliases.X]` sections
- No server, no RustFS, no AWS vars — nanograph is embedded
