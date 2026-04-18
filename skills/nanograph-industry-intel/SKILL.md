---
name: nanograph-industry-intel
description: 'Stand up a SPIKE industry intelligence graph on nanograph. Use this skill whenever the user wants to set up a SPIKE-style knowledge graph for one of six supported domains: AI, Biotech, Fintech, Space, Crypto, or Geopolitics. The skill focuses on ontology and schema: it picks a domain, uses a small sample of source types the user already reads to tune enums, adapts the schema, and initializes the database. Data loading, research, and ingestion are explicitly out of scope — use the nanograph-ops skill for those. Apply when the user says any of: set up SPIKE, bootstrap an intel graph, create a knowledge graph for X industry, initialize intel for Y, start a new context graph, set up a graph for AI / biotech / fintech / space / crypto / geopolitics.'
license: MIT
metadata:
  version: "0.2.0"
  upstream: Ported from omnigraph-starters/skills/omnigraph-intel-bootstrap
---

# nanograph industry intel — SPIKE schema bootstrap

Stand up a SPIKE knowledge-graph schema on nanograph — pick a supported domain, adapt the enum set, initialize an empty database. This skill is **ontology-focused**: data ingestion, research, and seed generation are handled by the `nanograph-ops` skill, not here.

- **Demo path** — use the AI template as-is (schema + real 2026 seed data). Ready to query in under a minute.
- **Custom path** — adapt the schema for one of five other domains: Biotech, Fintech, Space, Crypto, Geopolitics.
- Ask the user for 2–3 example source types they already read — just enough to shape `SourceEntity.type` and `artifactType` enums. Not a full source list, not an ingestion plan.
- Two reference files: [`references/domain-examples.md`](references/domain-examples.md) for ready-made enums per domain; [`references/schema-adaptation.md`](references/schema-adaptation.md) for the keep-vs-change rules.

## Prerequisites

```bash
brew install nanograph          # or see https://nanograph.io
nanograph version               # needs 1.2.0+
```

Semantic search needs an API key in `.env.nano` (`OPENAI_API_KEY` or `GEMINI_API_KEY` — Gemini also covers multimodal).

## Step 1: Ask which path

> Do you want to:
> - **Demo** — use the AI industry template as-is (109 nodes, 154 real signals from early 2026)
> - **Custom** — adapt the schema for a new domain (Biotech, Fintech, Space, Crypto, or Geopolitics)

## Path A — Demo

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

Supported domains: **Biotech, Fintech, Space, Crypto, Geopolitics**. If the user asks for something else, suggest the closest match and note the mapping.

### Step 1: Confirm domain + slug

Pick the domain and a project slug:

| Domain | Slug |
|--------|------|
| Biotech | `bio-intel` |
| Fintech | `fin-intel` |
| Space | `space-intel` |
| Crypto | `crypto-intel` |
| Geopolitics | `geo-intel` |

### Step 2: Ask for source-type examples (to shape enums)

Ask the user for **2–3 example source types** they already read. The goal is to choose the right values for `SourceEntity.type` and `InformationArtifact.artifactType` — not to build a source list or plan ingestion.

Good prompt:

> To tune the schema, give me a few example source types you'd pull from. For instance:
> - *Biotech*: FDA filings? Endpoints News? clinicaltrials.gov?
> - *Fintech*: SEC filings? Matt Levine? earnings calls?
> - *Space*: SpaceNews? FCC filings? NASA press releases?
> - *Crypto*: Bankless? DefiLlama on-chain data? governance forum posts?
> - *Geopolitics*: Foreign Affairs? CFR? State Department briefings?

Each answer tells you which enum values to keep or add. Examples:

- *"FDA filings and trial-registration"* → `artifactType` must include `fda-filing` and `trial-registration`; `SourceEntity.type` needs `regulatory-filing`
- *"FCC filings and launch reports"* → `artifactType` includes `filing` and `launch-report`; `Company.type` needs `agency`
- *"Governance forum posts and on-chain data"* → `SourceEntity.type` adds `governance-forum` and `on-chain-data`; `artifactType` adds `proposal` and `governance-vote`

Do not elicit a full source list or plan the ingestion pipeline — that's out of scope for this skill.

### Step 3: Copy and adapt the template

```bash
cp -r <repo>/templates/spike-intel <slug>
rm <slug>/seed.jsonl <slug>/seed.md      # user brings their own data via nanograph-ops
cd <slug>
```

Pull the enum block for the target domain from [`references/domain-examples.md`](references/domain-examples.md) and replace six enum families in `schema.pg`:

1. `Element.kind`
2. `Signal.domain` + `Element.domain` (must match exactly)
3. `Company.type`
4. `SourceEntity.type` — tweak with the user's source examples
5. `InformationArtifact.artifactType` — tweak with the user's source examples
6. Kind-specific `Element` properties (swap AI-specific like `repository` for domain-specific like `phase` / `orbit` / `chain`)

Keep `Pattern.kind` (`challenge, disruption, dynamic`) — domain-agnostic.

Update `nanograph.toml`:

```toml
[project]
name = "<Domain> Intel — SPIKE"

[db]
default_path = "<slug>.nano"
```

See [`references/schema-adaptation.md`](references/schema-adaptation.md) for the full keep-vs-change rules and nanograph syntax gotchas (`@embed`, no edge cardinality, `@key` on every node, etc.).

### Step 4: Lint + init

```bash
nanograph lint --schema schema.pg --query queries/signals.gq
cp .env.nano.example .env.nano           # add an API key
nanograph init
```

The database is now empty with the adapted schema. The user's next step — **loading data** — happens via the `nanograph-ops` skill (mutation queries for individual records, `nanograph load --mode merge` for batches). This skill stops here.

### Step 5: Hand off

Tell the user:

- Schema adapted for `<domain>` at `<slug>/schema.pg`
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
| [`references/domain-examples.md`](references/domain-examples.md) | Picking the enum block for a target domain |
| [`references/schema-adaptation.md`](references/schema-adaptation.md) | Keep-vs-change rules, nanograph syntax gotchas, validation checklist |

For everything beyond schema setup — mutations, data loading, CDC, embeddings, maintenance — use the `nanograph-ops` skill.
