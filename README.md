# nanograph-skills

Official agent skills for [nanograph](https://nanogra.ph) — an embedded local-first typed property graph database.

These skills teach AI agents (Claude Code, Codex, OpenCode, and others) how to safely operate, query, and migrate nanograph databases.

## Installation

```bash
npx skills add nanograph/nanograph-skills
```

## Available Skills

| Skill | Description |
|-------|-------------|
| **nanograph-ops** | Operational guide for managing nanograph databases — schema design, mutations, querying, search, and maintenance |
| **nanograph-lance** | Safely migrate a nanograph database from Lance storage format v2 to v2.2 |

## Skill Structure

Each skill follows the [agent skills specification](https://agentskills.io/specification):

```
skills/<skill-name>/
├── SKILL.md              # Skill definition with YAML frontmatter
└── references/           # Optional detailed reference docs
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT
