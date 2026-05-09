# LLM Wiki Docker — PoC Design

**Date:** 2026-05-08
**Status:** Approved
**Version:** v1

## Purpose

A privacy-first, single-user wiki app packaged as a Docker container for Docker Desktop on macOS. Proof of concept for the full v1 design: validates the end-to-end pipeline (LLM fact extraction, embedding, semantic search, BBS HTML UI, Docker packaging) with the complexity surface cut to the minimum.

The wiki memory is implemented by `@equationalapplications/core-llm-wiki`, used unmodified.

## Non-goals (PoC)

- `OutboxSQLiteAdapter` (plain `better-sqlite3` adapter used instead)
- Chokidar filesystem watcher (manual rescan button replaces it)
- `immutable/` upload form and separate source type
- MCP server
- `source_ref_map` collision tracking
- `ingest_failures` table
- `outbox` table
- Multi-user / authentication
- Browser-side JavaScript frameworks or a frontend build step

## Architecture

Single Node.js process inside a single container, launched via `docker compose up`. State lives in a single SQLite database file on a Docker volume.

```
┌─ Docker Desktop (Mac) ──────────────────────────────────┐
│  container: llm-wiki-poc                                │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Fastify (127.0.0.1:8080)                         │   │
│  │  ├─ GET  /              status + fact list        │   │
│  │  ├─ GET  /search?q=     semantic/keyword search   │   │
│  │  ├─ GET  /facts/:id     single fact detail        │   │
│  │  ├─ GET  /status        rescan progress + LLM ping│   │
│  │  ├─ POST /rescan        trigger manual rescan     │   │
│  │  └─ POST /maintenance/* librarian/heal/prune/     │   │
│  │                         reembed                   │   │
│  │                                                   │   │
│  │ Rescan (shared function)                          │   │
│  │  └─ walk wiki/, hash diff, FIFO queue, drain      │   │
│  │     called at boot + on POST /rescan              │   │
│  │                                                   │   │
│  │ Reconciler (FIFO, single worker)                  │   │
│  │  └─ WikiMemory.ingestDocument / soft-delete       │   │
│  │                                                   │   │
│  │ PlainSQLiteAdapter (better-sqlite3)               │   │
│  │  └─ implements SQLiteAdapter, no outbox           │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  Volumes (host bind mounts):                            │
│   ./data   → /app/data    SQLite file                   │
│   ./wiki   → /app/wiki    live documents                │
└─────────────────────────────────────────────────────────┘
       │
       └── outbound HTTP → user-supplied
           OpenAI-compat endpoint (LLM + embed)
```

## Components

### 1. `core-llm-wiki` (dependency)

`@equationalapplications/core-llm-wiki@^3.2.0`. Pinned, unmodified. Provides `WikiMemory`, the `SQLiteAdapter` interface, schema migrations, librarian/heal/prune/reembed maintenance jobs, and chunk-and-extract `ingestDocument` with embedding-based and MiniSearch retrieval.

### 2. `PlainSQLiteAdapter`

Wraps `better-sqlite3`. Implements the `SQLiteAdapter` interface required by core (`execAsync`, `runAsync`, `getFirstAsync`, `getAllAsync`, `withTransactionAsync`). No outbox. No special mutation tracking.

### 3. `LLMClient`

Thin OpenAI-compatible HTTP client. Reads:

- `LLM_BASE_URL` (e.g., `http://host.docker.internal:11434/v1`)
- `LLM_API_KEY` (optional; sent as `Authorization: Bearer …` if set)
- `LLM_MODEL` (e.g., `llama3.2`)
- `EMBED_MODEL` (e.g., `nomic-embed-text`)

Exposes `generateText({ systemPrompt, userPrompt })` and `embed(text)` matching core's `llmProvider` shape. Both methods target the same `LLM_BASE_URL` with their respective models. Retries on 5xx with exponential backoff (1s, 4s, 15s) up to 3 attempts; surfaces 4xx immediately.

### 4. `Rescan`

Exported function `rescan()`. Called at boot and when `POST /rescan` is received.

1. Walk `wiki/` recursively. Build `{ relpath → sha256(content) }`.
2. Query all distinct `source_ref` values from `llm_wiki_entries`.
3. Diff:
   - Present in FS, absent in DB → enqueue `add`.
   - Present in both, hash differs → enqueue `change`.
   - Absent in FS, present in DB → enqueue `unlink`.
4. Drain queue.

A simple in-memory boolean lock prevents concurrent rescans. `POST /rescan` while a rescan is running returns HTTP 409.

Rescan progress (current item, total) exposed at `/status`.

### 5. `Reconciler`

Single-worker FIFO queue. Concurrency 1. For each job:

- Compute `source_ref = sha1(relpath).slice(0, 16)`.
- For `add` / `change`:
  - Read file (UTF-8, skip silently if decode fails or > 10 MB; log to stderr).
  - Compute `source_hash = sha256(content)`.
  - Query existing facts where `source_ref == X`.
  - If any have a matching `source_hash`, skip (no change).
  - Otherwise soft-delete old facts, then call `WikiMemory.ingestDocument('default', content, { source_ref, source_hash })`.
  - Prepend `# path: <relpath>\n\n` to content for traceability.
- For `unlink`:
  - Soft-delete all facts where `source_ref == X`.

LLM errors: retry 3× with 1s/4s/15s backoff on 5xx; surface 4xx immediately. On final failure, log to stderr and skip the file. No `ingest_failures` table.

### 6. Fastify HTML server

Pure server-rendered HTML. No client JS. 1980s BBS aesthetic: monospace font, ASCII box borders, amber-on-black or green-on-black via a single inline `<style>`.

Routes:

| Method | Path | Description |
|---|---|---|
| GET | `/` | Status panel + recent events + paginated fact list + rescan button |
| GET | `/facts/:id` | Single fact detail with source ref, confidence, body |
| GET | `/search?q=` | Calls `WikiMemory.read('default', q)`, renders ranked results |
| GET | `/status` | Rescan progress, queue depth, LLM reachability ping |
| POST | `/rescan` | Triggers `rescan()`; 302 redirect to `/`; 409 if already running |
| POST | `/maintenance/librarian` | Manual trigger |
| POST | `/maintenance/heal` | Manual trigger |
| POST | `/maintenance/prune` | Manual trigger |
| POST | `/maintenance/reembed` | Manual trigger |

The rescan button is a plain HTML `<form method="POST" action="/rescan">` — no JavaScript required.

### 7. Maintenance scheduler

Core auto-runs `runLibrarian` every `autoLibrarianThreshold` events (default 20) and `runHeal` every `autoHealThreshold` events (default 100). Both also exposed as manual UI buttons. `WikiBusyError` surfaced as HTTP 409.

## Data flow

### Boot sequence

```
container start
  → PlainSQLiteAdapter opens SQLite
  → core migrations to CURRENT_SCHEMA_VERSION
  → rescan() → walk wiki/, hash diff, enqueue, drain
  → Fastify binds 127.0.0.1:8080
  → ready
```

### Manual rescan

```
user clicks Rescan button
  → POST /rescan
  → 409 if rescan already running
  → rescan() → walk wiki/, hash diff, enqueue, drain
  → 302 redirect to /
```

### Ingest (per reconciler job)

```
job { op, relpath }
  → source_ref = sha1(relpath).slice(0, 16)
  → read file → hash → compare against DB
    match → skip
    mismatch → soft-delete old facts
              → WikiMemory.ingestDocument
                → core chunks → LLM generateText → embed each fact
                → INSERT entries + events
  → librarian/heal auto-run on threshold
```

### Read (UI search)

```
GET /search?q=X
  → WikiMemory.read('default', X)
  → core: embed(X) → cosine over entries.embedding_blob
    fallback: MiniSearch keyword if embed throws (onRetrievalFallback logs to stderr)
  → render BBS-styled HTML results
```

## Configuration

