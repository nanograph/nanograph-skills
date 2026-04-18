---
name: nanograph-industry-intel
description: 'Stand up a SPIKE industry intelligence graph on nanograph. Use this skill whenever the user wants to set up a SPIKE-style knowledge graph — either using the existing AI industry example or designing the ontology for another domain (Biotech, Fintech, Space, Crypto, Geopolitics, or anything else the user brings). The skill focuses on ontology and schema: it either reuses the AI template or collaborates with the user to design a domain-specific ontology (entity kinds, sub-fields, actor roles, source/artifact types) following the AI pattern, then adapts the schema and initializes the database. Data loading, research, and ingestion are explicitly out of scope — use the nanograph-ops skill for those. Apply when the user says any of: set up SPIKE, bootstrap an intel graph, create a knowledge graph for X industry, initialize intel for Y, start a new context graph, set up a graph for AI / biotech / fintech / space / crypto / geopolitics / <any domain>.'
license: MIT
metadata:
  version: "0.3.0"
  upstream: Ported from omnigraph-starters/skills/omnigraph-intel-bootstrap
---

# nanograph industry intel — SPIKE schema bootstrap

Stand up a SPIKE knowledge-graph schema on nanograph. One worked ontology ships pre-built (AI industry intel); for anything else the user brings, design the ontology together using the AI set as a pattern. This skill is **ontology-focused**: data ingestion, research, and seed generation are handled by the `nanograph-ops` skill, not here.

- **AI path** — use the template as-is (schema + real 2026 seed data). Ready to query in under a minute.
- **Custom path** — collaborate with the user to design domain-specific enums (entity kinds, sub-fields, actor roles, source/artifact types), adapt `schema.pg`, initialize an empty database. Works for Biotech, Fintech, Space, Crypto, Geopolitics, or any other domain.
- Ask the user for 2–3 example source types they already read — just enough to shape `SourceEntity.type` and `artifactType`. Not a full source list, not an ingestion plan.
- Two reference files: [`references/domain-examples.md`](references/domain-examples.md) explains the AI ontology as a worked pattern; [`references/schema-adaptation.md`](references/schema-adaptation.md) covers keep-vs-change rules and nanograph syntax gotchas.

## Prerequisites

```bash
brew install nanograph          # or see https://nanograph.io
nanograph version               # needs 1.2.0+
```

Semantic search needs an API key in `.env.nano` (`OPENAI_API_KEY` or `GEMINI_API_KEY` — Gemini also covers multimodal).

## Step 1: Ask which path

> Do you want to:
> - **AI** — use the AI industry template as-is (109 nodes, 154 real signals from early 2026)
> - **Custom** — design a schema for a different domain (I'll walk through the ontology with you)

## Path A — AI (pre-built)

```bash
cp -r <repo>/templates/spike-intel <destination>/spike-intel
cd <destination>/spike-intel
cp .env.nano.example .env.nano     # add an API key
nanograph init
nanograph load --data seed.jsonl --mode overwrite
```

`<repo>` is the `nanograph-skills` checkout (or the install path if the skill came from `npx skills add`).

Verify:

```bash
nanograph run patterns disruption           # returns SaaSpocalypse, Sovereign AI
nanograph run pattern-signals pat-sovereign-ai
```

Done. For day-to-day ops, point the user at the `nanograph-ops` skill.

## Path B — Custom Domain

No pre-baked ontology sets. The AI schema at `templates/spike-intel/schema.pg` is the pattern; the target-domain enums come from a conversation with the user. Read [`references/domain-examples.md`](references/domain-examples.md) first — it explains *why* each AI enum was chosen, which is the mental model you'll use to design the new one.

### Step 1: Confirm domain + slug

Pick a short slug for the project folder and database directory (e.g. `bio-intel`, `fin-intel`, `space-intel`, `crypto-intel`, `geo-intel`, `climate-intel`). This becomes `<slug>/` and `<slug>.nano`.

### Step 2: Design the ontology with the user

Six enum families change per domain. Walk through them in order, proposing values and having the user confirm before moving on. Start from the AI set as a mental anchor — do **not** hand the user a ready-made list for their domain.

1. **`Element.kind`** — the things signals are about. Ask: *"What kinds of things do your signals talk about? 4–7 types."*
2. **`Signal.domain` + `Element.domain`** — verticals or sub-fields. Ask: *"What sub-fields or verticals matter?"* 5–8 values. Must be identical on both Signal and Element (nanograph inlines enums).
3. **`Company.type`** — ecosystem roles. Ask: *"What roles do organizations play in your space?"* 4–7 values. Each company picks one primary role.
4. **`SourceEntity.type`** — derived from source examples (see Step 3).
5. **`InformationArtifact.artifactType`** — derived from source examples (see Step 3).
6. **Kind-specific `Element` properties** — Ask: *"For each kind, what 2–4 properties matter most?"* Swap AI's kind-specific fields (`repository`, `use_cases`, etc.) for the domain's.

Keep `Pattern.kind` (`challenge, disruption, dynamic`) unchanged — intentionally abstract, works everywhere.

### Step 3: Ask for source-type examples (to shape types 4 and 5)

Ask for **2–3 example source types** the user already reads. The goal is to choose `SourceEntity.type` and `artifactType` values — not to build a source list or plan ingestion.

> To tune the schema, give me a few example source types you'd pull from — newsletters, publications, filings, podcasts, data feeds, whatever.

Their answers map directly to enum values. Examples of the mapping, not the answer:

- *"FDA filings and clinicaltrials.gov"* → `artifactType` includes `fda-filing` and `trial-registration`; `SourceEntity.type` gains `regulatory-filing`
- *"FCC filings and launch reports"* → `artifactType` includes `filing` and `launch-report`; `SourceEntity.type` gains `agency-release`; `Company.type` needs `agency`
- *"Governance forum posts and on-chain data"* → `SourceEntity.type` adds `governance-forum` and `on-chain-data`; `artifactType` adds `proposal` and `governance-vote`

Do not elicit a full source list or plan the ingestion pipeline — that's out of scope for this skill.

### Step 4: Present proposed enums for confirmation

Before editing any files, echo the proposed enum block back to the user:

```
Element.kind:      enum(...)
Signal.domain:     enum(...)
Element.domain:    enum(...)          // same as above
Company.type:      enum(...)
SourceEntity.type: enum(...)
InformationArtifact.artifactType: enum(...)

// Kind-specific Element properties:
...
```

Cheap to revise here; expensive once queries and data start flowing.

### Step 5: Copy and adapt the template

```bash
cp -r <repo>/templates/spike-intel <slug>
rm <slug>/seed.jsonl <slug>/seed.md      # user brings their own data via nanograph-ops
cd <slug>
```

Edit `<slug>/schema.pg`:
- Replace the six enum families with the confirmed values from Step 4
- Swap kind-specific Element properties
- Leave `Pattern.kind` alone

Update `<slug>/nanograph.toml`:

```toml
[project]
name = "<Domain> Intel — SPIKE"

[db]
default_path = "<slug>.nano"
```

See [`references/schema-adaptation.md`](references/schema-adaptation.md) for the full keep-vs-change rules and nanograph syntax gotchas (`@embed`, no edge cardinality, `@key` on every node, etc.).

### Step 6: Lint + init

```bash
nanograph lint --schema schema.pg --query queries/signals.gq
cp .env.nano.example .env.nano           # add an API key
nanograph init
```

The database is now empty with the adapted schema. The user's next step — **loading data** — happens via the `nanograph-ops` skill (mutation queries for individual records, `nanograph load --mode merge` for batches). This skill stops here.

### Step 7: Hand off

Tell the user:

- Schema designed for `<domain>` at `<slug>/schema.pg`
- Empty database at `<slug>/<slug>.nano`
- Aliases ready in `nanograph.toml` (they'll work once data is loaded)
- Next: use the `nanograph-ops` skill to add data via mutations or `load`

## nanograph Syntax Quick Reference

| Concern | nanograph |
|---|---|
| `@embed(prop)` | Unquoted property name is canonical; `@embed("prop")` also parses |
| `@card(...)` | Tolerated but **not enforced** — don't rely on it |
| `@unique(src)` in edge body | Unsupported |
| `@key` on every node | Required — `Chunk` gets a synthetic `slug @key` |
| Property separators | Newlines only; commas rejected |
| Comments | `//`, not `#` |

## References

| Reference | When to load |
|-----------|--------------|
| [`references/domain-examples.md`](references/domain-examples.md) | Reading the AI ontology as a worked pattern; designing for a new domain |
| [`references/schema-adaptation.md`](references/schema-adaptation.md) | Keep-vs-change rules, nanograph syntax gotchas, validation checklist |

For everything beyond schema setup — mutations, data loading, CDC, embeddings, maintenance — use the `nanograph-ops` skill.
