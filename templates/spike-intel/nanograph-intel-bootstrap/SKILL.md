---
name: nanograph-intel-bootstrap
description: 'Bootstrap a new nanograph-based SPIKE industry intelligence graph from scratch. Use this skill whenever a user wants to set up a new SPIKE graph — either with the existing AI industry demo data or for a new domain (biotech, fintech, crypto, geopolitics, macroeconomics, SaaS, climate tech, etc.). The flow presents a demo-vs-custom decision, then for custom setups asks about domain scope, actors, cadence, and sources, adapts schema and enums for the target domain, runs initial web research to generate real seed content, and executes init + load. Apply aggressively when the user says any of: set up nanograph SPIKE, bootstrap a new intel graph, create a new SPIKE starter, I want to track X industry, initialize intel for Y, new graph for Z domain, start a new context graph, or similar phrasing. This skill takes a user from zero to a populated, queryable graph.'
license: MIT
metadata:
  version: "0.1.0"
  upstream: Ported from omnigraph-starters/skills/omnigraph-intel-bootstrap
---

# SPIKE Starter Bootstrap (nanograph)

This skill takes a user from zero to a populated, queryable SPIKE graph on nanograph. Two paths:

- **Demo** — use the existing `spike-intel` template (AI/ML signals, early 2026). Ready to query in under a minute.
- **Custom** — set up a new domain (biotech, crypto, fintech, geopolitics, etc.). Takes ~30–60 minutes including research and user review.

nanograph is embedded and local-first — no Docker, no RustFS, no S3, no HTTP server. The database is a plain `*.nano/` directory. This makes bootstrap much simpler than the Omnigraph equivalent.

## Prerequisites

Only one thing to install:

```bash
brew install nanograph          # or see https://nanograph.io
command -v nanograph && nanograph version
```

Verify the version is 1.2.0 or later (earlier versions lack `NamespaceLineage` storage).

An embedding API key is needed for `@embed(...)` fields and semantic search. One of:

```bash
OPENAI_API_KEY=sk-...         # text-only
GEMINI_API_KEY=...            # text + multimodal (images, audio, video, PDF)
```

These go in `.env.nano` (see Step 2 of either path). The file is auto-loaded from the working directory.

## Step 1: Ask the user which path

