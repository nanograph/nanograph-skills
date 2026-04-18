# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repo contains agent skills that teach AI agents how to operate [nanograph](https://github.com/nanograph/nanograph) databases. Skills follow the [agent skills specification](https://agentskills.io/specification) and are installed via `npx skills add nanograph/nanograph-skills`.

## Structure

- `skills/<name>/SKILL.md` — skill definition with YAML frontmatter
- `skills/<name>/references/` — overflow reference docs for a skill
- `templates/<name>/` — ready-to-run graph starters (schema, seed, queries, config)

There are three skills:
- `nanograph-ops` — operational guide for day-to-day database work
- `nanograph-lance` — one-shot migration of legacy databases onto `NamespaceLineage` storage
- `nanograph-industry-intel` — bootstrap a new SPIKE intelligence graph from the `spike-intel` template (demo or custom domain)

And one template:
- `spike-intel` — AI/ML industry intelligence, ported from the `omnigraph-starters/industry-intel` starter

## No Build/Test Pipeline

This is a content-only repo (markdown skill files). There is no build step, test suite, or linter. Validation is manual: read the skill, check frontmatter, and verify accuracy against nanograph behavior.

`pyproject.toml` and `poetry.lock` are vestigial scaffolding — ignore them. There is no Python code.

## Skill Authoring Conventions

- Skill names use kebab-case and **must** match their directory name
- Every `SKILL.md` requires `name` and `description` in YAML frontmatter
- `name` must match the directory name exactly
- `description` should match what users naturally say — it drives agent auto-discovery
- Every `SKILL.md` opens with a `## Summary` section (short bulleted overview) — this is rendered by external skill listings like `skills.sh`, so preserve it when editing
- Keep `SKILL.md` under 500 lines; move detailed docs to `references/`
- Reference files are plain markdown (no frontmatter required)

## SKILL.md Frontmatter Format

```yaml
---
name: skill-name
description: "When to trigger this skill — written as natural-language matching criteria."
---
```

The `description` field is critical: agents use it to decide whether to load the skill. Write it as a list of trigger conditions (file patterns, user intent phrases, CLI commands) rather than a summary of content.
