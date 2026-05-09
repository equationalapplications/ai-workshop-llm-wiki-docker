# LLM Wiki Docker — Design

**Date:** 2026-05-08
**Status:** Approved
**Version:** v1

## Purpose

A privacy-first, single-user wiki app packaged as a Docker container for Docker Desktop on macOS. The app extracts facts from text/markdown documents and from an arbitrary code repository, reconciles facts as files change, and exposes the resulting knowledge through an HTML UI and an MCP server. All inference runs against a user-supplied OpenAI-compatible LLM endpoint (e.g., Ollama on the host).

The wiki memory is implemented by `@equationalapplications/core-llm-wiki`, used unmodified.

## Non-goals (v1)

- Multi-user / multi-tenant
- Authentication beyond localhost-only network binding
- External database synchronization (deferred to v2; outbox table is populated but not drained)
- Browser-side JavaScript frameworks or a frontend build step
- GPU-accelerated local inference inside the container

## Architecture

Single Node.js process inside a single container, launched via `docker compose up`. The process runs Fastify (HTML UI + REST endpoints), an MCP server (HTTP+SSE), filesystem watchers, an ingest queue, and a boot rescan job. All state lives in a single SQLite database file on a Docker volume.

```
┌─ Docker Desktop (Mac) ──────────────────────────────────┐
│  container: llm-wiki                                    │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Fastify (127.0.0.1:8080)                         │   │
│  │  ├─ HTML routes (BBS UI)                         │   │
│  │  ├─ POST /upload  → writes immutable/            │   │
│  │  └─ /search, /facts/:id, /status, /maintenance   │   │
│  │                                                  │   │
│  │ MCP server (127.0.0.1:8081, HTTP+SSE)            │   │
│  │  └─ tools + resources over wiki state            │   │
│  │                                                  │   │
│  │ Watcher (chokidar)                               │   │
│  │  ├─ wiki/   (recursive, .wikiignore)             │   │
│  │  └─ immutable/ (flat, hash-dedupe)               │   │
│  │   2s debounce → ingest queue                     │   │
│  │                                                  │   │
│  │ Reconciler (FIFO, single worker)                 │   │
│  │  └─ WikiMemory.ingestDocument / soft-delete      │   │
│  │                                                  │   │
│  │ OutboxSQLiteAdapter (better-sqlite3)             │   │
│  │  └─ mirrors wiki-table writes → outbox           │   │
│  │                                                  │   │
│  │ Boot rescan (full hash compare)                  │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  Volumes (host bind mounts):                            │
│   ./data/wiki.db        SQLite + outbox                 │
│   ./immutable/          uploaded docs (immutable)       │
│   ./wiki/               arbitrary repo (live)           │
└─────────────────────────────────────────────────────────┘
       │
       └── outbound HTTP → user-supplied
           OpenAI-compat endpoint (LLM + embed)
```

## Components

### 1. `core-llm-wiki` (dependency)

`@equationalapplications/core-llm-wiki@^3.2.0`. Pinned, unmodified. Provides `WikiMemory`, the `SQLiteAdapter` interface, schema migrations, librarian/heal/prune/reembed maintenance jobs, and chunk-and-extract `ingestDocument` with embedding-based and MiniSearch retrieval.

### 2. `OutboxSQLiteAdapter`

Wraps `better-sqlite3`. Implements the `SQLiteAdapter` interface required by core (`execAsync`, `runAsync`, `getFirstAsync`, `getAllAsync`, `withTransactionAsync`).

On every mutation, parses the SQL statement to detect operation type (INSERT/UPDATE/DELETE) and target table. If the target is one of `llm_wiki_entries`, `llm_wiki_tasks`, or `llm_wiki_events`, appends a row to `outbox` in the same transaction:

```sql
CREATE TABLE outbox (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  op TEXT NOT NULL,           -- 'insert' | 'update' | 'delete'
  table_name TEXT NOT NULL,
  row_id TEXT,                -- target row primary key when known
  payload_json TEXT NOT NULL, -- bind values + statement metadata
  created_at INTEGER NOT NULL
);
CREATE INDEX outbox_created_idx ON outbox(created_at);
```

No consumer in v1. Capped at 100,000 rows; oldest pruned with a warning. v2 will add a Prisma replay worker that drains this table.

### 3. `LLMClient`

Thin OpenAI-compatible HTTP client. Reads:

- `LLM_BASE_URL` (e.g., `http://host.docker.internal:11434/v1`)
- `LLM_API_KEY` (optional; sent as `Authorization: Bearer …` if set)
- `LLM_MODEL` (e.g., `llama3.2`)
- `EMBED_MODEL` (e.g., `nomic-embed-text`)

Exposes `generateText({ systemPrompt, userPrompt })` and `embed(text)` matching core's `llmProvider` shape. Retries on 5xx with exponential backoff (1s, 4s, 15s) up to 3 attempts; surfaces 4xx immediately.

### 4. `Watcher`

Two chokidar instances:

- `wiki/` — recursive, honors `wiki/.wikiignore` (gitignore-style via the `ignore` npm package). On parse error, falls back to default ignore set: `.git`, `node_modules`, `dist`, `.DS_Store`, common binary extensions.
- `immutable/` — flat (one level), no ignore file.

Both apply 2-second debounce per absolute path. Emits ingest jobs `{ op: 'add' | 'change' | 'unlink', sourceDir, relpath }`.

### 5. `Reconciler`

Single-worker FIFO queue. Concurrency 1. For each job:

- Compute `source_ref = sha1(relpath).slice(0, 16)` (sliced for normalizeSourceRef-compat — alphanumeric only).
- For `add` / `change`:
  - Read file (UTF-8, skip if decode fails or > 10 MB).
  - Compute `source_hash = sha256(content)`.
  - Query existing facts where `source_ref == X`.
  - If any have a different `source_hash`, soft-delete them.
  - Call `WikiMemory.ingestDocument('default', content, { source_ref, source_hash })`.
  - Prepend `# path: <relpath>\n\n` to content for traceability in fact bodies.
  - For `immutable/` files, force `source_type = 'immutable_document'` (override default `librarian_inferred`).
- For `unlink`:
  - Soft-delete all facts where `source_ref == X`.

Collision handling: if a `source_ref` collision is detected on insert (two paths sha1-truncate to the same 16 chars), append `-1`, `-2`, etc., and persist the mapping in a `source_ref_map(relpath, source_ref)` table.

### 6. `BootRescan`

On startup, before Fastify binds:

1. Run core's migrations to `CURRENT_SCHEMA_VERSION`.
2. Ensure `outbox` and `source_ref_map` and `ingest_failures` tables exist.
3. Walk `wiki/` (respecting `.wikiignore`) and `immutable/`. Build `{ relpath: source_hash }`.
4. Query all distinct `source_ref` values from `llm_wiki_entries` (joined back to `source_ref_map` for path resolution).
5. Diff:
   - Present in FS, absent in DB → enqueue `add`.
   - Present in both, hash differs → enqueue `change`.
   - Absent in FS, present in DB → enqueue `unlink`.
6. Drain queue.
7. Start chokidar watchers, bind Fastify, bind MCP.

Boot rescan progress (current item, total, ETA) exposed at `/status`.

### 7. Fastify HTML server

Pure server-rendered HTML. No client JS. 1980s BBS aesthetic: monospace font, ASCII box borders, amber-on-black or green-on-black via a single inline `<style>`.

Routes:

| Method | Path | Description |
|---|---|---|
| GET | `/` | Status panel + recent events + paginated fact list |
| GET | `/facts/:id` | Single fact detail with source ref, confidence, body |
| GET | `/search?q=` | Calls `WikiMemory.read('default', q)`, renders ranked results |
| GET | `/upload` | HTML form for `immutable/` upload |
| POST | `/upload` | Multipart, writes file to `immutable/<safename>`, redirects to `/` |
| GET | `/status` | Boot rescan progress, queue depth, LLM reachability ping, outbox depth |
| POST | `/maintenance/librarian` | Manual trigger |
| POST | `/maintenance/heal` | Manual trigger |
| POST | `/maintenance/prune` | Manual trigger |
| POST | `/maintenance/reembed` | Manual trigger |

Filename sanitization on upload: lowercase, replace whitespace with `_`, allow only `[a-z0-9._-]`, reject leading dots. Reject extensions outside an allowlist (`.md`, `.txt`, `.markdown`).

### 8. Maintenance scheduler

Core auto-runs `runLibrarian` every `autoLibrarianThreshold` events (default 20) and `runHeal` every `autoHealThreshold` events (default 100). Both also exposed as manual UI buttons (POST routes above) and as MCP tools. `WikiBusyError` surfaced as HTTP 409 / MCP error.

### 9. MCP server

`@modelcontextprotocol/sdk` server over HTTP+SSE on `127.0.0.1:8081` (override via `MCP_PORT`). Bound to localhost. No auth.

Tools:

- `wiki_read({ query })` — semantic + keyword search via `WikiMemory.read`
- `wiki_ingest_text({ title, body })` — writes a synthetic file into `immutable/<title>.md`, picked up by watcher
- `wiki_list_facts({ limit?, offset? })`
- `wiki_list_tasks({ status? })`
- `wiki_create_task({ description, priority? })`
- `wiki_complete_task({ id })`
- `wiki_run_librarian()`
- `wiki_run_heal()`

Resources:

- `wiki://fact/<id>` — full fact body
- `wiki://task/<id>` — task detail

All tools operate on `entityId = 'default'`. Multi-tenant deferred.

## Data flow

### Ingest (file change)

```
fs event → chokidar (debounce 2s) → ingest queue
  → Reconciler:
      hash file → query existing source_hash for source_ref
        match → skip
        mismatch → soft-delete old facts → WikiMemory.ingestDocument
          → core chunks → LLM generateText → embed each fact
          → INSERT entries + events
          → OutboxSQLiteAdapter mirrors INSERTs into outbox (same tx)
  → librarian/heal auto-run on threshold
```

### Read (UI search)

```
GET /search?q=X → WikiMemory.read('default', X)
  → core: embed(X) → cosine over entries.embedding_blob
    (preFilterLimit MiniSearch first if configured)
    → fallback: MiniSearch keyword if embed throws
       → onRetrievalFallback logs to stderr
  → render BBS-styled HTML results
```

### Upload

```
POST /upload (multipart) → validate filename + extension
  → write immutable/<safename>
  → 302 redirect to /
  → chokidar add event
  → Reconciler ingests with source_type='immutable_document'
```

### Boot

```
container start → core migrations → ensure ancillary tables
  → BootRescan: walk dirs, hash files, diff against DB, enqueue events
  → drain queue → start watchers → bind 127.0.0.1:8080 + 127.0.0.1:8081 → ready
```

## Configuration

Environment variables:

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | HTML/REST port |
| `MCP_PORT` | `8081` | MCP HTTP+SSE port |
| `LLM_BASE_URL` | (required) | OpenAI-compat base URL |
| `LLM_API_KEY` | (unset) | Optional bearer token |
| `LLM_MODEL` | `llama3.2` | Chat completion model |
| `EMBED_MODEL` | `nomic-embed-text` | Embeddings model |
| `WIKI_ENTITY_ID` | `default` | Single tenant id; reserved for v2 multi-tenant |

Volumes (declared in `docker-compose.yml`):

| Host path | Container path | Purpose |
|---|---|---|
| `./data` | `/app/data` | SQLite database file |
| `./immutable` | `/app/immutable` | Uploaded immutable documents |
| `./wiki` | `/app/wiki` | Live watched repository |

## Errors / edge cases

- **LLM unreachable on ingest** — retry 3× with backoff (1s/4s/15s). On final failure, write a row to `ingest_failures(path, hash, error, ts)`. Status panel shows count and a retry button.
- **Embed fails on read** — core auto-falls back to MiniSearch keyword search; failure logged via `onRetrievalFallback`.
- **File too large** (> 10 MB) — log, skip, write to `ingest_failures`.
- **Binary file slipped past `.wikiignore`** — UTF-8 decode test on read; on failure, skip silently.
- **Rapid file thrash** (e.g., `git checkout` between branches) — 2s debounce + serial queue absorb. UI/MCP remain responsive because the queue runs in a separate worker.
- **Boot rescan during file edits** — boot rescan completes before chokidar watchers start; no double-processing.
- **Outbox unbounded growth** (no v1 consumer) — capped at 100,000 rows; oldest pruned with warning. v2 worker will drain.
- **`wiki/.wikiignore` malformed** — log parse error, fall back to default ignore set.
- **Concurrent maintenance** — core throws `WikiBusyError`; UI returns HTTP 409, MCP returns tool error.
- **Container restart mid-ingest** — in-flight job lost; boot rescan re-detects via hash diff and re-queues.
- **`source_ref` collision** — append disambiguator (`<hash16>-1`); persist mapping in `source_ref_map`.

## Testing

### Unit (vitest)

- `OutboxSQLiteAdapter`: INSERT/UPDATE/DELETE on wiki tables produces correct outbox rows in the same transaction; non-wiki tables ignored; rollback on error rolls back outbox too.
- `Reconciler`: hash-same skips, hash-change soft-deletes then re-ingests, unlink soft-deletes all facts; `source_ref` encoding is deterministic; collision handling exercised.
- `.wikiignore` parser: gitignore semantics; malformed file falls back to default ignore set.
- `LLMClient`: mocked HTTP — retries on 5xx with backoff, surfaces 4xx immediately.

### Integration

- Spin Fastify against a tmp directory: drop file → assert facts appear via `WikiMemory.read`.
- Modify file → old facts soft-deleted, new facts present, outbox has correct ops.
- Delete file → all facts for ref soft-deleted.
- Boot rescan: pre-populate DB and filesystem in known states; verify reconciliation correct.
- MCP: spawn server, call each tool via SDK client, verify wiki state mutations.

### Manual smoke (documented in README)

- `docker compose up`, drop `.md` in `wiki/`, browse `http://127.0.0.1:8080`, see fact appear.
- Edit file, refresh, see updated fact.
- Upload via UI form, verify in `immutable/`.

### LLM stub

Tests use a deterministic stub `llmProvider` returning canned fact JSON. No real LLM calls in CI.

### Out of scope (v1)

- E2E browser tests (pure HTML, no JS, manual smoke is sufficient).

## Repository layout

```
/
├── docker-compose.yml
├── Dockerfile
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts                  # entrypoint: boot rescan → bind servers
│   ├── adapter/outbox.ts         # OutboxSQLiteAdapter
│   ├── llm/client.ts             # LLMClient
│   ├── watcher/                  # chokidar setup + .wikiignore parser
│   ├── reconciler/               # ingest queue + worker
│   ├── boot/rescan.ts            # full hash diff
│   ├── http/                     # Fastify routes + HTML templates + CSS
│   ├── mcp/server.ts             # MCP server + tool/resource handlers
│   └── config.ts                 # env var loading
├── data/                         # SQLite (gitignored, mounted volume)
├── immutable/                    # gitignored, mounted volume
├── wiki/                         # gitignored, mounted volume
├── docs/superpowers/specs/2026-05-08-llm-wiki-docker-design.md
└── README.md
```

## v2 roadmap (not in this spec)

- Prisma replay worker draining `outbox` to Postgres (or any Prisma-supported DB).
- Multi-tenant `entityId` support.
- Optional auth for non-localhost deployments.
- MCP improvements: streaming, additional tools, prompt templates.
