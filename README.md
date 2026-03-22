# neopsyke-pgvector-memory

First-party pgvector-backed long-term memory provider for NeoPsyke.

## Scope

- Default NeoPsyke memory provider for `memory=default`
- Semantic/vector recall backed by PostgreSQL + pgvector
- Typed imprint support for:
  - narrative memory
  - facts
  - relations
  - episodes
- HTTP `v1` provider contract for NeoPsyke
- Optional MCP adapter for external clients and tool interoperability

## Current Transport Posture

- NeoPsyke default transport: HTTP
- Optional adapter: MCP
- Default provider startup transport in this repository: HTTP

The stable HTTP contract is versioned under `/v1/...`. Breaking wire changes
must move to `/v2/...`.

## Stable HTTP v1 endpoints

- `GET /v1/health`
- `GET /v1/metrics`
- `POST /v1/recall`
- `POST /v1/imprint`
- `POST /v1/admin/forget`
- `POST /v1/admin/reset`

## HTTP API Quick Examples

Base URL examples below assume the provider is running locally on:

```text
http://127.0.0.1:7841
```

### Health

```bash
curl -s http://127.0.0.1:7841/v1/health
```

Example response:

```json
{
  "provider": "neopsyke-pgvector-memory",
  "available": true,
  "detail": "http_ready",
  "degraded": false
}
```

### Recall

```bash
curl -s http://127.0.0.1:7841/v1/recall \
  -H 'Content-Type: application/json' \
  -d '{
    "namespace": "neopsyke",
    "cue": "What do I know about the user preferences for learning?",
    "intent": "GENERAL",
    "maxItems": 4,
    "maxChars": 1200
  }'
```

Example response:

```json
{
  "provider": "neopsyke-pgvector-memory",
  "items": [
    {
      "id": "42",
      "kind": "preference",
      "summary": "The user likes structured explanations with concrete examples.",
      "content": "The user likes structured explanations with concrete examples.",
      "score": 0.91,
      "confidence": 0.82,
      "timestamp": "2026-03-23T10:12:00Z",
      "tags": ["kind:narrative", "subject:user"],
      "eventType": null,
      "metadata": {
        "namespace": "neopsyke",
        "source": "ego_memory_assessment"
      }
    }
  ],
  "renderedText": "- The user likes structured explanations with concrete examples.",
  "hitCount": 1,
  "truncated": false
}
```

### Imprint: narrative

```bash
curl -s http://127.0.0.1:7841/v1/imprint \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "narrative",
    "namespace": "neopsyke",
    "summary": "The user prefers concise explanations.",
    "kind": "PREFERENCE",
    "confidence": 0.9,
    "tags": ["kind:preference", "subject:user"],
    "source": "manual_example"
  }'
```

Example response:

```json
{
  "provider": "neopsyke-pgvector-memory",
  "accepted": true,
  "storedCount": 1,
  "detail": "inserted"
}
```

### Imprint types

The `POST /v1/imprint` endpoint supports these `type` values:

- `narrative`
- `fact`
- `relation`
- `episode`

Common shape notes:

- `narrative` uses `summary`
- `fact` uses `subject`, `predicate`, `obj`
- `relation` uses `from`, `relation`, `to`
- `episode` uses `summary` and may include `eventType`, `occurredAt`, `details`, and `metadata`

### Metrics

```bash
curl -s http://127.0.0.1:7841/v1/metrics
```

Example response:

```json
{
  "embedding": {
    "requests": 12,
    "errors": 0
  },
  "cache": {
    "hits": 8,
    "misses": 4
  },
  "database": {
    "searches": 7,
    "inserts": 3,
    "errors": 0
  },
  "tools": {}
}
```

The `/v1/metrics` endpoint is stable in existence and top-level object shape,
but individual metric keys may evolve within `v1`.

### Admin: forget

```bash
curl -s http://127.0.0.1:7841/v1/admin/forget \
  -H 'Content-Type: application/json' \
  -d '{
    "namespace": "neopsyke",
    "tagMarkers": ["kind:lesson"]
  }'
```

Example response:

```json
{
  "deletedCount": 2,
  "detail": ""
}
```

### Admin: reset

```bash
curl -s http://127.0.0.1:7841/v1/admin/reset \
  -H 'Content-Type: application/json' \
  -d '{
    "namespace": "neopsyke",
    "clearAll": true
  }'
```

Example response:

```json
{
  "deletedCount": 14,
  "detail": ""
}
```

### Error shape

Errors use a consistent JSON shape:

```json
{
  "provider": "neopsyke-pgvector-memory",
  "error": "unsupported_type",
  "detail": "Unsupported imprint type 'unknown'."
}
```

Typical status codes:

- `400` invalid or unsupported request
- `503` dependency/runtime failure
- `500` unexpected internal provider error

## Requirements

- JDK 21+
- `MISTRAL_API_KEY` or `EMBEDDING_API_KEY` for the default embedder
- Docker for the default local pgvector boot path

By default the provider now owns the local pgvector boot path:

- it targets a local PostgreSQL JDBC URL by default
- it auto-starts a Docker-managed `pgvector/pgvector` container when needed
- it persists data in the named Docker volume
  `neopsyke-pgvector-memory-data`

Set `PGVECTOR_BOOTSTRAP_MODE=off` if you want to point the provider at an
already managed PostgreSQL/pgvector instance instead.

## Quick start

1. Copy the env template:

```bash
cp .env.example .env
```

2. Set `MISTRAL_API_KEY` in `.env` or your shell.

3. Run the HTTP provider:

```bash
./gradlew run --args="--transport=http --port=7841"
```

The provider will:

- ensure Docker is available
- start or reuse the local pgvector container if the configured DB is not reachable
- initialize the schema
- expose the HTTP `v1` contract on `http://127.0.0.1:7841`

## Local development

Run the HTTP provider:

```bash
./gradlew run --args="--transport=http --port=7841"
```

Run the MCP adapter:

```bash
./gradlew run --args="--transport=mcp"
```

Build the standalone fat jar:

```bash
./gradlew fatJar
```

Build the standalone release bundle:

```bash
./gradlew releaseBundleZip
```

Artifacts are written to:

```text
build/libs/neopsyke-pgvector-memory-0.1.0-all.jar
build/distributions/neopsyke-pgvector-memory-0.1.0-bundle.zip
```

## Environment

- `PGVECTOR_DB_URL`
- `PGVECTOR_DB_USER`
- `PGVECTOR_DB_PASSWORD`
- `MEMORY_DEFAULT_NAMESPACE`
- `EMBEDDING_API_KEY`
- `MISTRAL_API_KEY`
- `EMBEDDING_BASE_URL`
- `EMBEDDING_MODEL`
- `EMBEDDING_DIMENSIONS`
- `MEMORY_SEARCH_DEFAULT_LIMIT`
- `MEMORY_SEMANTIC_DEDUPE_SIMILARITY_THRESHOLD`
- `MEMORY_SEMANTIC_DEDUPE_MIN_CONFIDENCE`
- `MEMORY_FACT_DEFAULT_SUBJECT`
- `PGVECTOR_BOOTSTRAP_MODE` (`auto|off`, default: `auto`)
- `PGVECTOR_BOOTSTRAP_DOCKER_IMAGE`
- `PGVECTOR_BOOTSTRAP_CONTAINER_NAME`
- `PGVECTOR_BOOTSTRAP_VOLUME_NAME`
- `PGVECTOR_BOOTSTRAP_STARTUP_TIMEOUT_MS`
- `MEMORY_PROVIDER_TRANSPORT`
- `MEMORY_PROVIDER_HTTP_HOST`
- `MEMORY_PROVIDER_HTTP_PORT`

## Manual Docker management

If you want to run the local pgvector container yourself instead of letting the
provider auto-bootstrap it:

```bash
docker compose up -d pgvector
```

Stop it:

```bash
docker compose stop pgvector
```

Reset local provider data:

```bash
docker compose down -v
```

## Release shape

The standalone publishable artifacts for this project are:

- fat jar: `neopsyke-pgvector-memory-<version>-all.jar`
- release bundle zip: `neopsyke-pgvector-memory-<version>-bundle.zip`
- checksum manifest: `SHA256SUMS`

Current release channel:

- GitHub Releases only
- version tags matching `v*` trigger the release workflow
- Maven Central publishing is intentionally deferred for now

The release bundle includes:

- the fat jar
- this README
- license and notice files
- `.env.example`
- `docker-compose.yml`
- convenience launcher scripts under `scripts/`

## Consuming Release Artifacts

Download artifacts from the project's GitHub Releases page.

Choose the artifact that matches how you want to consume the provider:

- `neopsyke-pgvector-memory-<version>-all.jar` if you want the simplest runtime path with `java -jar`
- `neopsyke-pgvector-memory-<version>-bundle.zip` if you want the jar plus launcher scripts, `.env.example`, and Docker helper files

Run the published fat jar directly:

```bash
export MISTRAL_API_KEY=your-key
java -jar neopsyke-pgvector-memory-<version>-all.jar --transport=http --port=7841
```

Or unpack the bundle and use the included launcher:

```bash
unzip neopsyke-pgvector-memory-<version>-bundle.zip
cd dist
cp .env.example .env
./scripts/run-http-provider.sh
```

For projects consuming the provider over HTTP, point them at the provider base
URL:

```text
http://127.0.0.1:7841
```

The stable consumer endpoints are:

- `GET /v1/health`
- `GET /v1/metrics`
- `POST /v1/recall`
- `POST /v1/imprint`
- `POST /v1/admin/forget`
- `POST /v1/admin/reset`

Typical consumer configuration looks like:

- provider URL: `http://127.0.0.1:7841`
- transport: `http`
- API version/path family: `/v1/...`

Use the MCP transport only if your client expects a stdio MCP server.
For most consuming projects, the intended integration path is the HTTP provider.

## Using the MCP Adapter

The MCP adapter runs as a stdio server. MCP-capable clients such as Codex or
Claude Code can launch it with either of these commands:

```bash
java -jar neopsyke-pgvector-memory-<version>-all.jar --transport=mcp
```

or, from the release bundle:

```bash
./scripts/run-mcp-provider.sh
```

The MCP server exposes these tools:

- `remember`
- `create_memory`
- `search_memory`
- `add_observations`
- `read_graph`
- `delete_observations`
- `create_entities`
- `get_memory_metrics`

### Namespaces in MCP

Yes: different namespaces are supported through MCP.

- pass `namespace` on a tool call to separate memories by project, tenant, or user
- the MCP tools also accept `tenant`, `workspace`, or `client` as aliases
- if no namespace-like field is provided, the provider falls back to its default namespace

Example namespace values:

- `project-alpha`
- `customer-acme`
- `workspace_docs`

### MCP tool examples

Store a narrative memory:

```json
{
  "tool": "remember",
  "arguments": {
    "text": "The user prefers concise explanations with concrete examples.",
    "namespace": "project-alpha",
    "write_mode": "dedupe_if_similar"
  }
}
```

Store a fact with upsert behavior:

```json
{
  "tool": "remember",
  "arguments": {
    "text": "The deployment region is eu-central-1.",
    "namespace": "project-alpha",
    "write_mode": "upsert_fact",
    "fact_subject": "deployment",
    "fact_key": "region",
    "fact_value": "eu-central-1",
    "fact_versioned_at": "2026-03-23T12:00:00Z"
  }
}
```

Store a richer memory with explicit source and confidence:

```json
{
  "tool": "create_memory",
  "arguments": {
    "content": "Customer ACME wants weekly rollout summaries.",
    "namespace": "customer-acme",
    "source": "meeting_notes",
    "confidence": 0.9,
    "write_mode": "append"
  }
}
```

Search only within one namespace:

```json
{
  "tool": "search_memory",
  "arguments": {
    "query": "What do we know about rollout preferences?",
    "namespace": "customer-acme",
    "limit": 5
  }
}
```

Add graph-style observations into a namespace:

```json
{
  "tool": "add_observations",
  "arguments": {
    "namespace": "workspace_docs",
    "observations": [
      {
        "entityName": "docs_agent",
        "contents": [
          "The docs repo uses MkDocs tags=documentation,docs",
          "Release notes are published from GitHub releases tags=release,docs"
        ]
      }
    ]
  }
}
```

### Example Prompts for Codex or Claude Code

Once the MCP server is connected, prompts like these should work in either
client:

- "Use the `remember` MCP tool to store that the user prefers short answers in namespace `project-alpha`."
- "Use `create_memory` to save this note in namespace `customer-acme` with source `support_call` and confidence `0.8`."
- "Use `search_memory` in namespace `project-alpha` to find what we know about deployment constraints."
- "Use `remember` with `write_mode=upsert_fact` to save that deployment.region is eu-central-1 in namespace `project-alpha`."

## Release verification

Each GitHub release publishes a `SHA256SUMS` file alongside the jar and bundle
zip.

Verify on Linux:

```bash
sha256sum -c SHA256SUMS
```

Verify on macOS:

```bash
shasum -a 256 -c SHA256SUMS
```

If you only downloaded one artifact, verify it directly:

```bash
shasum -a 256 neopsyke-pgvector-memory-<version>-all.jar
shasum -a 256 neopsyke-pgvector-memory-<version>-bundle.zip
```

Breaking HTTP wire changes must move from `/v1/...` to `/v2/...`.