> Do you want to:
> - **Demo** — set up the AI industry intel demo (5 patterns, 15 signals, ~110 nodes, ready to query in ~30 seconds)
> - **Custom** — set up a graph for a new domain (I'll ask about your domain + sources, adapt the schema, research real seed data, and wire it up)

Branch based on the answer.

## Path A: Demo Setup

The `spike-intel/` template ships alongside this skill. Copy it into the user's project directory (or wherever they want the graph to live):

```bash
# Pick a destination — often the user's current project, or ~/graphs/spike-intel
cp -r <path-to>/templates/spike-intel <destination>/spike-intel
cd <destination>/spike-intel
```

If the agent is already running inside a checkout of the `nanograph-skills` repo, `<path-to>` is the repo root.

Then set up the API key and initialize:

```bash
cp .env.nano.example .env.nano
# Edit .env.nano — put OPENAI_API_KEY or GEMINI_API_KEY

nanograph init
nanograph load --data seed.jsonl --mode overwrite
```

Expected load output:

```
loaded with overwrite: 109 nodes across 10 node types, 154 edges across 23 edge types
```

Verify with an alias:

```bash
nanograph run patterns disruption
nanograph run pattern-signals pat-sovereign-ai
```

The first returns 2 patterns (SaaSpocalypse, Sovereign AI); the second returns 3 signals.

**After setup**, point the user at the `nanograph-ops` skill for day-to-day operations.

## Path B: Custom Domain Setup

Six phases, in order. Each phase's output feeds the next — don't skip ahead.

### Phase 1 — Domain identification

Ask the user which domain they want to track. Present these as options:

- Biotech / therapeutics
- Fintech / financial services
- Crypto / web3
- Geopolitics / policy
- Macroeconomics
- SaaS / enterprise software
- Climate tech / energy transition
- Other (user specifies)

Then narrow:
- Scope: "all of X" or "only Y within X"?
- Global or regional?

Capture a **project slug** for the new starter: `bio-intel`, `crypto-intel`, `geo-intel`, etc. This becomes the folder name and the database name (`<slug>.nano`).

### Phase 2 — Key questions

Ask each in turn. See [`references/custom-domain.md`](references/custom-domain.md) for full phrasing and multi-select options.

- **Actors to track** (multi-select): companies, labs, regulators, individuals, protocols, investors
- **Time horizon**: recent only (3mo), medium (12mo), or full historical
- **Update cadence**: daily, weekly, monthly, ad-hoc
- **Primary consumer**: human analysts, internal dashboard, AI agents, mixed

### Phase 3 — Sources (most important)

Sources are the lifeblood of a SPIKE graph. The quality of the output is bounded by the quality of the sources. Spend time here.

Ask in order (see [`references/custom-domain.md`](references/custom-domain.md) for exact wording):

1. **Primary reading list** — newsletters, blogs, publications the user already reads (free-form, 5–15 entries)
2. **Priority analysts / experts** — 3–10 people whose takes should be first-class entities
3. **Regulatory / authoritative sources** — governmental, self-regulatory (FDA, SEC, IMF, etc.)
4. **Academic / primary sources** — journals, preprint servers, research aggregators
5. **Social / community** — X accounts, podcasts, forums

### Phase 4 — Confirm summary

Before making changes, echo what you captured back to the user:

- Domain + scope + project slug
- Actor types to track
- Horizon + cadence + consumer
- Source list (grouped by category)

Write this to `<slug>/setup-notes.md` in the new starter folder. Confirming now is cheap; rework later isn't.

### Phase 5 — Adapt the schema

Copy the `spike-intel` template into `<slug>/`:

```bash
# <template> is the path to templates/spike-intel/ alongside this skill
cp -r <template> <slug>
rm <slug>/seed.jsonl <slug>/seed.md   # regenerated in Phase 6
rm -rf <slug>/nanograph-intel-bootstrap  # the skill doesn't belong in the user's project
```

Update in `<slug>/schema.pg`:

- `Element.kind` enum — replace with domain-appropriate kinds
- `Signal.domain` / `Element.domain` enum — replace with domain slices (must be identical on both)
- `Company.type` enum — match the ecosystem
- `SourceEntity.type` enum — match how sources publish
- `ArtifactType` enum — include domain-relevant formats
- Kind-specific Element properties (biotech wants `phase`, `moa`; crypto wants `chain`, `token_symbol`; etc.)

Update in `<slug>/nanograph.toml`:

- `[db].default_path` → `"<slug>.nano"`
- `[project].name` → domain-appropriate name
- `[project].description` / `[project].instruction` — domain-specific

**Pattern.kind** (`challenge`, `disruption`, `dynamic`) is usually domain-agnostic. Don't change it unless the user has strong reasons.

See [`references/schema-adaptation.md`](references/schema-adaptation.md) for the full keep-vs-change rules. See [`references/domain-examples.md`](references/domain-examples.md) for worked examples across biotech, crypto, fintech, geopolitics, and macro.

After editing:

```bash
cd <slug>
nanograph lint --schema schema.pg --query queries/signals.gq
```

Fix any lint errors before moving on.

### Phase 6 — Research, seed, init, load

Use web research to build real seed content. **Do not fabricate signals or dates.** See [`references/research.md`](references/research.md) for the workflow. High-level:

1. For each source from Phase 3, pull recent items (WebFetch / WebSearch)
2. Extract candidate signals (dated, URL-backed, specific)
3. Cluster into 3–5 patterns (recurring themes)
4. For each pattern, identify the Elements, Companies, Experts mentioned
5. Write `<slug>/seed.md` (tabular, human-readable) — **present this to the user for review before generating JSONL**
6. Generate `<slug>/seed.jsonl` from the confirmed seed.md
7. Initialize and load:

```bash
cd <slug>
cp .env.nano.example .env.nano
# Add API key to .env.nano

nanograph init
nanograph load --data seed.jsonl --mode overwrite
```

8. Verify:

```bash
nanograph describe --json
nanograph run patterns <pattern-kind>   # or another alias
```

### Phase 7 — Hand-off

Tell the user:

- What got created (starter folder, schema, seed counts, `.nano` path)
- How to query (via aliases defined in `nanograph.toml`)
- To use the `nanograph-ops` skill for day-to-day ops (adding signals, schema evolution, CDC, embeddings)

## nanograph Schema Syntax Notes

A few gotchas when editing `schema.pg` — they differ from Omnigraph:

| Concern | nanograph |
|---|---|
| `@embed(prop)` | Unquoted property name is canonical; `@embed("prop")` also parses |
| `@card(...)` edge cardinality | **Not enforced.** The parser tolerates `@name("string_lit")` on edges but attaches no semantics. Treat cardinality as application-level (enforce in the code/queries that emit edges, not in the schema). |
| `@unique(src)` in edge body | Unsupported. No schema-level way to enforce per-source uniqueness today. |
| `@key` on every node | Required. Every node type needs one (the `Chunk` node has a synthetic `slug @key`). |
| Property separators | Newlines only — commas between properties are rejected. |
| Comments | `//` (not `#`) |

## Deep Dives

Load these only when you reach the relevant phase.

| Reference | When to load |
|-----------|--------------|
| [`references/custom-domain.md`](references/custom-domain.md) | Phases 1–4: elicitation question bank and source patterns |
| [`references/schema-adaptation.md`](references/schema-adaptation.md) | Phase 5: what stays vs changes in the schema |
| [`references/domain-examples.md`](references/domain-examples.md) | Phase 5: ready-made enum sets for biotech, crypto, fintech, geopolitics, macro |
| [`references/research.md`](references/research.md) | Phase 6: web research → seed.md → seed.jsonl workflow |

For operational rules (mutations, CDC, embeddings, storage), defer to the `nanograph-ops` skill instead of duplicating them here.
