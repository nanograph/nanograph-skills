# nanograph-skills

This repo is a skill marketplace for AI agents. It contains skills that teach agents how to operate nanograph databases.

## Structure

- `plugins/nanograph/` — the nanograph plugin
- `plugins/nanograph/.claude-plugin/plugin.json` — plugin manifest
- `plugins/nanograph/skills/<name>/SKILL.md` — individual skills with YAML frontmatter
- `plugins/nanograph/skills/<name>/references/` — overflow documentation for skills
- `.claude-plugin/marketplace.json` — top-level marketplace catalog
## Conventions

- Skill names use kebab-case and must match their directory name
- Every `SKILL.md` requires `name` and `description` in YAML frontmatter
- Descriptions should match what users naturally say — they drive auto-discovery
- Keep `SKILL.md` under 500 lines; move detailed docs to `references/`
- Reference files are plain markdown (no frontmatter required)
