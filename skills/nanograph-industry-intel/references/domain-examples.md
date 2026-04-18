# AI as the Worked Ontology Example

The only pre-built ontology lives at `templates/spike-intel/schema.pg` (AI industry intel). It is the canonical worked example — a concrete realization of every SPIKE design choice.

For any other domain, **do not copy enum values from here**. Use this file to understand *why* each choice was made, then design the target domain's ontology together with the user (see [Adapting to another domain](#adapting-to-another-domain)).

## Contents

- [AI enum set](#ai-enum-set)
- [Why these choices](#why-these-choices)
- [Adapting to another domain](#adapting-to-another-domain)

## AI enum set

```
Element.kind:      enum(product, technology, framework, concept, ops)
Signal.domain:     enum(training, inference, infra, harness, robotics, security, data-eng, context)?
Element.domain:    enum(training, inference, infra, harness, robotics, security, data-eng, context)?
Company.type:      enum(bigtech, developer, investor, research, hardware, media)?
SourceEntity.type: enum(blog, newsletter, video_channel, academic_repository, podcast, organization)?
InformationArtifact.artifactType: enum(email, youtube, pdf, article)
```

Kind-specific Element properties in the AI schema:

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

`Pattern.kind` — `enum(challenge, disruption, dynamic)` — stays the same across all domains.

## Why these choices

- **`Element.kind` (5 values)** — covers the "things" AI signals point at: shipped products, underlying technologies, reusable frameworks, concepts/ideas, and operational practices. Five is enough to disambiguate queries (`elements product` vs `elements concept`) without overfitting.
- **`domain` (8 values)** — splits the AI landscape along the slice people actually reason about: training vs inference is the classic split, infra/harness distinguish "the rails" from "the agent loop", robotics/security/data-eng/context are recurring cross-cutting areas. Same enum on Signal and Element because they're describing the same landscape from two angles.
- **`Company.type` (6 values)** — the ecosystem roles that matter for AI: giant platforms (bigtech), product/API companies (developer), capital (investor), labs (research), chip/silicon (hardware), and outlets (media). A company can only be one primary type here — resist multi-type.
- **`SourceEntity.type` (6 values)** — how AI news actually ships: blogs, newsletters, YouTube, arXiv-style repos, podcasts, and institutional orgs. Enough granularity to filter `signal-sources`, not so much that categorizing becomes debatable.
- **`artifactType` (4 values)** — covers >95% of what you'll link to: email, youtube, pdf, article. Keep it short; add only if the user routinely consumes a format not in this list.
- **Kind-specific Element properties** — optional fields that only apply to a subset of kinds. Frameworks get `repository`, concepts get `definition` + `key_points`. Agents should populate only what's relevant; unused fields stay null.

The pattern: **narrow enums beat broad ones**, and every enum family exists for a *querying* reason — it has to make `run elements <kind>` or `run signals <domain>` produce a sharp slice.

## Adapting to another domain

When the user wants SPIKE for something other than AI (biotech, fintech, space, crypto, geopolitics, climate, anything else), do not hand them a pre-baked enum set — collaborate:

1. **Explain the AI set above as the pattern.** Show the six enum families and say: we need the same six for your domain, tuned to what you care about.
2. **Elicit `Element.kind`.** Ask the user: *"What kinds of things do your signals talk about? Aim for 4–7 types."* Examples to prime them, not prescribe:
   - Biotech might distinguish therapeutics from platforms from trials.
   - Space might distinguish launchers from spacecraft from missions.
   - Crypto might distinguish protocols from tokens from dapps.
   - Let the user's language drive the enum values, not these examples.
3. **Elicit `domain`.** Ask: *"What sub-fields or verticals matter in your space?"* 5–8 values. Declare the same enum identically on `Signal` and `Element`.
4. **Elicit `Company.type`.** Ask: *"What roles do organizations play in your ecosystem?"* Usually 4–7 values. Remind the user each company picks one primary role.
5. **Derive `SourceEntity.type` and `artifactType` from source examples.** The SKILL.md flow already asks the user for 2–3 example source types they read — those answers give you these two enums. If they mention FDA filings → `regulatory-filing` + `fda-filing`. If they mention FCC filings and launch reports → `regulatory-filing` + `launch-report`. If they mention governance forum posts → `governance-forum` + `proposal`.
6. **Adapt kind-specific properties.** Ask: *"For each kind you named, what 2–4 properties matter most?"* Biotech `therapeutic` probably wants `phase`, `indication`, `moa`, `trial_id`. Space `mission` wants `orbit`, `launch_date`, `mission_agency`. Don't invent — let the user's answers drive it.
7. **Present the proposed enum block back to the user before editing `schema.pg`.** Cheap to revise here; expensive once queries and data start flowing.

Keep `Pattern.kind` (`challenge, disruption, dynamic`) unchanged — it's intentionally domain-agnostic and works everywhere.
