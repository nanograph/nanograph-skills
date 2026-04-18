# Blobs and multimodal embeddings

nanograph models media as normal nodes with URI properties. For imported managed assets, new graphs store the bytes in hidden Lance-backed storage inside `__blob_store`. For remote references, nanograph stores the logical URI directly.

This gives you:

- normal graph modeling with edges to media nodes
- managed storage for imported `@file:` and `@base64:` assets
- multimodal embeddings through Gemini
- stable logical URIs for exported and queried rows

## Contents

- [Storage model](#storage-model)
- [`@media_uri` annotation](#media_uri-annotation)
- [Media input formats](#media-input-formats)
- [Managed storage vs external URIs](#managed-storage-vs-external-uris)
- [Multimodal embeddings](#multimodal-embeddings)
- [Provider setup](#provider-setup)
- [Current Gemini limits](#current-gemini-limits)
- [URI handling for media embeddings](#uri-handling-for-media-embeddings)
- [Loading media](#loading-media)
- [Querying media](#querying-media)
- [Migrating from OpenAI to Gemini multimodal embeddings](#migrating-from-openai-to-gemini-multimodal-embeddings)
- [Operational notes](#operational-notes)

## Storage model

Use media nodes, not raw blob properties on domain nodes:

```graphql
node Product {
    slug: String @key
    name: String
}

node PhotoAsset {
    slug: String @key
    uri: String @media_uri(mime)
    mime: String
    width: I32?
    height: I32?
    embedding: Vector(768)? @embed(uri) @index
}

edge HasPhoto: Product -> PhotoAsset
```

- `uri` is the logical media reference
- `mime` stores the media type
- `embedding` stores the multimodal vector
- edges connect media to the rest of the graph

nanograph does not write media payload bytes into `nodes/*` or `edges/*` tables. Managed imported bytes live in `__blob_store`.

## `@media_uri` annotation

`@media_uri(mime_prop)` marks a `String` property as a media URI and names the sibling MIME property.

Rules:

- valid only on `String` or `String?`
- the argument must name a sibling `String` or `String?` property
- plain strings are rejected during load; use a supported media input format
- list properties cannot use `@media_uri`

## Media input formats

| Format | Behavior |
|--------|----------|
| `@file:path/to/image.jpg` | reads local bytes, imports them into managed blob storage, stores `lanceblob://sha256/...` |
| `@base64:...` | decodes bytes, imports them into managed blob storage, stores `lanceblob://sha256/...` |
| `@uri:file:///absolute/path/image.jpg` | stores the URI directly, no copy |
| `@uri:https://example.com/image.jpg` | stores the URI directly, no copy |
| `@uri:s3://bucket/path/image.jpg` | stores the URI directly, no copy |

Relative `@file:` paths are resolved relative to the source JSONL file.

## Managed storage vs external URIs

For new `NamespaceLineage` graphs:

- imported managed assets from `@file:` and `@base64:` become `lanceblob://sha256/...`
- remote references such as `https://`, `s3://`, and `abfss://` remain as logical URIs
- local `file://` URIs remain external references unless you explicitly import them

`NANOGRAPH_MEDIA_ROOT` is a legacy input only used during older storage migration flows. It is not the active steady-state storage model for new graphs.

## Multimodal embeddings

`@embed(source_prop)` works on media URI properties. When the source has `@media_uri(...)`, `@embed(uri)` becomes multimodal embedding for images, audio, video, and PDFs instead of text embedding.

When the embedding is null or missing:

- `nanograph load` generates the vector automatically
- `nanograph embed` can backfill it later

## Provider setup

Gemini is the practical multimodal provider today. OpenAI embeddings remain text-only in nanograph.

```toml
[embedding]
provider = "gemini"
model = "gemini-embedding-2-preview"
```

```bash
GEMINI_API_KEY=...   # https://aistudio.google.com/
```

Backfill:

```bash
nanograph embed --db app.nano --only-null
```

## Current Gemini limits

nanograph enforces Gemini-side limits locally:

| Media | Constraints |
|-------|-------------|
| Text | conservative local estimate capped at 8192 input tokens |
| Images | PNG or JPEG only, batched up to 6 per request |
| Audio | any `audio/*` type |
| Video | MP4 or MOV only, up to 120 seconds |
| PDF | up to 6 pages |

If validation fails, `load` or `embed` fails before the provider call is sent.

## URI handling for media embeddings

- `lanceblob://...` managed assets are read back from `__blob_store`
- local `file://` assets are read and sent inline
- `http://` and `https://` media URIs are fetched, validated, and embedded inline
- non-HTTP remote image and audio URIs may be passed through when the provider supports them
- non-HTTP remote PDF and video URIs are rejected because nanograph cannot validate page count or duration without reading the bytes

## Loading media

Example JSONL:

```jsonl
{"type":"PhotoAsset","data":{"slug":"space","uri":"@file:photos/space.jpg"}}
{"type":"PhotoAsset","data":{"slug":"beach","uri":"@uri:https://example.com/beach.jpg","mime":"image/jpeg"}}
{"type":"Product","data":{"slug":"rocket","name":"Rocket Poster"}}
{"edge":"HasPhoto","from":"rocket","to":"space"}
```

```bash
nanograph load --db app.nano --data data.jsonl --mode overwrite
```

After load:

- imported managed assets are represented as `lanceblob://sha256/...`
- external `@uri:` assets keep their logical URI values
- `mime` is filled automatically when possible
- `embedding` is generated if configured and currently null

## Querying media

Media is modeled as nodes, so query it like any other type.

Text-to-image search:

```graphql
query image_search($q: String) {
    match { $img: PhotoAsset }
    return {
        $img.slug as slug,
        $img.uri as uri,
        $img.mime as mime,
        nearest($img.embedding, $q) as score
    }
    order { nearest($img.embedding, $q) }
    limit 5
}
```

Traverse from matched images to related entities:

```graphql
query products_from_image_search($q: String) {
    match {
        $product: Product
        $product hasPhoto $img
    }
    return {
        $product.slug as product,
        $product.name as name,
        $img.slug as image,
        $img.uri as uri
    }
    order { nearest($img.embedding, $q) }
    limit 5
}
```

The intended multimodal workflow:

1. Store media as nodes with URI properties.
2. Embed those media nodes.
3. Issue a text query.
4. Rank media nodes with `nearest(...)`.
5. Traverse from media nodes into the rest of the graph.

## Migrating from OpenAI to Gemini multimodal embeddings

OpenAI embeddings in nanograph are text-only. Gemini supports text and media in one embedding space.

### 1. Update config

```toml
[embedding]
provider = "gemini"
model = "gemini-embedding-2-preview"
```

Swap the key in `.env.nano`:

```bash
GEMINI_API_KEY=...
```

### 2. Resize vector dimensions if needed

| Model | Native dim | Typical schema dim |
|-------|------------|--------------------|
| `text-embedding-3-small` | 1536 | `Vector(1536)` |
| `gemini-embedding-2-preview` | 3072 | `Vector(768)` or `Vector(3072)` |

If changing dimensions, edit the schema and migrate:

```bash
nanograph migrate --dry-run
nanograph migrate
```

### 3. Re-embed everything

Old vectors are not comparable to the new model:

```bash
nanograph embed --reindex
```

This is the one time a full re-embed is necessary. After that, `--only-null` works again for incremental backfills.

### 4. Verify

```bash
nanograph doctor
nanograph run <your-search-alias> "test query"
```

After migration:

| Source property | Behavior |
|----------------|----------|
| Plain `String` | text embedding via Gemini |
| `String @media_uri(mime)` | multimodal embedding via Gemini |

Text and media nodes share the same embedding space, so `nearest(...)` can rank them together.

## Operational notes

- nanograph stores managed imported blob bytes in `__blob_store`, not in node or edge tables
- imported managed assets do not require a repo-level media directory
- external URIs remain external references
- `nanograph export` preserves logical URIs; it does not copy remote assets
