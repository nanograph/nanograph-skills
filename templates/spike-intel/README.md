# SPIKE Intel — nanograph starter

A local-first graph starter modeling AI/ML industry intelligence on the SPIKE framework. Ported from [`omnigraph-starters/industry-intel`](https://github.com/ModernRelay/omnigraph-starters) to nanograph's embedded, server-free runtime.

## SPIKE Framework

- **Signal** — a dated external fact, movement, or observation
- **Pattern** — a recurring theme formed, contradicted, or driven by signals
- **Insight** — a synthesized interpretation explaining why a pattern matters
- **KnowHow** — an actionable practice or playbook grounded in the graph
- **Element** — a concrete product, framework, company, or concept the signals are about

Supportive: `Company`, `SourceEntity`, `Expert`, `InformationArtifact`, `Chunk`.

## Core Analytical Loop

Signals and Patterns form the analytical core. Insights interpret them. Elements and KnowHows map the domain around them.

```
  Signal ── FormsPattern ──────────▶ Pattern
    │                                  │
    ├── ContradictsPattern ──────────▶ │
    │                                  │
    │                                  ├── DrivesPattern ──────▶ Pattern
    │                                  └── ReliesOnPattern ────▶ Pattern
    │
    ├── OnElement ──▶ Element
    │                   │
    │                   ├── ExemplifiesPattern ──▶ Pattern
    │                   ├── EnablesPattern ──────▶ Pattern
    │                   │
    │                   ├── EnablesElement ──▶ Element
    │                   └── UsesElement ────▶ Element
    │
  Insight ── HighlightsPattern ──────▶ Pattern
    │
    └── ReliesOnElement ─────────────▶ Element

  KnowHow ── ReferencesElement ───────▶ Element
```

## Reference Seed: AI Industry, Early 2026

Five live patterns in the AI industry:

| Pattern | Kind | What it captures |
|---------|------|------------------|
| **Sovereign AI** | disruption | Enterprises moving AI off public cloud — driven by regulation (DORA, EU AI Act) and collapsing on-prem setup cost |
| **SaaSpocalypse** | disruption | Per-seat SaaS pricing breaking as agents replace workflows — $830B wiped from S&P software index in six days |
| **Context Graphs** | dynamic | Decision traces + ontology + temporal reasoning as a new infrastructure layer above databases |
| **New Cyber Threats** | challenge | AI models autonomously exploiting vulnerabilities + agentic attack surfaces inside enterprises |
| **Accelerated Research** | dynamic | AI agents running 100s of experiments autonomously — from Karpathy loops to AlphaEvolve to AI-proven math |

Each pattern is backed by ~3 real, dated signals with source URLs.

**Totals:** 109 nodes, 154 edges across 10 node types and 23 edge types.

## Schema Essentials

**Nodes (10):** Signal, Pattern, Insight, KnowHow, Element + Company, SourceEntity, Expert, InformationArtifact, Chunk

**Enums that carry the analytical lens:**

| Enum | Values |
|------|--------|
| **PatternKind** | `challenge, disruption, dynamic` |
| **ElementKind** | `product, technology, framework, concept, ops` |
| **Domain** | `training, inference, infra, harness, robotics, security, data-eng, context` |

**Key design choices:**

- Flat node types with `kind` enums (no subtypes or interfaces)
- Domain is a property, not a node
- Edges follow `VerbTargetType` naming so direction is obvious
- `slug` is the external identity everywhere (`sig-`, `pat-`, `el-`, `ins-`, `how-to-`, `co-`, `exp-`, `ia-`, `source-`)
- Embeddings only on Chunk (`Vector(3072)` — requires `text-embedding-3-large` or Gemini `gemini-embedding-2-preview`)

Full property tables and constraints in [`schema.pg`](./schema.pg).

## Files

- `schema.pg` — Executable nanograph schema (source of truth)
- `seed.md` / `seed.jsonl` — Seed dataset (human-readable / loadable)
- `queries/*.gq` — Read and mutation queries
- `nanograph.toml` — Project config with query aliases
- `.env.nano.example` — Template for API keys (copy to `.env.nano`, not committed)

## Quick Start

All commands run from this directory.

```bash
# 1. Copy the env template and add an API key
cp .env.nano.example .env.nano
# edit .env.nano — put OPENAI_API_KEY or GEMINI_API_KEY

# 2. Initialize the database
nanograph init

# 3. Load the seed
nanograph load --data seed.jsonl --mode overwrite

# 4. Query via aliases
nanograph run patterns disruption
nanograph run pattern-signals pat-sovereign-ai
nanograph run signal sig-zylon-onprem-ai
```

nanograph is embedded — no server, no Docker, no bucket. The database lives in `spike-intel.nano/` in this directory.

## Everyday Operations

See the [`nanograph-ops`](../../skills/nanograph-ops/) skill for the operational guide (mutations, schema evolution, CDC, embeddings, common gotchas). For bootstrapping a SPIKE graph in a new domain (biotech, fintech, etc.), use the [`nanograph-industry-intel`](../../skills/nanograph-industry-intel/) skill.

## Bootstrap a Different Domain

The bootstrap skill takes the same schema shape and adapts it for biotech, fintech, crypto, geopolitics, or any other domain. It walks you through domain identification, source selection, schema adaptation, initial web research, and `init` + `load`.

## Provenance

Ported from [`omnigraph-starters/industry-intel`](https://github.com/ModernRelay/omnigraph-starters) (Apr 2026). Content is identical; schema and config syntax have been adapted for nanograph 1.2+.
