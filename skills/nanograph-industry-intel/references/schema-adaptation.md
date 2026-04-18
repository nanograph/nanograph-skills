# Schema Adaptation

What to **keep** vs **change** when adapting the `spike-intel` schema for a new domain. The AI schema is the only pre-built example — see [`domain-examples.md`](domain-examples.md) for its enum set and the reasoning behind each choice, then collaborate with the user to design the target domain's enums.

## What Stays (SPIKE Invariants)

Changing any of these means you're not doing SPIKE anymore.

**Node types** (10):
- SPIKE: `Signal`, `Pattern`, `Insight`, `KnowHow`, `Element`
- Supportive: `Company`, `SourceEntity`, `Expert`, `InformationArtifact`, `Chunk`

If a new domain seems to want a new node type (e.g., `DAO` for crypto, `Mission` for space), prefer modeling it as `Element.kind = dao` or `Element.kind = mission`. Resist adding node types — the SPIKE core works better narrow.

**Analytical edges** — the graph skeleton:
- `FormsPattern`, `ContradictsPattern` (Signal → Pattern)
- `DrivesPattern`, `ReliesOnPattern`, `ContradictsToPattern` (Pattern → Pattern)
- `HighlightsPattern` (Insight → Pattern)
- `OnElement` (Signal → Element)
- `EnablesElement`, `UsesElement` (Element → Element)
- `ExemplifiesPattern`, `EnablesPattern` (Element → Pattern)
- `ReferencesElement` (KnowHow → Element)

**Provenance edges**:
- `DevelopedByCompany`, `RelevantCompany`, `AffiliatedWithCompany`
- `SpottedInArtifact`, `IdentifiedInArtifact`, `SourcedFromArtifact`, `SourcedFromSource`
- `PublishedBySource` (one publisher per artifact — enforced in loader code, not schema)
- `ContributedByExpert`
- `PartOfArtifact` (each Chunk belongs to exactly one artifact — enforced in loader code)

**Fixed properties**:
- `slug: String @key` on every node (including `Chunk` — nanograph requires it)
- `stagingTimestamp`, `createdAt`, `updatedAt` where they are
- `embedding: Vector(3072) @embed(text) @index` on Chunk
- Slug prefixes: `sig-`, `pat-`, `el-`, `ins-`, `how-to-`, `co-`, `exp-`, `ia-`, `source-`

**`Pattern.kind` enum**: `challenge, disruption, dynamic` — intentionally abstract, works across all domains. Only change if the user has a strong reason.

## What Changes (Domain-Specific)

Six enum families vary by domain. For anything other than AI, design these values in conversation with the user — use the AI set in [`domain-examples.md`](domain-examples.md) as a mental pattern, not a template to fill in.

1. **`Element.kind`** — the thing-types in this domain. Rule of thumb: 4–7 kinds. Fewer = coarse and useless for filtering; more = overfit.
2. **`Signal.domain` + `Element.domain`** — sub-fields or verticals. Must be declared **identically on both** Signal and Element — nanograph inlines enums, so shared enums must be duplicated.
3. **`Company.type`** — ecosystem roles. Each company picks one primary role.
4. **`SourceEntity.type`** — how sources publish in this domain. Derive from the source-type examples the user gives.
5. **`InformationArtifact.artifactType`** — formats the domain generates. Derive from the source-type examples the user gives.
6. **Kind-specific `Element` properties** — for each `Element.kind`, ask the user which 2–4 properties matter most. Swap the AI-specific ones (`repository`, `use_cases`, etc.) for the domain's.

## Using source examples to shape enums

Ask the user for 2–3 representative source types they'd pull from — not a full source list, just enough to sanity-check the schema. Their answer tells you which enum values to keep or add:

- *"FDA filings and trial-registration"* → `artifactType` must include `fda-filing` and `trial-registration`; `SourceEntity.type` needs `regulatory-filing`
- *"FCC filings and launch reports"* → `artifactType` includes `filing` and `launch-report`; `Company.type` needs `agency`
- *"Governance forum posts and on-chain data"* → `SourceEntity.type` adds `governance-forum` and `on-chain-data`; `artifactType` adds `proposal` and `governance-vote`

This is the only purpose for asking about sources. Do not elicit a full source list or plan the ingestion pipeline — that's out of scope for this skill.

## nanograph Syntax Reminders

When editing `schema.pg`:

- `@embed(prop)` — unquoted is canonical (both `@embed(text)` and `@embed("text")` parse)
- **No edge cardinality.** nanograph tolerates `@card("...")` but enforces nothing. Don't carry over Omnigraph's `@card(0..1)` / `@card(1..1)` — enforce those in the code that emits edges.
- `@unique(src)` in edge body is unsupported
- Every node type needs `@key` — even `Chunk`. Use a synthetic slug like `{artifact_slug}-c{index}` if needed.
- Comments use `//`, not `#`
- Properties in a node/edge body are newline-separated, not comma-separated

## Update nanograph.toml

After schema changes, update `<slug>/nanograph.toml`:

```toml
[project]
name = "<domain> Intel — SPIKE"

[db]
default_path = "<slug>.nano"
```

Aliases referring to enum values (e.g., `patterns disruption`) still work since `Pattern.kind` is unchanged.

## Validate

After every schema edit:

```bash
nanograph lint --schema schema.pg --query queries/signals.gq
```

The queries themselves usually don't need changes — they operate on slugs, not enum values. Lint will flag anything broken.

## Checklist

- [ ] Copied `templates/spike-intel/` to `<slug>/`
- [ ] Removed template's `seed.jsonl` and `seed.md` (the user brings their own data later)
- [ ] Designed and confirmed `Element.kind` with the user
- [ ] Designed and confirmed `domain` enum (identical on `Signal` and `Element`)
- [ ] Designed and confirmed `Company.type`, `SourceEntity.type`, `artifactType`
- [ ] Replaced kind-specific `Element` properties for the domain
- [ ] Kept `Pattern.kind` as-is
- [ ] Updated `nanograph.toml` project name + `db.default_path`
- [ ] `nanograph lint` passes with 0 errors
