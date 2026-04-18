# nanograph-skills

Official agent skills for [nanograph](https://github.com/nanograph/nanograph) — an embedded local-first typed property graph database.

These skills teach AI agents (Claude Code, Codex, OpenCode, and others) how to safely operate, query, and migrate nanograph databases.

## Installation

```bash
npx skills add nanograph/nanograph-skills
```

## Available Skills

| Skill | Description |
|-------|-------------|
| **nanograph-ops** | Operational guide for managing nanograph databases — schema design, mutations, querying, CDC, embeddings, media, and maintenance |
| **nanograph-lance** | Safely migrate a legacy nanograph database onto the current `NamespaceLineage` storage generation |
| **nanograph-intel-bootstrap** | Bootstrap a new SPIKE intelligence graph (demo AI domain or custom biotech / fintech / crypto / etc.). Lives alongside the template it deploys. |

## Starter Templates

Ready-to-run graph templates under [`templates/`](./templates).

| Template | Description |
|----------|-------------|
| [`spike-intel/`](./templates/spike-intel) | AI/ML industry intelligence on the SPIKE framework — schema, seed, queries, aliases. 109 nodes, 154 edges of real 2026 signals. Ships with the `nanograph-intel-bootstrap` skill. |

## Skill Structure

Each skill follows the [agent skills specification](https://agentskills.io/specification):

```
skills/<skill-name>/
├── SKILL.md              # Skill definition with YAML frontmatter
└── references/           # Optional detailed reference docs
```

Some skills are co-located with the template they bootstrap (e.g. `templates/spike-intel/nanograph-intel-bootstrap/`). The agent-skills loader picks up `SKILL.md` files from anywhere in the repo.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT
