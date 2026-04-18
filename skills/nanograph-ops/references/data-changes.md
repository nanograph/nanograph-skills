# Data changes

How to safely modify data in a nanograph database — mutations, keys, aliases, and the JSONL format used for loads.

## Contents

- [Use mutations, not batch reingestion](#use-mutations-not-batch-reingestion)
- [Mutation constraints](#mutation-constraints)
- [Put `@key` on every node type](#put-key-on-every-node-type)
- [Define aliases for every agent operation](#define-aliases-for-every-agent-operation)
- [JSONL: nodes vs edges](#jsonl-nodes-vs-edges)

## Use mutations, not batch reingestion

The most common agent mistake. Do not export the graph as JSONL, edit rows, and `load --mode overwrite` to change a few records. That destroys CDC continuity, invalidates Lance dataset history, and is far slower than a targeted mutation.

```bash
# Wrong — full reingestion to update one field
nanograph export --format jsonl > dump.jsonl
# ... edit dump.jsonl ...
nanograph load --data dump.jsonl --mode overwrite

# Right — targeted mutation
nanograph run --query queries.gq --name update_status \
  --param slug=cli-acme --param status=healthy
```

Use each operation for its intended purpose:

| Goal | Command |
|------|---------|
| Add one record | `insert` mutation query |
| Update a field on an existing record | `update` mutation query |
| Remove a record | `delete` mutation query |
| Initial bulk data load | `nanograph load --mode overwrite` |
| Add a batch of new records | `nanograph load --mode append` |
| Reconcile external data into keyed records | `nanograph load --mode merge` |

Write reusable parameterized mutations in the `.gq` file when designing the schema, not later:

```graphql
node Task {
    slug: String @key
    title: String
    status: enum(open, in_progress, done)
}

query add_task($slug: String, $title: String, $status: String) {
    insert Task { slug: $slug, title: $title, status: $status }
}

query update_task_status($slug: String, $status: String) {
    update Task set { status: $status } where slug = $slug
}

query remove_task($slug: String) {
    delete Task where slug = $slug
}
```

## Mutation constraints

- All mutations should be parameterized. Do not hardcode slugs, dates, or one-off values into the query body.
- `update` requires `@key` on the target node type.
- `now()` is available in filters, projections, and mutation assignments and resolves once per execution:
  ```graphql
  query touch_client($slug: String) {
      update Client set { updatedAt: now() } where slug = $slug
  }
  ```
- Array params cannot be passed through `--param`.
- Edge inserts use `from:` and `to:` with endpoint `@key` values, not internal IDs:
  ```graphql
  query link_signal($signal: String, $client: String) {
      insert SignalAffects { from: $signal, to: $client }
  }
  ```
- Edge names in queries use camelCase even though the schema defines them in PascalCase. `HasMentor` becomes `hasMentor`.

## Put `@key` on every node type

Every node type that participates in edges or will ever need updates should have a `@key`. In practice, every node type should have one.

```graphql
node Client {
    slug: String @key
    name: String @unique
    status: enum(active, churned, prospect)
}
```

Without `@key`:

- `update` mutations cannot target a stable identity
- Edge JSONL cannot resolve `from` / `to`
- `load --mode merge` has nothing to match on

Use stable human-readable slugs where possible. They make debugging, edge wiring, aliases, and CLI workflows much easier.

## Define aliases for every agent operation

Aliases turn fragile multi-flag commands into short positional calls:

```toml
[query_aliases.search]
query = "queries.gq"
name = "semantic_search"
args = ["q"]
format = "json"

[query_aliases.client]
query = "queries.gq"
name = "client_lookup"
args = ["slug"]
format = "json"

[query_aliases.add-task]
query = "queries.gq"
name = "add_task"
args = ["slug", "title", "status"]
format = "jsonl"

[query_aliases.status]
query = "queries.gq"
name = "update_task_status"
args = ["slug", "status"]
format = "jsonl"
```

```bash
nanograph run search "renewal risk"
nanograph run client cli-acme
nanograph run add-task task-42 "Draft proposal" open
nanograph run status task-42 done
```

Alias design rules:

- Always set `format`. Use `json` for reads, `jsonl` for mutations.
- Use short verb-like names.
- Put the natural scoping argument first.
- List the available aliases in `CLAUDE.md` or `AGENTS.md`.
- Do not create aliases for built-in commands like `describe` or `doctor`.

## JSONL: nodes vs edges

Nodes use `"type"` and `"data"`:

```json
{"type": "Client", "data": {"slug": "cli-acme", "name": "Acme Corp", "status": "active"}}
```

Edges use `"edge"`, `"from"`, and `"to"`:

```json
{"edge": "SignalAffects", "from": "sig-renewal", "to": "cli-acme"}
```

Common mistakes:

- Using `type` for edge rows — this is parsed as a node record.
- Using `src` / `dst` instead of `from` / `to`.
- Using internal IDs instead of endpoint `@key` values.
- Type and edge names in JSONL use schema PascalCase exactly (unlike queries, which use camelCase for edges).
