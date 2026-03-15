# nanograph-skills

Official agent skills for [nanograph](https://nanogra.ph) — an embedded local-first typed property graph database.

These skills teach AI agents (Claude Code, Codex, OpenCode, and others) how to safely operate, query, and migrate nanograph databases.

## Installation

### Claude Code

```bash
claude install-skill https://github.com/nanograph/nanograph-skills
```

### Manual

Clone this repo and point your agent's skill configuration at the `plugins/nanograph/skills/` directory.

## Available Skills

| Skill | Description |
|-------|-------------|
| **nanograph-ops** | Operational guide for managing nanograph databases — schema design, mutations, querying, search, and maintenance |
| **nanograph-lance** | Safely migrate a nanograph database from Lance storage format v2 to v2.2 |

## Skill Structure

Each skill follows the [agent skills specification](https://agentskills.io/specification):

```
plugins/nanograph/skills/<skill-name>/
├── SKILL.md              # Skill definition with YAML frontmatter
└── references/           # Optional detailed reference docs
```

## Contributing

Test your skills locally first. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT
