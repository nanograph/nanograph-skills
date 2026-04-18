# Domain Examples

Ready-made enum sets for the six supported domains. Use as starting points — adapt to the user's specific scope. This is an **ontology reference**, not a research guide: the goal is to help the user pick schema values, not to build their source pipeline.

The canonical AI schema ships at `templates/spike-intel/schema.pg` and is the default starting point for any domain (copy it, then swap the domain-specific enums below).

## Contents

- [AI](#ai)
- [Biotech](#biotech)
- [Fintech](#fintech)
- [Space](#space)
- [Crypto / Web3](#crypto--web3)
- [Geopolitics](#geopolitics)

---

## AI

Ships as the template. No adaptation needed — just `cp -r templates/spike-intel <destination>` and you have the AI enum set below.

```
Element.kind:      enum(product, technology, framework, concept, ops)
Signal.domain:     enum(training, inference, infra, harness, robotics, security, data-eng, context)?
Element.domain:    enum(training, inference, infra, harness, robotics, security, data-eng, context)?
Company.type:      enum(bigtech, developer, investor, research, hardware, media)?
SourceEntity.type: enum(blog, newsletter, video_channel, academic_repository, podcast, organization)?
InformationArtifact.artifactType: enum(email, youtube, pdf, article)
```

Example source shape (if asking the user): *"Do you read Stratechery, The Information, arXiv papers, or AI-specific podcasts?"* — answers influence `SourceEntity.type` and `artifactType`.

---

## Biotech

```
Element.kind:      enum(therapeutic, mechanism, trial, platform, device, reagent, diagnostic)
Signal.domain:     enum(oncology, neuro, cardio, immuno, rare-disease, metabolic, infectious, gene-therapy)?
Element.domain:    enum(oncology, neuro, cardio, immuno, rare-disease, metabolic, infectious, gene-therapy)?
Company.type:      enum(pharma, biotech, cro, academic, investor, regulator)?
SourceEntity.type: enum(journal, newsletter, conference, regulatory-filing, blog, podcast)?
InformationArtifact.artifactType: enum(paper, preprint, trial-registration, fda-filing, article, pdf)
```

Kind-specific Element properties (replace the AI-specific ones):

```
// Therapeutic / trial
phase: enum(preclinical, phase-1, phase-2, phase-3, approved, withdrawn)?
indication: String?
moa: String?                // mechanism of action
modality: String?           // small-molecule, biologic, cell-therapy, gene-therapy
sponsor: String?
trial_id: String?           // NCT identifier
fda_status: String?
```

Example source shape: *"FDA filings, Endpoints News, clinicaltrials.gov?"* → `artifactType` gains `fda-filing` and `trial-registration`; `SourceEntity.type` gains `regulatory-filing`.

Likely pattern themes: FDA approval cascades, platform-vs-modality competition, trial-failure clusters, M&A under regulatory uncertainty.

---

## Fintech

```
Element.kind:      enum(product, platform, regulation, rail, asset-class, license)
Signal.domain:     enum(lending, payments, bnpl, neobank, wealth, infra, compliance, crypto-banking)?
Element.domain:    enum(lending, payments, bnpl, neobank, wealth, infra, compliance, crypto-banking)?
Company.type:      enum(bank, fintech, card-network, neobank, regulator, investor, processor, aggregator)?
SourceEntity.type: enum(blog, newsletter, publication, regulatory-filing, earnings-call, podcast)?
InformationArtifact.artifactType: enum(article, earnings-report, regulatory-filing, sec-disclosure, press-release, pdf)
```

Kind-specific Element properties:

```
jurisdiction: String?
license_type: String?
asset_class: String?
volume_usd: F64?
customer_count: I64?
launch_year: I32?
```

Example source shape: *"SEC filings, Matt Levine, Fintech Takes?"* → `artifactType` gains `sec-disclosure`; `SourceEntity.type` leans on `newsletter` + `regulatory-filing`.

Likely pattern themes: rail modernization (RTP, FedNow, SEPA Instant), embedded finance, stablecoin-as-payment-rail, BaaS consolidation under regulatory scrutiny.

---

## Space

```
Element.kind:      enum(launcher, spacecraft, payload, mission, constellation, platform, concept)
Signal.domain:     enum(launch, earth-obs, comms, science, human-spaceflight, defense, deep-space, commercial)?
Element.domain:    enum(launch, earth-obs, comms, science, human-spaceflight, defense, deep-space, commercial)?
Company.type:      enum(launch-provider, satellite-operator, integrator, agency, investor, analyst, media)?
SourceEntity.type: enum(newsletter, publication, podcast, agency-release, regulatory-filing, conference)?
InformationArtifact.artifactType: enum(article, press-release, launch-report, filing, mission-report, pdf)
```

Kind-specific Element properties:

```
// Launcher / spacecraft / mission
orbit: String?               // LEO, MEO, GEO, SSO, HEO, cislunar, interplanetary
payload_mass_kg: F64?
launch_date: DateTime?
launch_provider: String?
constellation_size: I32?     // for constellations
vehicle_family: String?      // e.g., Falcon, Starship, Ariane
mission_agency: String?      // NASA, ESA, CNSA, ISRO, JAXA
```

Example source shape: *"SpaceNews, Ars Technica space section, Jonathan McDowell's launch log, FCC filings?"* → `artifactType` gains `filing` (for FCC IBFS entries) and `launch-report`; `Company.type` needs `agency` (NASA, ESA) alongside commercial operators.

Likely pattern themes: launch cadence acceleration, commercial LEO transition (post-ISS), mega-constellation maturation, lunar & Mars commercial contracts, GPS/PNT alternatives.

---

## Crypto / Web3

```
Element.kind:      enum(protocol, token, dapp, standard, l1-chain, l2-chain, dao, bridge, oracle)
Signal.domain:     enum(defi, gaming, infra, memecoins, stablecoins, privacy, identity, dex, lending, staking)?
Element.domain:    enum(defi, gaming, infra, memecoins, stablecoins, privacy, identity, dex, lending, staking)?
Company.type:      enum(protocol, exchange, investor, dao, l1, l2, media, market-maker, infra-provider)?
SourceEntity.type: enum(blog, newsletter, podcast, governance-forum, on-chain-data, exchange-feed, research)?
InformationArtifact.artifactType: enum(article, proposal, governance-vote, on-chain-tx, podcast-episode, research-report, pdf)
```

Kind-specific Element properties:

```
chain: String?              // ethereum, solana, base, etc.
token_symbol: String?
contract_address: String?
tvl_usd: F64?
mcap_usd: F64?
governance_model: String?
audit_status: String?
launch_date: DateTime?
```

Example source shape: *"Bankless, Messari, on-chain data from DefiLlama, protocol governance forums?"* → `SourceEntity.type` needs `governance-forum` and `on-chain-data`; `artifactType` gains `proposal` and `governance-vote`.

Likely pattern themes: L2 fragmentation vs consolidation, stablecoin regulation, DEX/CEX liquidity shifts, governance attacks, bridge exploits.

---

## Geopolitics

```
Element.kind:      enum(treaty, sanction, conflict, election, policy, institution, actor)
Signal.domain:     enum(us-china, europe-russia, middle-east, trade, climate-negotiations, nuclear, cyber, migration)?
Element.domain:    enum(us-china, europe-russia, middle-east, trade, climate-negotiations, nuclear, cyber, migration)?
Company.type:      enum(government, multilateral, think-tank, media, analyst, ngo, alliance)?
SourceEntity.type: enum(publication, newsletter, podcast, research-institute, government-release, academic-journal)?
InformationArtifact.artifactType: enum(article, policy-paper, communique, treaty-text, sanctions-list, speech, pdf)
```

Kind-specific Element properties:

```
jurisdiction: String?           // country or bloc
effective_date: DateTime?
status: enum(proposed, active, expired, withdrawn)?
signatories: [String]?
```

Example source shape: *"Foreign Affairs, CFR, State Department briefings, Lawfare?"* → `Company.type` needs `think-tank` and `government`; `artifactType` gains `policy-paper` and `communique`.

Likely pattern themes: sanctions expansions and workarounds, multi-polar alignment shifts, export-control cascades, climate-finance delivery gaps, conflict-to-negotiation transitions.

---

## Using These as Defaults

When adapting for a new domain, don't build from scratch: pick the closest domain above, copy its enum block into `schema.pg`, then tweak.

If the user is vague about sources, offer the matching domain's "example source shape" prompt and adjust enums based on their answer. The goal is **enough information to shape the schema** — not a full source list or ingestion plan.