Environment variables:

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | Fastify port |
| `LLM_BASE_URL` | (required) | OpenAI-compat base URL |
| `LLM_API_KEY` | (unset) | Optional bearer token |
| `LLM_MODEL` | `llama3.2` | Chat completion model |
| `EMBED_MODEL` | `nomic-embed-text` | Embeddings model |
| `WIKI_ENTITY_ID` | `default` | Single tenant id |

Volumes (declared in `docker-compose.yml`):

| Host path | Container path | Purpose |
|---|---|---|
| `./data` | `/app/data` | SQLite database file |
| `./wiki` | `/app/wiki` | Live watched documents |

## Errors / edge cases

- **LLM unreachable on ingest** — retry 3× with backoff (1s/4s/15s). On final failure, log to stderr and skip (no UI indicator in PoC; check container logs).
- **Embed fails on read** — core auto-falls back to MiniSearch keyword search; logged via `onRetrievalFallback`.
- **File too large** (> 10 MB) — log to stderr, skip silently.
- **Binary file** — UTF-8 decode test on read; on failure, skip silently.
- **Concurrent rescan** — in-memory boolean lock; second `POST /rescan` returns HTTP 409.
- **Container restart mid-rescan** — in-flight job lost; next boot rescan re-detects via hash diff and re-queues.
- **Concurrent maintenance** — core throws `WikiBusyError`; UI returns HTTP 409.

## Testing

### Unit (vitest)

- `PlainSQLiteAdapter`: `execAsync`/`runAsync`/`getFirstAsync`/`getAllAsync`/`withTransactionAsync` correctness against in-memory SQLite; rollback on error.
- `LLMClient`: mocked HTTP — retries on 5xx with backoff, surfaces 4xx immediately; `generateText` and `embed` both exercised.
- `Reconciler`: hash-same skips; hash-change soft-deletes then re-ingests; unlink soft-deletes all facts; `source_ref` encoding is deterministic; file > 10 MB skipped; binary file (UTF-8 decode fail) skipped.

### Integration (vitest)

- Fastify against a tmp directory: drop file → `rescan()` → assert facts appear via `WikiMemory.read`.
- Modify file → old facts soft-deleted, new facts present.
- Delete file → all facts for ref soft-deleted.
- `POST /rescan` while rescan in progress → 409.

### LLM stub

Tests use a deterministic stub `llmProvider` returning canned fact JSON. No real LLM calls in CI.

### Manual smoke (documented in README)

1. `docker compose up`
2. Drop a `.md` file into `./wiki/`
3. Visit `http://127.0.0.1:8080`, click Rescan
4. See fact appear in the fact list and in search results

## Repository layout

```
/
├── docker-compose.yml
├── Dockerfile
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts                  # entrypoint: boot rescan → bind Fastify
│   ├── config.ts                 # env var loading
│   ├── adapter/plain.ts          # PlainSQLiteAdapter
│   ├── llm/client.ts             # LLMClient (generateText + embed)
│   ├── reconciler/
│   │   ├── queue.ts              # FIFO single-worker queue
│   │   └── worker.ts             # per-job logic
│   ├── boot/rescan.ts            # rescan() — exported, called at boot + POST /rescan
│   └── http/
│       ├── server.ts             # Fastify setup
│       ├── routes.ts             # all route handlers
│       └── templates.ts          # BBS HTML rendering
├── data/                         # SQLite (gitignored, mounted volume)
├── wiki/                         # gitignored, mounted volume
├── docs/superpowers/specs/
│   ├── 2026-05-08-llm-wiki-docker-design.md       # full v1 design
│   └── 2026-05-08-llm-wiki-docker-poc-design.md   # this document
└── README.md
```

## Upgrade path to v1

The PoC is structured to minimize refactoring when promoting to v1:

- `PlainSQLiteAdapter` → replaced by `OutboxSQLiteAdapter` (same interface)
- `boot/rescan.ts` → boot logic reused; chokidar watcher added alongside
- `http/routes.ts` → add `/upload` route; add `immutable/` volume
- New: `watcher/` module, `mcp/server.ts`, `source_ref_map` table, `ingest_failures` table, `outbox` table
