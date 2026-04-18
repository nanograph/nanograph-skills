# Schema Adaptation

What to **keep** vs **change** when adapting the `spike-intel` schema for a new domain.

## What Stays (Domain-Agnostic)

These are SPIKE invariants. Changing them means you're not doing SPIKE anymore.

### Node types

Keep all 10 node types unchanged:

- **SPIKE nodes**: Signal, Pattern, Insight, KnowHow, Element
- **Supportive**: Company, SourceEntity, Expert, InformationArtifact, Chunk

If a domain seems to want a new node type (e.g., DAO for crypto, ClinicalTrial for biotech), prefer modeling it as `Element.kind = dao` or adding a `trial_id` property to Element. Resist adding node types — the SPIKE core works better when it stays narrow.

### Core analytical edges

- `FormsPattern`, `ContradictsPattern` (Signal → Pattern)
- `DrivesPattern`, `ReliesOnPattern`, `ContradictsToPattern` (Pattern → Pattern)
- `HighlightsPattern` (Insight → Pattern)
- `OnElement` (Signal → Element)
- `EnablesElement`, `UsesElement` (Element → Element)
- `ExemplifiesPattern`, `EnablesPattern` (Element → Pattern)
- `ReferencesElement` (KnowHow → Element)

These are the analytical skeleton. Don't touch them.

### Classification / provenance edges

- `DevelopedByCompany` (Element → Company)
- `RelevantCompany` (Signal → Company)
- `AffiliatedWithCompany` (Expert → Company)
- `SpottedInArtifact` (Signal → InformationArtifact)
- `IdentifiedInArtifact` (Element → InformationArtifact)
- `SourcedFromArtifact` (KnowHow → InformationArtifact)
- `SourcedFromSource` (Signal → SourceEntity)
- `PublishedBySource` (InformationArtifact → SourceEntity) — one publisher per artifact (enforced in loader code, not schema)
- `ContributedByExpert` (InformationArtifact → Expert)
- `PartOfArtifact` (Chunk → InformationArtifact) — each Chunk belongs to exactly one artifact (enforced in loader code, not schema)

Keep.

### Fixed properties

- `slug: String @key` on every node (including Chunk — nanograph requires it)
- `stagingTimestamp`, `createdAt`, `updatedAt` timestamps where they are
- `embedding: Vector(3072) @embed(text) @index` on Chunk
- Slug prefixes: `sig-`, `pat-`, `el-`, `ins-`, `how-to-`, `co-`, `exp-`, `ia-`, `source-`

## What Changes (Domain-Specific)

### 1. `Element.kind` enum

The AI schema:
```
kind: enum(product, technology, framework, concept, ops)
```

Replace with domain-appropriate kinds. See [`domain-examples.md`](domain-examples.md) for worked examples:
- Biotech: `therapeutic, mechanism, trial, platform, device, reagent`
- Crypto: `protocol, token, dapp, standard, l1-chain, l2-chain, dao`
- Fintech: `product, platform, regulation, rail, asset-class`
- Geopolitics: `treaty, sanction, conflict, election, policy, institution`

Rule of thumb: 4–7 kinds. Fewer = coarse and useless for filtering; more = overfit.

### 2. `domain` enum on Signal + Element

The AI schema:
```
domain: enum(training, inference, infra, harness, robotics, security, data-eng, context)?
```

Replace with sub-fields of the target domain:
- Biotech: `oncology, neuro, cardio, immuno, rare-disease, metabolic, infectious`
- Crypto: `defi, gaming, infra, memecoins, stablecoins, privacy, identity, dex`
- Fintech: `lending, payments, bnpl, neobank, wealth, infra, compliance`

Must be declared **identically on both Signal and Element** — nanograph inlines enums, so shared enums must be duplicated.

### 3. `Company.type` enum

The AI schema:
```
type: enum(bigtech, developer, investor, research, hardware, media)?
```

Replace with ecosystem roles:
- Biotech: `pharma, biotech, cro, academic, investor, regulator`
- Crypto: `protocol, exchange, investor, dao, l1, l2, media`
- Fintech: `bank, fintech, card-network, neobank, regulator, investor`
- Geopolitics: `government, multilateral, think-tank, media, analyst`

