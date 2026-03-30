# Blobs and Multimodal Embeddings

NanoGraph does not store raw blob bytes inside `.nano/`. Media is modeled as normal nodes with external URIs. The database stores the media URI, mime type, optional derived metadata, and optional embeddings. This keeps `.nano/` small and Git-friendly while enabling text-to-image retrieval and graph traversal from media nodes.

## Storage model

Use media nodes, not blob properties:

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

- `uri` points to the external media asset
- `mime` stores the media type
- `embedding` stores the multimodal vector
- edges connect domain entities to media assets

NanoGraph never writes media bytes into `nodes/*` or `edges/*` Lance datasets.

## `@media_uri` annotation

`@media_uri(mime_prop)` marks a `String` property as an external media URI and names the sibling mime field.

Rules:
- Valid only on `String` or `String?`
- The argument must name a sibling `String` or `String?` property
- Plain strings are rejected during load — values must use one of the media input formats
- List properties cannot have `@media_uri`

### Media input formats

| Format | Behavior |
|--------|----------|
| `@file:path/to/image.jpg` | Reads local file, imports into media root, stores `file://` URI |
| `@base64:...` | Decodes bytes, imports into media root, stores `file://` URI |
| `@uri:file:///absolute/path/image.jpg` | Stores URI directly, no copy |
| `@uri:https://example.com/image.jpg` | Stores URI directly, no copy |
| `@uri:s3://bucket/path/image.jpg` | Stores URI directly, no copy |

When NanoGraph imports bytes from `@file:` or `@base64:`, it writes them outside the database under the media root and stores the final `file://...` URI in the node.

### Media root

Default location: `<db-parent>/media/`

Override with:

```bash
NANOGRAPH_MEDIA_ROOT=/absolute/path/to/media
```

Imported files are content-addressed and grouped by node type:

```
media/
  photoasset/
    4c8f...e2a1.jpg
```

Relative `@file:` paths are resolved relative to the source JSONL file, so they only work when loading from a file on disk.

## Multimodal embeddings

`@embed(source_prop)` works on media URI properties. When the source has `@media_uri(...)`, NanoGraph treats `@embed(uri)` as multimodal embedding (images, audio, video, PDFs), not text embedding.

When the embedding is null or missing:
- `nanograph load` generates the vector automatically
- `nanograph embed` can backfill it later

## Provider setup

Gemini is the practical multimodal provider today. OpenAI embeddings remain text-only in NanoGraph.

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

### Current Gemini limits

NanoGraph enforces Gemini-side limits locally:

| Media | Constraints |
|-------|-------------|
| Images | PNG or JPEG only; batched up to 6 per request |
| Audio | any `audio/*` type |
| Video | MP4 or MOV only, up to 120 seconds |
| PDF | up to 6 pages |
| Text | conservative local estimate capped at 8192 input tokens |

If validation fails, `load` or `embed` fails before the provider call is sent.

### URI handling for media embeddings

- `@file:` and `@base64:` import bytes into the media root, then embed from local `file://` assets
- Local `file://` media files are read and sent inline
- `http://` and `https://` media URIs are fetched, validated, and embedded inline
- Non-HTTP remote image and audio URIs can be passed through as provider file URIs
- Non-HTTP remote PDF and video URIs are rejected (NanoGraph cannot validate page count or duration without reading bytes)

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
- `PhotoAsset.uri` is a durable external URI
- `PhotoAsset.mime` is filled automatically if possible
- `PhotoAsset.embedding` is generated if configured and null

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
1. Store media as nodes with external URIs
2. Embed those media nodes
3. Issue a text query
4. Rank media nodes with `nearest(...)`
5. Traverse from media nodes into the rest of the graph

## Migrating from OpenAI to Gemini multimodal embeddings

OpenAI embeddings in NanoGraph are text-only. Gemini supports text and media (images, audio, video, PDFs) with a single model. If you started with OpenAI and want unified text + media embeddings:

### 1. Update config

```toml
[embedding]
provider = "gemini"
model = "gemini-embedding-2-preview"
```

Swap the key in `.env.nano`:

```bash
GEMINI_API_KEY=...   # https://aistudio.google.com/
```

### 2. Resize vector dimensions if needed

| Model | Native dim | Typical schema dim |
|-------|-----------|-------------------|
| `text-embedding-3-small` (OpenAI) | 1536 | `Vector(1536)` |
| `gemini-embedding-2-preview` (Gemini) | 3072 | `Vector(768)` or `Vector(3072)` |

`Vector(768)` is a practical default for Gemini — good quality at half the storage of OpenAI 1536.

If changing dimensions, edit the schema and migrate:

```bash
nanograph migrate --dry-run
nanograph migrate
```

### 3. Re-embed everything

Old vectors are not comparable to the new model's vectors:

```bash
nanograph embed --reindex
```

This is the one time a full re-embed is necessary. After this, `--only-null` works normally for incremental backfills.

### 4. Verify

```bash
nanograph doctor
nanograph run <your-search-alias> "test query"
```

After migration, `@embed(source_prop)` does the right thing for both content types:

| Source property | Behavior |
|----------------|----------|
| Plain `String` | Text embedding via Gemini |
| `String @media_uri(mime)` pointing to media | Multimodal embedding via Gemini (images, audio, video, PDFs) |

Text and media nodes share the same embedding space, so `nearest(...)` queries rank them together.

## Operational notes

- NanoGraph stores references and derived vectors, not blob payloads, in `.nano/`
- Imported media files live under the media root, not under `nodes/` or `edges/`
- If you keep the media root inside your repo, add it to `.gitignore`
- `nanograph export` preserves URIs by default; it does not copy external assets
