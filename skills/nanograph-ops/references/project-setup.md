# Project setup

How to lay out a nanograph project so agents and humans can work from the same conventions — config, committed artifacts, secrets, and CLI output expectations.

## Contents

- [Run from the project directory](#run-from-the-project-directory)
- [Treat `nanograph.toml` as the operational backbone](#treat-nanographtoml-as-the-operational-backbone)
- [Commit the right artifacts to git](#commit-the-right-artifacts-to-git)
- [Never commit `.env.nano`](#never-commit-envnano)
- [Add nanograph rules to `CLAUDE.md` or `AGENTS.md`](#add-nanograph-rules-to-claudemd-or-agentsmd)
- [Agent output conventions](#agent-output-conventions)

## Run from the project directory

nanograph resolves `nanograph.toml` from the current working directory. It does not walk parent directories.

```bash
# Wrong — "database path is required" or missing-config behavior
cd /some/other/dir && nanograph describe

# Right — run from the project root
cd /path/to/project && nanograph describe
```

If the project uses `[db] default_path` and `[schema] default_path` in `nanograph.toml`, commands can omit `--db` and `--schema` — but only from the right directory.

## Treat `nanograph.toml` as the operational backbone

`nanograph.toml` is the interface contract between the project and the agents operating on it. A good config eliminates fragile one-off CLI invocations:

```toml
[project]
name = "My Graph"
description = "What this graph is for — agents read this."
instruction = "Run from this directory. Use aliases for common operations."

[db]
default_path = "app.nano"

[schema]
default_path = "schema.pg"

[query]
roots = ["."]

[embedding]
provider = "openai"

[cli]
output_format = "table"

[query_aliases.search]
query = "queries.gq"
name = "semantic_search"
args = ["q"]
format = "json"

[query_aliases.add-task]
query = "queries.gq"
name = "add_task"
args = ["slug", "title", "status"]
format = "jsonl"
```

Without this config, every command requires `--db`, `--query`, `--name`, and `--format`.

## Commit the right artifacts to git

| File | Commit? | Why |
|------|---------|-----|
| `nanograph.toml` | Yes | Shared project config, aliases, defaults |
| `schema.pg` | Yes | Schema is code |
| `queries.gq` | Yes | Query and mutation library |
| `seed.jsonl` | Yes | Reproducible bootstrap data |
| `*.nano/` | Yes, if small enough | Embedded database state is often part of the project |
| `.env.nano` | No | Secrets |

The database is not regenerable from seed. Once mutations have been applied, the DB contains records, edges, CDC history, and embeddings that do not exist anywhere else.

Small databases under roughly 50 MB per file are fine to commit — treat them like a SQLite file checked into the repo. Large ones should be backed up outside git (S3 sync, filesystem snapshots, rsync), but the schema, queries, config, and seed files should still be committed.

## Never commit `.env.nano`

`.env.nano` holds secrets like `OPENAI_API_KEY`. Add it to `.gitignore` before the first commit:

```gitignore
.env.nano
```

If it was already committed:

```bash
git rm --cached .env.nano
echo ".env.nano" >> .gitignore
git add .gitignore
git commit -m "stop tracking .env.nano"
```

Then rotate the exposed key immediately.

## Add nanograph rules to `CLAUDE.md` or `AGENTS.md`

AI agents read `CLAUDE.md` and `AGENTS.md` at the repo root. If a project uses nanograph, include operational rules:

```markdown
## nanograph

This project uses a nanograph database at `app.nano`.

- Always run nanograph commands from the directory where `nanograph.toml` lives
- Use `nanograph describe --format json` before querying
- Use mutation queries for data changes — never export/edit/reimport JSONL
- Use `nanograph run <alias>` instead of constructing raw `--query` / `--name` / `--param` calls
- After editing `.gq` or `.pg`, run `nanograph lint --query <file>`
- Never commit `.env.nano`
- Use `--format json` or `--format jsonl` for machine-readable output
```

Tailor to the project: list aliases, slug conventions, and any required fields.

## Agent output conventions

`table` output is for humans. Agents should use machine-readable output.

For `nanograph run`:

```bash
nanograph run search "renewal risk" --format json
nanograph run search "renewal risk" --format jsonl
```

For other commands:

```bash
nanograph describe --json
nanograph migrate --dry-run --json
nanograph doctor --json
```

Set `format` in aliases so agents do not need to remember output flags.

Prefer aliases over raw invocations:

```bash
# Fragile
nanograph run --query queries.gq --name update_task_status \
  --param slug=task-123 --param status=done --format jsonl

# Better
nanograph run status task-123 done
```