### 4. `SourceType` enum on SourceEntity

The AI schema:
```
type: enum(blog, newsletter, video_channel, academic_repository, podcast, organization)?
```

Usually only needs minor additions. Add anything domain-specific:
- Biotech: `journal, conference, regulatory-filing`
- Crypto: `governance-forum, on-chain-data, exchange-feed`
- Macro: `central-bank, statistical-agency, research-institute`

### 5. `ArtifactType` enum on InformationArtifact

The AI schema:
```
artifactType: enum(email, youtube, pdf, article)
```

Add domain-specific formats:
- Biotech: `paper, preprint, trial-registration, fda-filing`
- Crypto: `proposal, governance-vote, block-explorer, on-chain-tx`
- Macro: `statistical-release, policy-statement, transcript`

### 6. Kind-specific Element properties

The AI schema has these optional properties tied to `kind`:

```
// Product / Framework shared
website: String?
release_year: I32?
license: String?

// Framework-specific
repository: String?
use_cases: [String]?
key_features: [String]?
target_users: String?

// Concept-specific
definition: String?
key_points: [String]?
```

Keep the shared ones (`website`, `release_year`) if they fit. Replace kind-specific with domain-relevant:

**Biotech:**
```
// Therapeutic-specific
phase: enum(preclinical, phase-1, phase-2, phase-3, approved, withdrawn)?
indication: String?
moa: String?           // mechanism of action
sponsor: String?
trial_id: String?

// Platform-specific
modality: String?      // small-molecule, biologic, cell-therapy
```

**Crypto:**
```
// Protocol/token specific
chain: String?          // ethereum, solana, etc.
token_symbol: String?
tvl_usd: F64?
contract_address: String?
governance_model: String?
```

**Fintech:**
```
// Product specific
jurisdiction: String?
license_type: String?
asset_class: String?
```

### 7. `Pattern.kind` enum — usually keep

```
kind: enum(challenge, disruption, dynamic)
```

Intentionally abstract and works across domains. **Only change if the user has strong reasons.** Resist the urge to over-specify.

## nanograph syntax reminders

When editing `schema.pg`, remember:

- `@embed(prop)` — property name unquoted (both `@embed(text)` and `@embed("text")` parse, but unquoted is canonical)
- **No edge cardinality.** nanograph tolerates `@card("...")` but enforces nothing. Don't carry the Omnigraph `@card(0..1)` / `@card(1..1)` annotations over — enforce those constraints in the code that emits edges instead.
- `@unique(src)` inside edge body is unsupported
- Every node type needs `@key` — even `Chunk`. Use a synthetic slug like `{artifact_slug}-c{index}` if needed.
- Comments use `//`, not `#`
- Properties in a node/edge body are newline-separated, not comma-separated

## Update nanograph.toml

After schema changes, update `<slug>/nanograph.toml`:

```toml
[project]
name = "<domain> Intel — SPIKE Framework"
description = "..."

[db]
default_path = "<slug>.nano"
```

If the domain-specific enums changed, aliases referring to enum values (e.g., `patterns disruption`) still work since `Pattern.kind` was kept. But if you added kind-specific aliases (e.g., `elements product`), audit them for the new kinds.

## Validate

After every schema edit:

```bash
cd <slug>
nanograph lint --schema schema.pg --query queries/signals.gq
```

The queries themselves probably don't need changes — they mostly operate on slugs and don't reference enum values. Lint will flag anything that broke.

## Checklist

Before moving to Phase 6:

- [ ] Copied `spike-intel/` to `<slug>/` and removed the embedded skill subfolder
- [ ] Updated `Element.kind` enum
- [ ] Updated `domain` enum (on both Signal and Element)
- [ ] Updated `Company.type` enum
- [ ] Updated `SourceEntity.type` enum if needed
- [ ] Updated `artifactType` enum if needed
- [ ] Replaced kind-specific Element properties
- [ ] Kept `Pattern.kind` as-is (usually)
- [ ] Updated `nanograph.toml` project name + `db.default_path`
- [ ] Deleted the template's `seed.jsonl` and `seed.md` (regenerate in Phase 6)
- [ ] `nanograph lint` passes with 0 errors
