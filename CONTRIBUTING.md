# Contributing to nanograph-skills

Thanks for your interest in contributing skills to nanograph!

## Getting Started

1. Fork this repo and clone it locally
2. Create a new skill directory under `plugins/nanograph/skills/`
3. Use the [skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator) skill to generate a well-structured skill from scratch

## Skill Guidelines

- Every skill needs a `SKILL.md` with `name` and `description` in YAML frontmatter
- The `name` field must match the directory name (kebab-case)
- Write descriptions that match what users naturally say — this is how agents discover your skill
- Keep `SKILL.md` under 500 lines; move detailed docs to a `references/` subdirectory
- Test your skill locally with real agent workflows before submitting

## Submitting

1. Test the skill in your own projects first — make sure it actually helps agents do the right thing
2. Open a pull request with a short description of what the skill teaches and when it should trigger
3. Include an example of the kind of user request that should activate the skill

## Questions?

Open an issue at https://github.com/nanograph/nanograph-skills/issues — happy to help.
