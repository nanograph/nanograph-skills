# nanograph-skills

This repo contains agent skills that teach AI agents how to operate nanograph databases.

## Structure

- `skills/<name>/SKILL.md` — individual skills with YAML frontmatter
- `skills/<name>/references/` — overflow documentation for skills

## Conventions

- Skill names use kebab-case and must match their directory name
- Every `SKILL.md` requires `name` and `description` in YAML frontmatter
- Descriptions should match what users naturally say — they drive auto-discovery
- Keep `SKILL.md` under 500 lines; move detailed docs to `references/`
- Reference files are plain markdown (no frontmatter required)
