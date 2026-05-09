# LLM Wiki Docker Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build privacy-first single-user wiki app in Docker that extracts facts from files using user-supplied LLM, exposes HTML UI and MCP server

**Architecture:** Single Node.js process running Fastify (HTML + REST), MCP server (HTTP+SSE), filesystem watchers (chokidar), FIFO reconciler queue, and boot rescan. Core wiki logic delegated to `@equationalapplications/core-llm-wiki`. All state in SQLite on Docker volume.

**Tech Stack:** TypeScript, Node 24 (alpine), Fastify 5.1.3, better-sqlite3, chokidar, @modelcontextprotocol/sdk, @equationalapplications/core-llm-wiki, Docker Desktop 4.72.0, Compose v5.1.3

---

## File Structure

```
/
├── package.json              # deps + scripts
├── tsconfig.json             # TS config
├── vitest.config.ts          # test config
├── Dockerfile                # container build
├── docker-compose.yml        # orchestration
├── .dockerignore             # build excludes
├── src/
│   ├── index.ts              # entrypoint
│   ├── config.ts             # env var loading
│   ├── adapter/
│   │   └── outbox.ts         # OutboxSQLiteAdapter
│   ├── llm/
│   │   └── client.ts         # LLMClient
│   ├── watcher/
│   │   ├── index.ts          # chokidar setup
│   │   └── ignore-parser.ts  # .wikiignore parser
│   ├── reconciler/
│   │   ├── index.ts          # queue + worker
│   │   └── hash.ts           # source_ref + source_hash utils
│   ├── boot/
│   │   └── rescan.ts         # startup reconciliation
│   ├── http/
│   │   ├── server.ts         # Fastify setup
│   │   ├── routes/
│   │   │   ├── index.ts      # GET /
│   │   │   ├── facts.ts      # GET /facts/:id
│   │   │   ├── search.ts     # GET /search
│   │   │   ├── upload.ts     # GET/POST /upload
│   │   │   ├── status.ts     # GET /status
│   │   │   └── maintenance.ts # POST /maintenance/*
│   │   ├── templates/
│   │   │   ├── layout.ts     # HTML shell + BBS CSS
│   │   │   ├── index.ts      # home template
│   │   │   ├── fact.ts       # fact detail template
│   │   │   ├── search.ts     # search results template
│   │   │   ├── upload.ts     # upload form template
│   │   │   └── status.ts     # status panel template
│   ├── mcp/
│   │   └── server.ts         # MCP HTTP+SSE server
│   └── types.ts              # shared types
├── tests/
│   ├── unit/
│   │   ├── adapter.test.ts
│   │   ├── llm-client.test.ts
│   │   ├── reconciler.test.ts
│   │   └── ignore-parser.test.ts
│   ├── integration/
│   │   ├── ingest-flow.test.ts
│   │   ├── boot-rescan.test.ts
│   │   └── mcp-server.test.ts
│   └── fixtures/
│       └── stub-llm.ts       # deterministic LLM stub
└── README.md                 # setup + usage
```

---

## Task 1: Project scaffold + dependencies

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `vitest.config.ts`
- Create: `.gitignore`
- Create: `.dockerignore`

- [ ] **Step 1: Initialize package.json with dependencies**

```json
{
  "name": "llm-wiki-docker",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "@equationalapplications/core-llm-wiki": "^3.2.0",
    "@fastify/multipart": "^9.0.0",
    "@modelcontextprotocol/sdk": "^1.0.0",
    "better-sqlite3": "^11.0.0",
    "chokidar": "^4.0.0",
    "fastify": "^5.1.3",
    "ignore": "^6.0.0",
    "p-queue": "^8.0.0"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.11",
    "@types/node": "^24.0.0",
    "tsx": "^4.0.0",
    "typescript": "^5.6.0",
    "vitest": "^2.0.0"
  }
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "types": ["node", "better-sqlite3"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

- [ ] **Step 3: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: ['tests/**/*.test.ts']
  }
});
```

- [ ] **Step 4: Create .gitignore**

```
node_modules/
dist/
data/
immutable/
wiki/
*.db
*.db-shm
*.db-wal
.DS_Store
```

- [ ] **Step 5: Create .dockerignore**

```
node_modules/
dist/
data/
immutable/
wiki/
tests/
*.md
.git/
.gitignore
tsconfig.json
vitest.config.ts
```

- [ ] **Step 6: Install dependencies**

Run: `npm install`
Expected: dependencies installed successfully

- [ ] **Step 7: Create src and tests directory structure**

Run: `mkdir -p src/{adapter,llm,watcher,reconciler,boot,http/routes,http/templates,mcp} tests/{unit,integration,fixtures}`
Expected: directories created

- [ ] **Step 8: Commit**

```bash
git add package.json tsconfig.json vitest.config.ts .gitignore .dockerignore
git commit -m "feat: project scaffold with dependencies"
```

---

## Task 2: Config loader

**Files:**
- Create: `src/config.ts`
- Create: `src/types.ts`
- Create: `tests/unit/config.test.ts`

- [ ] **Step 1: Write test for config loading**

```typescript
// tests/unit/config.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { loadConfig } from '../../src/config';

describe('loadConfig', () => {
  const originalEnv = process.env;

  beforeEach(() => {
    process.env = { ...originalEnv };
  });

  afterEach(() => {
    process.env = originalEnv;
  });

  it('loads all required config from env', () => {
    process.env.LLM_BASE_URL = 'http://host.docker.internal:11434/v1';
    process.env.LLM_MODEL = 'llama3.2';
    process.env.EMBED_MODEL = 'nomic-embed-text';

    const config = loadConfig();

    expect(config.port).toBe(8080);
    expect(config.mcpPort).toBe(8081);
    expect(config.llm.baseUrl).toBe('http://host.docker.internal:11434/v1');
    expect(config.llm.model).toBe('llama3.2');
    expect(config.llm.embedModel).toBe('nomic-embed-text');
    expect(config.llm.apiKey).toBeUndefined();
    expect(config.entityId).toBe('default');
  });

  it('overrides defaults with env vars', () => {
    process.env.PORT = '9000';
    process.env.MCP_PORT = '9001';
    process.env.LLM_BASE_URL = 'http://llm:8000';
    process.env.LLM_API_KEY = 'secret';
    process.env.WIKI_ENTITY_ID = 'test-tenant';

    const config = loadConfig();

    expect(config.port).toBe(9000);
    expect(config.mcpPort).toBe(9001);
    expect(config.llm.apiKey).toBe('secret');
    expect(config.entityId).toBe('test-tenant');
  });

  it('throws if LLM_BASE_URL missing', () => {
    delete process.env.LLM_BASE_URL;

    expect(() => loadConfig()).toThrow('LLM_BASE_URL required');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/unit/config.test.ts`
Expected: FAIL with "Cannot find module '../../src/config'"

- [ ] **Step 3: Define types**

```typescript
// src/types.ts
export interface Config {
  port: number;
  mcpPort: number;
  llm: {
    baseUrl: string;
    apiKey?: string;
    model: string;
    embedModel: string;
  };
  entityId: string;
  paths: {
    db: string;
    immutable: string;
    wiki: string;
  };
}

export interface IngestJob {
  op: 'add' | 'change' | 'unlink';
  sourceDir: 'wiki' | 'immutable';
  relpath: string;
}

export interface OutboxRow {
  id: number;
  op: 'insert' | 'update' | 'delete';
  table_name: string;
  row_id: string | null;
  payload_json: string;
  created_at: number;
}
```

- [ ] **Step 4: Implement config loader**

```typescript
// src/config.ts
import { Config } from './types.js';

export function loadConfig(): Config {
  const llmBaseUrl = process.env.LLM_BASE_URL;
  if (!llmBaseUrl) {
    throw new Error('LLM_BASE_URL required');
  }

  return {
    port: parseInt(process.env.PORT || '8080', 10),
    mcpPort: parseInt(process.env.MCP_PORT || '8081', 10),
    llm: {
      baseUrl: llmBaseUrl,
      apiKey: process.env.LLM_API_KEY,
      model: process.env.LLM_MODEL || 'llama3.2',
      embedModel: process.env.EMBED_MODEL || 'nomic-embed-text'
    },
    entityId: process.env.WIKI_ENTITY_ID || 'default',
    paths: {
      db: process.env.DB_PATH || '/app/data/wiki.db',
      immutable: process.env.IMMUTABLE_PATH || '/app/immutable',
      wiki: process.env.WIKI_PATH || '/app/wiki'
    }
  };
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm test tests/unit/config.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/config.ts src/types.ts tests/unit/config.test.ts
git commit -m "feat: config loader with env var validation"
```

---

## Task 3: LLMClient with retry logic

**Files:**
- Create: `src/llm/client.ts`
- Create: `tests/unit/llm-client.test.ts`

- [ ] **Step 1: Write test for LLMClient**

```typescript
// tests/unit/llm-client.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { LLMClient } from '../../src/llm/client';

describe('LLMClient', () => {
  const config = {
    baseUrl: 'http://localhost:11434/v1',
    apiKey: undefined,
    model: 'llama3.2',
    embedModel: 'nomic-embed-text'
  };

  beforeEach(() => {
    vi.restoreAllMocks();
  });

  it('generates text with system and user prompts', async () => {
    global.fetch = vi.fn().mockResolvedValueOnce({
      ok: true,
      status: 200,
      json: async () => ({
        choices: [{ message: { content: 'generated text' } }]
      })
    });

    const client = new LLMClient(config);
    const result = await client.generateText({
      systemPrompt: 'You are helpful',
      userPrompt: 'Say hello'
    });

    expect(result).toBe('generated text');
    expect(fetch).toHaveBeenCalledWith(
      'http://localhost:11434/v1/chat/completions',
      expect.objectContaining({
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          model: 'llama3.2',
          messages: [
            { role: 'system', content: 'You are helpful' },
            { role: 'user', content: 'Say hello' }
          ]
        })
      })
    );
  });

  it('includes Authorization header when apiKey set', async () => {
    const configWithKey = { ...config, apiKey: 'secret' };
    global.fetch = vi.fn().mockResolvedValueOnce({
      ok: true,
      status: 200,
      json: async () => ({ choices: [{ message: { content: 'ok' } }] })
    });

    const client = new LLMClient(configWithKey);
    await client.generateText({ systemPrompt: 'sys', userPrompt: 'usr' });

    expect(fetch).toHaveBeenCalledWith(
      expect.any(String),
      expect.objectContaining({
        headers: {
          'Content-Type': 'application/json',
          Authorization: 'Bearer secret'
        }
      })
    );
  });

  it('retries on 5xx with exponential backoff', async () => {
    global.fetch = vi
      .fn()
      .mockResolvedValueOnce({ ok: false, status: 503 })
      .mockResolvedValueOnce({ ok: false, status: 500 })
      .mockResolvedValueOnce({
        ok: true,
        status: 200,
        json: async () => ({ choices: [{ message: { content: 'ok' } }] })
      });

    const client = new LLMClient(config);
    const result = await client.generateText({
      systemPrompt: 's',
      userPrompt: 'u'
    });

    expect(result).toBe('ok');
    expect(fetch).toHaveBeenCalledTimes(3);
  });

  it('throws on 4xx without retry', async () => {
    global.fetch = vi.fn().mockResolvedValueOnce({ ok: false, status: 400 });

    const client = new LLMClient(config);

    await expect(
      client.generateText({ systemPrompt: 's', userPrompt: 'u' })
    ).rejects.toThrow('LLM request failed with status 400');
    expect(fetch).toHaveBeenCalledTimes(1);
  });

  it('throws after 3 failed retries', async () => {
    global.fetch = vi.fn().mockResolvedValue({ ok: false, status: 503 });

    const client = new LLMClient(config);

    await expect(
      client.generateText({ systemPrompt: 's', userPrompt: 'u' })
    ).rejects.toThrow('LLM request failed after 3 attempts');
    expect(fetch).toHaveBeenCalledTimes(3);
  });

  it('generates embeddings', async () => {
    global.fetch = vi.fn().mockResolvedValueOnce({
      ok: true,
      status: 200,
      json: async () => ({
        data: [{ embedding: [0.1, 0.2, 0.3] }]
      })
    });

    const client = new LLMClient(config);
    const result = await client.embed('test text');

    expect(result).toEqual([0.1, 0.2, 0.3]);
    expect(fetch).toHaveBeenCalledWith(
      'http://localhost:11434/v1/embeddings',
      expect.objectContaining({
        method: 'POST',
        body: JSON.stringify({
          model: 'nomic-embed-text',
          input: 'test text'
        })
      })
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/unit/llm-client.test.ts`
Expected: FAIL with "Cannot find module '../../src/llm/client'"

- [ ] **Step 3: Implement LLMClient**

```typescript
// src/llm/client.ts
interface LLMConfig {
  baseUrl: string;
  apiKey?: string;
  model: string;
  embedModel: string;
}

interface GenerateTextOptions {
  systemPrompt: string;
  userPrompt: string;
}

export class LLMClient {
  constructor(private config: LLMConfig) {}

  async generateText(opts: GenerateTextOptions): Promise<string> {
    const url = `${this.config.baseUrl}/chat/completions`;
    const body = {
      model: this.config.model,
      messages: [
        { role: 'system', content: opts.systemPrompt },
        { role: 'user', content: opts.userPrompt }
      ]
    };

    const result = await this.fetchWithRetry(url, body);
    return result.choices[0].message.content;
  }

  async embed(text: string): Promise<number[]> {
    const url = `${this.config.baseUrl}/embeddings`;
    const body = {
      model: this.config.embedModel,
      input: text
    };

    const result = await this.fetchWithRetry(url, body);
    return result.data[0].embedding;
  }

  private async fetchWithRetry(url: string, body: unknown): Promise<any> {
    const delays = [1000, 4000, 15000];
    let lastError: Error | undefined;

    for (let attempt = 0; attempt < 3; attempt++) {
      try {
        const headers: Record<string, string> = {
          'Content-Type': 'application/json'
        };
        if (this.config.apiKey) {
          headers.Authorization = `Bearer ${this.config.apiKey}`;
        }

        const response = await fetch(url, {
          method: 'POST',
          headers,
          body: JSON.stringify(body)
        });

        if (response.ok) {
          return await response.json();
        }

        if (response.status >= 400 && response.status < 500) {
          throw new Error(`LLM request failed with status ${response.status}`);
        }

        lastError = new Error(
          `LLM request failed with status ${response.status}`
        );

        if (attempt < 2) {
          await new Promise(resolve => setTimeout(resolve, delays[attempt]));
        }
      } catch (err) {
        if (err instanceof Error && err.message.includes('status 4')) {
          throw err;
        }
        lastError = err as Error;
        if (attempt < 2) {
          await new Promise(resolve => setTimeout(resolve, delays[attempt]));
        }
      }
    }

    throw new Error('LLM request failed after 3 attempts');
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/unit/llm-client.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/llm/client.ts tests/unit/llm-client.test.ts
git commit -m "feat: LLMClient with retry and backoff"
```

---

## Task 4: OutboxSQLiteAdapter

**Files:**
- Create: `src/adapter/outbox.ts`
- Create: `tests/unit/adapter.test.ts`

- [ ] **Step 1: Write test for OutboxSQLiteAdapter**

```typescript
// tests/unit/adapter.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import Database from 'better-sqlite3';
import { OutboxSQLiteAdapter } from '../../src/adapter/outbox';
import { mkdtempSync, rmSync } from 'fs';
import { tmpdir } from 'os';
import { join } from 'path';

describe('OutboxSQLiteAdapter', () => {
  let tmpDir: string;
  let db: Database.Database;
  let adapter: OutboxSQLiteAdapter;

  beforeEach(() => {
    tmpDir = mkdtempSync(join(tmpdir(), 'outbox-test-'));
    db = new Database(join(tmpDir, 'test.db'));

    db.exec(`
      CREATE TABLE outbox (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        op TEXT NOT NULL,
        table_name TEXT NOT NULL,
        row_id TEXT,
        payload_json TEXT NOT NULL,
        created_at INTEGER NOT NULL
      );
      CREATE INDEX outbox_created_idx ON outbox(created_at);

      CREATE TABLE llm_wiki_entries (
        id INTEGER PRIMARY KEY,
        content TEXT
      );
    `);

    adapter = new OutboxSQLiteAdapter(db);
  });

  afterEach(() => {
    db.close();
    rmSync(tmpDir, { recursive: true, force: true });
  });

  it('mirrors INSERT to outbox', async () => {
    await adapter.runAsync(
      'INSERT INTO llm_wiki_entries (id, content) VALUES (?, ?)',
      [1, 'test']
    );

    const outboxRows = db
      .prepare('SELECT * FROM outbox ORDER BY id')
      .all() as any[];

    expect(outboxRows).toHaveLength(1);
    expect(outboxRows[0].op).toBe('insert');
    expect(outboxRows[0].table_name).toBe('llm_wiki_entries');
    expect(outboxRows[0].row_id).toBe('1');
    expect(JSON.parse(outboxRows[0].payload_json)).toEqual({
      binds: [1, 'test']
    });
  });

  it('mirrors UPDATE to outbox', async () => {
    db.prepare('INSERT INTO llm_wiki_entries (id, content) VALUES (?, ?)').run(
      1,
      'original'
    );

    await adapter.runAsync(
      'UPDATE llm_wiki_entries SET content = ? WHERE id = ?',
      ['updated', 1]
    );

    const outboxRows = db
      .prepare('SELECT * FROM outbox ORDER BY id')
      .all() as any[];

    expect(outboxRows).toHaveLength(1);
    expect(outboxRows[0].op).toBe('update');
    expect(outboxRows[0].table_name).toBe('llm_wiki_entries');
  });

  it('mirrors DELETE to outbox', async () => {
    db.prepare('INSERT INTO llm_wiki_entries (id, content) VALUES (?, ?)').run(
      1,
      'test'
    );

    await adapter.runAsync('DELETE FROM llm_wiki_entries WHERE id = ?', [1]);

    const outboxRows = db
      .prepare('SELECT * FROM outbox ORDER BY id')
      .all() as any[];

    expect(outboxRows).toHaveLength(1);
    expect(outboxRows[0].op).toBe('delete');
  });

  it('ignores non-wiki tables', async () => {
    db.exec('CREATE TABLE other_table (id INTEGER PRIMARY KEY)');
    await adapter.runAsync('INSERT INTO other_table (id) VALUES (?)', [1]);

    const outboxRows = db.prepare('SELECT * FROM outbox').all();
    expect(outboxRows).toHaveLength(0);
  });

  it('rolls back outbox on transaction error', async () => {
    try {
      await adapter.withTransactionAsync(async () => {
        await adapter.runAsync(
          'INSERT INTO llm_wiki_entries (id, content) VALUES (?, ?)',
          [1, 'test']
        );
        throw new Error('test rollback');
      });
    } catch (err) {
      // expected
    }

    const entries = db.prepare('SELECT * FROM llm_wiki_entries').all();
    const outboxRows = db.prepare('SELECT * FROM outbox').all();

    expect(entries).toHaveLength(0);
    expect(outboxRows).toHaveLength(0);
  });

  it('prunes outbox when over 100k rows', async () => {
    const stmt = db.prepare('INSERT INTO outbox (op, table_name, payload_json, created_at) VALUES (?, ?, ?, ?)');
    for (let i = 0; i < 100005; i++) {
      stmt.run('insert', 'test', '{}', Date.now() - (100005 - i));
    }

    await adapter.runAsync(
      'INSERT INTO llm_wiki_entries (id, content) VALUES (?, ?)',
      [1, 'trigger prune']
    );

    const count = db.prepare('SELECT COUNT(*) as cnt FROM outbox').get() as any;
    expect(count.cnt).toBe(100000);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/unit/adapter.test.ts`
Expected: FAIL with "Cannot find module '../../src/adapter/outbox'"

- [ ] **Step 3: Implement OutboxSQLiteAdapter**

```typescript
// src/adapter/outbox.ts
import type Database from 'better-sqlite3';

interface SQLiteAdapter {
  execAsync(sql: string): Promise<void>;
  runAsync(sql: string, params?: any[]): Promise<{ changes: number; lastID: number }>;
  getFirstAsync<T = any>(sql: string, params?: any[]): Promise<T | undefined>;
  getAllAsync<T = any>(sql: string, params?: any[]): Promise<T[]>;
  withTransactionAsync<T>(fn: () => Promise<T>): Promise<T>;
}

const WIKI_TABLES = ['llm_wiki_entries', 'llm_wiki_tasks', 'llm_wiki_events'];
const MAX_OUTBOX_ROWS = 100000;

export class OutboxSQLiteAdapter implements SQLiteAdapter {
  constructor(private db: Database.Database) {}

  async execAsync(sql: string): Promise<void> {
    this.db.exec(sql);
  }

  async runAsync(
    sql: string,
    params?: any[]
  ): Promise<{ changes: number; lastID: number }> {
    const result = this.db.prepare(sql).run(...(params || []));

    const op = this.extractOp(sql);
    const tableName = this.extractTableName(sql);

    if (op && tableName && WIKI_TABLES.includes(tableName)) {
      this.appendOutbox(op, tableName, params || []);
    }

    this.pruneOutboxIfNeeded();

    return { changes: result.changes, lastID: Number(result.lastInsertRowid) };
  }

  async getFirstAsync<T = any>(
    sql: string,
    params?: any[]
  ): Promise<T | undefined> {
    return this.db.prepare(sql).get(...(params || [])) as T | undefined;
  }

  async getAllAsync<T = any>(sql: string, params?: any[]): Promise<T[]> {
    return this.db.prepare(sql).all(...(params || [])) as T[];
  }

  async withTransactionAsync<T>(fn: () => Promise<T>): Promise<T> {
    this.db.exec('BEGIN');
    try {
      const result = await fn();
      this.db.exec('COMMIT');
      return result;
    } catch (err) {
      this.db.exec('ROLLBACK');
      throw err;
    }
  }

  private extractOp(sql: string): 'insert' | 'update' | 'delete' | null {
    const normalized = sql.trim().toLowerCase();
    if (normalized.startsWith('insert')) return 'insert';
    if (normalized.startsWith('update')) return 'update';
    if (normalized.startsWith('delete')) return 'delete';
    return null;
  }

  private extractTableName(sql: string): string | null {
    const normalized = sql.trim().toLowerCase();
    const insertMatch = normalized.match(/insert\s+into\s+(\w+)/);
    if (insertMatch) return insertMatch[1];

    const updateMatch = normalized.match(/update\s+(\w+)/);
    if (updateMatch) return updateMatch[1];

    const deleteMatch = normalized.match(/delete\s+from\s+(\w+)/);
    if (deleteMatch) return deleteMatch[1];

    return null;
  }

  private appendOutbox(op: string, tableName: string, binds: any[]): void {
    const rowId = op === 'insert' && binds.length > 0 ? String(binds[0]) : null;
    const payloadJson = JSON.stringify({ binds });

    this.db
      .prepare(
        'INSERT INTO outbox (op, table_name, row_id, payload_json, created_at) VALUES (?, ?, ?, ?, ?)'
      )
      .run(op, tableName, rowId, payloadJson, Date.now());
  }

  private pruneOutboxIfNeeded(): void {
    const countRow = this.db
      .prepare('SELECT COUNT(*) as cnt FROM outbox')
      .get() as { cnt: number };

    if (countRow.cnt > MAX_OUTBOX_ROWS) {
      const toDelete = countRow.cnt - MAX_OUTBOX_ROWS;
      this.db
        .prepare(
          'DELETE FROM outbox WHERE id IN (SELECT id FROM outbox ORDER BY created_at ASC LIMIT ?)'
        )
        .run(toDelete);

      console.warn(
        `Outbox pruned: removed ${toDelete} oldest rows (limit: ${MAX_OUTBOX_ROWS})`
      );
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/unit/adapter.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/adapter/outbox.ts tests/unit/adapter.test.ts
git commit -m "feat: OutboxSQLiteAdapter with automatic mirroring"
```

---

## Task 5: Hash utilities for source_ref

**Files:**
- Create: `src/reconciler/hash.ts`
- Create: `tests/unit/reconciler.test.ts`

- [ ] **Step 1: Write test for hash utilities**

```typescript
// tests/unit/reconciler.test.ts
import { describe, it, expect } from 'vitest';
import {
  computeSourceRef,
  computeSourceHash,
  sanitizeSourceRef
} from '../../src/reconciler/hash';

describe('hash utilities', () => {
  it('computeSourceRef produces 16-char alphanumeric hash', () => {
    const ref = computeSourceRef('path/to/file.md');
    expect(ref).toMatch(/^[a-f0-9]{16}$/);
  });

  it('computeSourceRef is deterministic', () => {
    const ref1 = computeSourceRef('same/path.txt');
    const ref2 = computeSourceRef('same/path.txt');
    expect(ref1).toBe(ref2);
  });

  it('computeSourceHash produces hex sha256', () => {
    const hash = computeSourceHash('content');
    expect(hash).toMatch(/^[a-f0-9]{64}$/);
  });

  it('computeSourceHash changes with content', () => {
    const hash1 = computeSourceHash('content A');
    const hash2 = computeSourceHash('content B');
    expect(hash1).not.toBe(hash2);
  });

  it('sanitizeSourceRef strips non-alphanumeric', () => {
    const sanitized = sanitizeSourceRef('abc-123_XYZ!@#');
    expect(sanitized).toBe('abc123XYZ');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/unit/reconciler.test.ts`
Expected: FAIL with "Cannot find module '../../src/reconciler/hash'"

- [ ] **Step 3: Implement hash utilities**

```typescript
// src/reconciler/hash.ts
import { createHash } from 'crypto';

export function computeSourceRef(relpath: string): string {
  const hash = createHash('sha1').update(relpath).digest('hex');
  return hash.slice(0, 16);
}

export function computeSourceHash(content: string): string {
  return createHash('sha256').update(content).digest('hex');
}

export function sanitizeSourceRef(ref: string): string {
  return ref.replace(/[^a-zA-Z0-9]/g, '');
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/unit/reconciler.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/reconciler/hash.ts tests/unit/reconciler.test.ts
git commit -m "feat: hash utilities for source_ref and source_hash"
```

---

## Task 6: .wikiignore parser

**Files:**
- Create: `src/watcher/ignore-parser.ts`
- Create: `tests/unit/ignore-parser.test.ts`

- [ ] **Step 1: Write test for ignore parser**

```typescript
// tests/unit/ignore-parser.test.ts
import { describe, it, expect } from 'vitest';
import { parseWikiIgnore, DEFAULT_IGNORE_PATTERNS } from '../../src/watcher/ignore-parser';

describe('parseWikiIgnore', () => {
  it('parses valid gitignore syntax', () => {
    const content = `
# comment
node_modules/
*.log
!important.log
dist
`;
    const ignore = parseWikiIgnore(content);

    expect(ignore.ignores('node_modules/index.js')).toBe(true);
    expect(ignore.ignores('test.log')).toBe(true);
    expect(ignore.ignores('important.log')).toBe(false);
    expect(ignore.ignores('dist')).toBe(true);
  });

  it('falls back to defaults on parse error', () => {
    const malformed = 'node_modules/\n[[[invalid';
    const ignore = parseWikiIgnore(malformed);

    // Should still ignore common patterns
    expect(ignore.ignores('.git/config')).toBe(true);
    expect(ignore.ignores('node_modules/pkg')).toBe(true);
    expect(ignore.ignores('.DS_Store')).toBe(true);
  });

  it('uses defaults when content empty', () => {
    const ignore = parseWikiIgnore('');

    expect(ignore.ignores('.git/HEAD')).toBe(true);
    expect(ignore.ignores('node_modules/x')).toBe(true);
    expect(ignore.ignores('dist/bundle.js')).toBe(true);
  });

  it('default patterns include common artifacts', () => {
    expect(DEFAULT_IGNORE_PATTERNS).toContain('.git');
    expect(DEFAULT_IGNORE_PATTERNS).toContain('node_modules');
    expect(DEFAULT_IGNORE_PATTERNS).toContain('dist');
    expect(DEFAULT_IGNORE_PATTERNS).toContain('.DS_Store');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test tests/unit/ignore-parser.test.ts`
Expected: FAIL with "Cannot find module '../../src/watcher/ignore-parser'"

- [ ] **Step 3: Implement ignore parser**

```typescript
// src/watcher/ignore-parser.ts
import ignore from 'ignore';

export const DEFAULT_IGNORE_PATTERNS = [
  '.git',
  'node_modules',
  'dist',
  '.DS_Store',
  '*.pdf',
  '*.zip',
  '*.tar.gz',
  '*.jpg',
  '*.jpeg',
  '*.png',
  '*.gif',
  '*.mp4',
  '*.mov'
];

export function parseWikiIgnore(content: string): ReturnType<typeof ignore> {
  try {
    const ig = ignore();
    if (content.trim()) {
      ig.add(content);
    } else {
      ig.add(DEFAULT_IGNORE_PATTERNS);
    }
    return ig;
  } catch (err) {
    console.warn('.wikiignore parse error, using defaults:', err);
    return ignore().add(DEFAULT_IGNORE_PATTERNS);
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test tests/unit/ignore-parser.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/watcher/ignore-parser.ts tests/unit/ignore-parser.test.ts
git commit -m "feat: .wikiignore parser with fallback to defaults"
```

---

## Task 7: Watcher setup (chokidar)

**Files:**
- Create: `src/watcher/index.ts`
- Modify: `src/types.ts` (add IngestQueue type)

- [ ] **Step 1: Add IngestQueue type**

```typescript
// src/types.ts (append)

export interface IngestQueue {
  add(job: IngestJob): void;
}
```

- [ ] **Step 2: Implement watcher**

```typescript
// src/watcher/index.ts
import chokidar from 'chokidar';
import { readFileSync, existsSync } from 'fs';
import { join, relative } from 'path';
import { parseWikiIgnore } from './ignore-parser.js';
import type { IngestQueue, IngestJob } from '../types.js';

export function startWatchers(
  wikiPath: string,
  immutablePath: string,
  queue: IngestQueue
): () => void {
  const wikiIgnorePath = join(wikiPath, '.wikiignore');
  let wikiIgnore = parseWikiIgnore('');

  if (existsSync(wikiIgnorePath)) {
    try {
      const content = readFileSync(wikiIgnorePath, 'utf-8');
      wikiIgnore = parseWikiIgnore(content);
    } catch (err) {
      console.warn('Failed to read .wikiignore, using defaults');
    }
  } else {
    wikiIgnore = parseWikiIgnore('');
  }

  const debounceMap = new Map<string, NodeJS.Timeout>();

  function enqueueDebounced(job: IngestJob) {
    const key = job.relpath;
    const existing = debounceMap.get(key);
    if (existing) {
      clearTimeout(existing);
    }

    const timer = setTimeout(() => {
      debounceMap.delete(key);
      queue.add(job);
    }, 2000);

    debounceMap.set(key, timer);
  }

  const wikiWatcher = chokidar.watch(wikiPath, {
    ignored: (path: string) => {
      const rel = relative(wikiPath, path);
      return wikiIgnore.ignores(rel);
    },
    persistent: true,
    ignoreInitial: true
  });

  wikiWatcher.on('add', path => {
    const relpath = relative(wikiPath, path);
    enqueueDebounced({ op: 'add', sourceDir: 'wiki', relpath });
  });

  wikiWatcher.on('change', path => {
    const relpath = relative(wikiPath, path);
    enqueueDebounced({ op: 'change', sourceDir: 'wiki', relpath });
  });

  wikiWatcher.on('unlink', path => {
    const relpath = relative(wikiPath, path);
    enqueueDebounced({ op: 'unlink', sourceDir: 'wiki', relpath });
  });

  const immutableWatcher = chokidar.watch(immutablePath, {
    depth: 0,
    persistent: true,
    ignoreInitial: true
  });

  immutableWatcher.on('add', path => {
    const relpath = relative(immutablePath, path);
    enqueueDebounced({ op: 'add', sourceDir: 'immutable', relpath });
  });

  immutableWatcher.on('change', path => {
    const relpath = relative(immutablePath, path);
    enqueueDebounced({ op: 'change', sourceDir: 'immutable', relpath });
  });

  immutableWatcher.on('unlink', path => {
    const relpath = relative(immutablePath, path);
    enqueueDebounced({ op: 'unlink', sourceDir: 'immutable', relpath });
  });

  return () => {
    wikiWatcher.close();
    immutableWatcher.close();
  };
}
```

- [ ] **Step 3: Commit**

```bash
git add src/watcher/index.ts src/types.ts
git commit -m "feat: chokidar watchers with debounce and .wikiignore"
```

---

## Task 8: Reconciler queue and worker

**Files:**
- Create: `src/reconciler/index.ts`
- Create: `tests/fixtures/stub-llm.ts`

- [ ] **Step 1: Create stub LLM for tests**

```typescript
// tests/fixtures/stub-llm.ts
export function createStubLLMProvider() {
  return {
    async generateText(opts: { systemPrompt: string; userPrompt: string }) {
      return JSON.stringify([
        {
          claim: 'stub fact',
          confidence: 0.9,
          tags: ['test'],
          relatedClaims: []
        }
      ]);
    },
    async embed(text: string) {
      return new Array(128).fill(0.1);
    }
  };
}
```

- [ ] **Step 2: Implement reconciler**

```typescript
// src/reconciler/index.ts
import PQueue from 'p-queue';
import { readFileSync, statSync } from 'fs';
import { join } from 'path';
import type { WikiMemory } from '@equationalapplications/core-llm-wiki';
import type { IngestJob, IngestQueue } from '../types.js';
import { computeSourceRef, computeSourceHash } from './hash.js';

const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10 MB

export class Reconciler implements IngestQueue {
  private queue: PQueue;

  constructor(
    private wikiMemory: WikiMemory,
    private entityId: string,
    private wikiPath: string,
    private immutablePath: string,
    private adapter: any
  ) {
    this.queue = new PQueue({ concurrency: 1 });
  }

  add(job: IngestJob): void {
    this.queue.add(() => this.processJob(job));
  }

  async drain(): Promise<void> {
    await this.queue.onIdle();
  }

  get size(): number {
    return this.queue.size + this.queue.pending;
  }

  private async resolveSourceRef(relpath: string): Promise<string> {
    const existing = await this.adapter.getFirstAsync<{ source_ref: string }>(
      'SELECT source_ref FROM source_ref_map WHERE relpath = ?',
      [relpath]
    );
    if (existing) return existing.source_ref;

    const base = computeSourceRef(relpath);
    let candidate = base;
    let suffix = 0;
    while (true) {
      const collision = await this.adapter.getFirstAsync<{ relpath: string }>(
        'SELECT relpath FROM source_ref_map WHERE source_ref = ?',
        [candidate]
      );
      if (!collision) break;
      suffix += 1;
      candidate = `${base}-${suffix}`;
    }

    await this.adapter.runAsync(
      'INSERT INTO source_ref_map (relpath, source_ref) VALUES (?, ?)',
      [relpath, candidate]
    );
    return candidate;
  }

  private async processJob(job: IngestJob): Promise<void> {
    const sourceRef = await this.resolveSourceRef(job.relpath);

    if (job.op === 'unlink') {
      await this.softDeleteBySourceRef(sourceRef);
      return;
    }

    const basePath =
      job.sourceDir === 'wiki' ? this.wikiPath : this.immutablePath;
    const fullPath = join(basePath, job.relpath);

    let stat;
    try {
      stat = statSync(fullPath);
    } catch {
      return;
    }
    if (stat.size > MAX_FILE_SIZE) {
      await this.logIngestFailure(job.relpath, '', 'File too large (>10MB)');
      return;
    }

    let content: string;
    try {
      const buf = readFileSync(fullPath);
      // UTF-8 decode test: fail silently on binary
      const decoded = buf.toString('utf-8');
      if (decoded.includes('\uFFFD')) return;
      content = decoded;
    } catch {
      return;
    }

    const sourceHash = computeSourceHash(content);

    const existing = await this.adapter.getAllAsync(
      'SELECT source_hash FROM llm_wiki_entries WHERE source_ref = ? AND deleted_at IS NULL',
      [sourceRef]
    );

    if (existing.length > 0 && existing[0].source_hash === sourceHash) {
      return;
    }

    if (existing.length > 0 && existing[0].source_hash !== sourceHash) {
      await this.softDeleteBySourceRef(sourceRef);
    }

    const contentWithPath = `# path: ${job.relpath}\n\n${content}`;

    try {
      const opts: Record<string, unknown> = {
        source_ref: sourceRef,
        source_hash: sourceHash
      };
      if (job.sourceDir === 'immutable') {
        opts.source_type = 'immutable_document';
      }
      await this.wikiMemory.ingestDocument(this.entityId, contentWithPath, opts);
    } catch (err) {
      await this.logIngestFailure(
        job.relpath,
        sourceHash,
        err instanceof Error ? err.message : String(err)
      );
    }
  }

  private async softDeleteBySourceRef(sourceRef: string): Promise<void> {
    await this.adapter.runAsync(
      'UPDATE llm_wiki_entries SET deleted_at = ? WHERE source_ref = ? AND deleted_at IS NULL',
      [Date.now(), sourceRef]
    );
  }

  private async logIngestFailure(
    path: string,
    hash: string,
    error: string
  ): Promise<void> {
    await this.adapter.runAsync(
      'INSERT OR REPLACE INTO ingest_failures (path, hash, error, ts) VALUES (?, ?, ?, ?)',
      [path, hash, error, Date.now()]
    );
  }
}
```

- [ ] **Step 3: Commit**

```bash
git add src/reconciler/index.ts tests/fixtures/stub-llm.ts
git commit -m "feat: reconciler queue with hash-based dedup"
```

---

## Task 9: Boot rescan

**Files:**
- Create: `src/boot/rescan.ts`

- [ ] **Step 1: Implement boot rescan**

```typescript
// src/boot/rescan.ts
import { readdirSync, statSync, readFileSync } from 'fs';
import { join, relative } from 'path';
import { parseWikiIgnore } from '../watcher/ignore-parser.js';
import { computeSourceRef, computeSourceHash } from '../reconciler/hash.js';
import type { Reconciler } from '../reconciler/index.js';

interface RescanProgress {
  current: number;
  total: number;
  eta: number;
}

let rescanProgress: RescanProgress | null = null;

export function getRescanProgress(): RescanProgress | null {
  return rescanProgress;
}

export async function runBootRescan(
  wikiPath: string,
  immutablePath: string,
  adapter: any,
  reconciler: Reconciler
): Promise<void> {
  console.log('Boot rescan: ensuring tables exist');

  await adapter.execAsync(`
    CREATE TABLE IF NOT EXISTS outbox (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      op TEXT NOT NULL,
      table_name TEXT NOT NULL,
      row_id TEXT,
      payload_json TEXT NOT NULL,
      created_at INTEGER NOT NULL
    );
    CREATE INDEX IF NOT EXISTS outbox_created_idx ON outbox(created_at);

    CREATE TABLE IF NOT EXISTS source_ref_map (
      relpath TEXT PRIMARY KEY,
      source_ref TEXT NOT NULL
    );

    CREATE TABLE IF NOT EXISTS ingest_failures (
      path TEXT PRIMARY KEY,
      hash TEXT NOT NULL,
      error TEXT NOT NULL,
      ts INTEGER NOT NULL
    );
  `);

  console.log('Boot rescan: walking filesystem');

  const wikiIgnore = parseWikiIgnore('');
  const fsHashes = new Map<string, string>();

  function walk(dir: string, basePath: string, depth: number, maxDepth?: number) {
    if (maxDepth !== undefined && depth > maxDepth) return;

    const entries = readdirSync(dir);
    for (const entry of entries) {
      const fullPath = join(dir, entry);
      const relpath = relative(basePath, fullPath);

      if (basePath === wikiPath && wikiIgnore.ignores(relpath)) {
        continue;
      }

      const stat = statSync(fullPath);
      if (stat.isDirectory()) {
        walk(fullPath, basePath, depth + 1, maxDepth);
      } else if (stat.isFile()) {
        if (stat.size > 10 * 1024 * 1024) continue;
        try {
          const buf = readFileSync(fullPath);
          const content = buf.toString('utf-8');
          if (content.includes('\uFFFD')) continue;
          const hash = computeSourceHash(content);
          fsHashes.set(relpath, hash);
        } catch {
          // skip binary or unreadable
        }
      }
    }
  }

  walk(wikiPath, wikiPath, 0);
  walk(immutablePath, immutablePath, 0, 0);

  console.log(`Boot rescan: found ${fsHashes.size} files`);

  const dbRows = await adapter.getAllAsync<{ source_ref: string }>(
    'SELECT DISTINCT source_ref FROM llm_wiki_entries WHERE deleted_at IS NULL'
  );

  const dbRefs = new Set(dbRows.map(r => r.source_ref));

  const jobs: Array<{ op: 'add' | 'change' | 'unlink'; sourceDir: 'wiki' | 'immutable'; relpath: string }> = [];

  for (const [relpath, hash] of fsHashes) {
    const ref = computeSourceRef(relpath);
    const sourceDir = relpath.startsWith('immutable') || relpath.startsWith('immutable/') ? 'immutable' : 'wiki';

    if (!dbRefs.has(ref)) {
      jobs.push({ op: 'add', sourceDir, relpath });
    } else {
      const existingRows = await adapter.getAllAsync<{ source_hash: string }>(
        'SELECT source_hash FROM llm_wiki_entries WHERE source_ref = ? AND deleted_at IS NULL',
        [ref]
      );
      if (existingRows.length > 0 && existingRows[0].source_hash !== hash) {
        jobs.push({ op: 'change', sourceDir, relpath });
      }
      dbRefs.delete(ref);
    }
  }

  for (const ref of dbRefs) {
    const rows = await adapter.getAllAsync<{ relpath: string }>(
      'SELECT relpath FROM source_ref_map WHERE source_ref = ?',
      [ref]
    );
    if (rows.length > 0) {
      const relpath = rows[0].relpath;
      const sourceDir = relpath.startsWith('immutable') || relpath.startsWith('immutable/') ? 'immutable' : 'wiki';
      jobs.push({ op: 'unlink', sourceDir, relpath });
    }
  }

  console.log(`Boot rescan: ${jobs.length} reconciliation jobs`);

  rescanProgress = { current: 0, total: jobs.length, eta: 0 };

  const startTime = Date.now();
  for (let i = 0; i < jobs.length; i++) {
    reconciler.add(jobs[i]);
    rescanProgress.current = i + 1;
    if (i > 0) {
      const elapsed = Date.now() - startTime;
      const rate = elapsed / (i + 1);
      rescanProgress.eta = Math.round(rate * (jobs.length - i - 1));
    }
  }

  await reconciler.drain();

  rescanProgress = null;
  console.log('Boot rescan: complete');
}
```

- [ ] **Step 2: Commit**

```bash
git add src/boot/rescan.ts
git commit -m "feat: boot rescan with full hash diff"
```

---

## Task 10: BBS HTML templates

**Files:**
- Create: `src/http/templates/layout.ts`
- Create: `src/http/templates/index.ts`
- Create: `src/http/templates/fact.ts`
- Create: `src/http/templates/search.ts`
- Create: `src/http/templates/upload.ts`
- Create: `src/http/templates/status.ts`

- [ ] **Step 1: Create layout template with BBS CSS**

```typescript
// src/http/templates/layout.ts
export function layout(title: string, body: string): string {
  return `<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>${title} — LLM Wiki</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: #000;
      color: #ffb000;
      font-family: 'Courier New', monospace;
      font-size: 14px;
      line-height: 1.4;
      padding: 20px;
    }
    a { color: #ffb000; text-decoration: none; }
    a:hover { text-decoration: underline; }
    .box {
      border: 2px solid #ffb000;
      padding: 10px;
      margin-bottom: 20px;
    }
    .box-title {
      font-weight: bold;
      margin-bottom: 10px;
      text-align: center;
    }
    pre { white-space: pre-wrap; }
    input, textarea, button {
      background: #000;
      color: #ffb000;
      border: 1px solid #ffb000;
      font-family: 'Courier New', monospace;
      font-size: 14px;
      padding: 5px;
    }
    button { cursor: pointer; }
    button:hover { background: #ffb000; color: #000; }
  </style>
</head>
<body>
  <div class="box">
    <div class="box-title">╔═══════════════════════════╗</div>
    <div class="box-title">║   LLM WIKI DOCKER v1.0   ║</div>
    <div class="box-title">╚═══════════════════════════╝</div>
  </div>
  ${body}
  <div class="box">
    <a href="/">[Home]</a> | <a href="/search?q=">[Search]</a> | <a href="/upload">[Upload]</a> | <a href="/status">[Status]</a>
  </div>
</body>
</html>`;
}
```

- [ ] **Step 2: Create index template**

```typescript
// src/http/templates/index.ts
import { layout } from './layout.js';

export function indexTemplate(facts: any[], recentEvents: any[]): string {
  const factsHtml = facts
    .map(
      f =>
        `<div><a href="/facts/${f.id}">[${f.id}]</a> ${f.claim.slice(0, 80)}... (conf: ${f.confidence})</div>`
    )
    .join('\n');

  const eventsHtml = recentEvents
    .map(e => `<div>[${new Date(e.created_at).toISOString()}] ${e.type}</div>`)
    .join('\n');

  const body = `
<div class="box">
  <div class="box-title">Recent Events</div>
  ${eventsHtml || '<div>No events</div>'}
</div>

<div class="box">
  <div class="box-title">Facts</div>
  ${factsHtml || '<div>No facts</div>'}
</div>
`;

  return layout('Home', body);
}
```

- [ ] **Step 3: Create fact detail template**

```typescript
// src/http/templates/fact.ts
import { layout } from './layout.js';

export function factTemplate(fact: any): string {
  const body = `
<div class="box">
  <div class="box-title">Fact #${fact.id}</div>
  <div><strong>Claim:</strong> ${fact.claim}</div>
  <div><strong>Confidence:</strong> ${fact.confidence}</div>
  <div><strong>Source Ref:</strong> ${fact.source_ref}</div>
  <div><strong>Tags:</strong> ${fact.tags?.join(', ') || 'none'}</div>
  <div><strong>Created:</strong> ${new Date(fact.created_at).toISOString()}</div>
</div>

<div class="box">
  <div class="box-title">Body</div>
  <pre>${fact.body || '(empty)'}</pre>
</div>
`;

  return layout(`Fact #${fact.id}`, body);
}
```

- [ ] **Step 4: Create search results template**

```typescript
// src/http/templates/search.ts
import { layout } from './layout.js';

export function searchTemplate(query: string, results: any[]): string {
  const resultsHtml = results
    .map(
      r =>
        `<div><a href="/facts/${r.id}">[${r.id}]</a> ${r.claim} (score: ${r.score?.toFixed(2)})</div>`
    )
    .join('\n');

  const body = `
<div class="box">
  <form method="get" action="/search">
    <input type="text" name="q" value="${query}" placeholder="Search query" style="width: 80%;">
    <button type="submit">Search</button>
  </form>
</div>

<div class="box">
  <div class="box-title">Results (${results.length})</div>
  ${resultsHtml || '<div>No results</div>'}
</div>
`;

  return layout('Search', body);
}
```

- [ ] **Step 5: Create upload form template**

```typescript
// src/http/templates/upload.ts
import { layout } from './layout.js';

export function uploadTemplate(): string {
  const body = `
<div class="box">
  <div class="box-title">Upload Document</div>
  <form method="post" action="/upload" enctype="multipart/form-data">
    <div><input type="file" name="file" accept=".md,.txt,.markdown" required></div>
    <div style="margin-top: 10px;"><button type="submit">Upload</button></div>
  </form>
  <div style="margin-top: 10px;">Allowed: .md, .txt, .markdown</div>
</div>
`;

  return layout('Upload', body);
}
```

- [ ] **Step 6: Create status template**

```typescript
// src/http/templates/status.ts
import { layout } from './layout.js';

export function statusTemplate(status: any): string {
  const body = `
<div class="box">
  <div class="box-title">Boot Rescan</div>
  ${
    status.rescan
      ? `<div>Progress: ${status.rescan.current}/${status.rescan.total} (ETA: ${status.rescan.eta}ms)</div>`
      : '<div>Complete</div>'
  }
</div>

<div class="box">
  <div class="box-title">Queue Depth</div>
  <div>${status.queueDepth} pending</div>
</div>

<div class="box">
  <div class="box-title">Outbox Depth</div>
  <div>${status.outboxDepth} rows</div>
</div>

<div class="box">
  <div class="box-title">LLM Reachability</div>
  <div>${status.llmReachable ? 'OK' : 'FAIL'}</div>
</div>

<div class="box">
  <div class="box-title">Ingest Failures</div>
  <div>${status.ingestFailures} failures</div>
</div>

<div class="box">
  <div class="box-title">Maintenance</div>
  <form method="post" action="/maintenance/librarian" style="display:inline;">
    <button type="submit">Run Librarian</button>
  </form>
  <form method="post" action="/maintenance/heal" style="display:inline;">
    <button type="submit">Run Heal</button>
  </form>
  <form method="post" action="/maintenance/prune" style="display:inline;">
    <button type="submit">Run Prune</button>
  </form>
  <form method="post" action="/maintenance/reembed" style="display:inline;">
    <button type="submit">Run Reembed</button>
  </form>
</div>
`;

  return layout('Status', body);
}
```

- [ ] **Step 7: Commit**

```bash
git add src/http/templates/
git commit -m "feat: BBS-style HTML templates"
```

---

## Task 11: Fastify routes

**Files:**
- Create: `src/http/routes/index.ts`
- Create: `src/http/routes/facts.ts`
- Create: `src/http/routes/search.ts`
- Create: `src/http/routes/upload.ts`
- Create: `src/http/routes/status.ts`
- Create: `src/http/routes/maintenance.ts`

- [ ] **Step 1: Implement index route**

```typescript
// src/http/routes/index.ts
import type { FastifyInstance } from 'fastify';
import type { WikiMemory } from '@equationalapplications/core-llm-wiki';
import { indexTemplate } from '../templates/index.js';

export async function indexRoute(
  app: FastifyInstance,
  wikiMemory: WikiMemory,
  entityId: string,
  adapter: any
) {
  app.get('/', async (req, reply) => {
    const facts = await adapter.getAllAsync(
      'SELECT id, claim, confidence FROM llm_wiki_entries WHERE deleted_at IS NULL ORDER BY created_at DESC LIMIT 50'
    );

    const events = await adapter.getAllAsync(
      'SELECT type, created_at FROM llm_wiki_events ORDER BY created_at DESC LIMIT 10'
    );

    reply.type('text/html').send(indexTemplate(facts, events));
  });
}
```

- [ ] **Step 2: Implement facts route**

```typescript
// src/http/routes/facts.ts
import type { FastifyInstance } from 'fastify';
import { factTemplate } from '../templates/fact.js';

export async function factsRoute(app: FastifyInstance, adapter: any) {
  app.get('/facts/:id', async (req, reply) => {
    const { id } = req.params as { id: string };

    const fact = await adapter.getFirstAsync(
      'SELECT * FROM llm_wiki_entries WHERE id = ?',
      [id]
    );

    if (!fact) {
      return reply.status(404).send('Fact not found');
    }

    reply.type('text/html').send(factTemplate(fact));
  });
}
```

- [ ] **Step 3: Implement search route**

```typescript
// src/http/routes/search.ts
import type { FastifyInstance } from 'fastify';
import type { WikiMemory } from '@equationalapplications/core-llm-wiki';
import { searchTemplate } from '../templates/search.js';

export async function searchRoute(
  app: FastifyInstance,
  wikiMemory: WikiMemory,
  entityId: string
) {
  app.get('/search', async (req, reply) => {
    const { q } = req.query as { q?: string };

    if (!q) {
      return reply.type('text/html').send(searchTemplate('', []));
    }

    const results = await wikiMemory.read(entityId, q);

    reply.type('text/html').send(searchTemplate(q, results));
  });
}
```

- [ ] **Step 4: Implement upload route**

```typescript
// src/http/routes/upload.ts
import type { FastifyInstance } from 'fastify';
import { writeFileSync } from 'fs';
import { join } from 'path';
import { uploadTemplate } from '../templates/upload.js';

export async function uploadRoute(
  app: FastifyInstance,
  immutablePath: string
) {
  app.get('/upload', async (req, reply) => {
    reply.type('text/html').send(uploadTemplate());
  });

  app.post('/upload', async (req, reply) => {
    const data = await req.file();

    if (!data) {
      return reply.status(400).send('No file uploaded');
    }

    const filename = data.filename
      .toLowerCase()
      .replace(/\s+/g, '_')
      .replace(/[^a-z0-9._-]/g, '');

    const ext = filename.split('.').pop();
    if (!['md', 'txt', 'markdown'].includes(ext || '')) {
      return reply.status(400).send('Invalid file extension');
    }

    if (filename.startsWith('.')) {
      return reply.status(400).send('Invalid filename');
    }

    const buffer = await data.toBuffer();
    const targetPath = join(immutablePath, filename);

    writeFileSync(targetPath, buffer);

    reply.redirect('/');
  });
}
```

- [ ] **Step 5: Implement status route**

```typescript
// src/http/routes/status.ts
import type { FastifyInstance } from 'fastify';
import { getRescanProgress } from '../../boot/rescan.js';
import type { Reconciler } from '../../reconciler/index.js';
import { statusTemplate } from '../templates/status.js';

export async function statusRoute(
  app: FastifyInstance,
  adapter: any,
  reconciler: Reconciler,
  llmReachableFn: () => Promise<boolean>
) {
  app.get('/status', async (req, reply) => {
    const outboxCount = await adapter.getFirstAsync<{ cnt: number }>(
      'SELECT COUNT(*) as cnt FROM outbox'
    );

    const failuresCount = await adapter.getFirstAsync<{ cnt: number }>(
      'SELECT COUNT(*) as cnt FROM ingest_failures'
    );

    const llmReachable = await llmReachableFn();

    const status = {
      rescan: getRescanProgress(),
      queueDepth: reconciler.size,
      outboxDepth: outboxCount?.cnt || 0,
      llmReachable,
      ingestFailures: failuresCount?.cnt || 0
    };

    reply.type('text/html').send(statusTemplate(status));
  });
}
```

- [ ] **Step 6: Implement maintenance route**

```typescript
// src/http/routes/maintenance.ts
import type { FastifyInstance } from 'fastify';
import type { WikiMemory } from '@equationalapplications/core-llm-wiki';

export async function maintenanceRoute(
  app: FastifyInstance,
  wikiMemory: WikiMemory,
  entityId: string
) {
  app.post('/maintenance/librarian', async (req, reply) => {
    try {
      await wikiMemory.runLibrarian(entityId);
      reply.redirect('/status');
    } catch (err: any) {
      if (err.name === 'WikiBusyError') {
        return reply.status(409).send('Wiki busy');
      }
      throw err;
    }
  });

  app.post('/maintenance/heal', async (req, reply) => {
    try {
      await wikiMemory.runHeal(entityId);
      reply.redirect('/status');
    } catch (err: any) {
      if (err.name === 'WikiBusyError') {
        return reply.status(409).send('Wiki busy');
      }
      throw err;
    }
  });

  app.post('/maintenance/prune', async (req, reply) => {
    try {
      await wikiMemory.runPrune(entityId);
      reply.redirect('/status');
    } catch (err: any) {
      if (err.name === 'WikiBusyError') {
        return reply.status(409).send('Wiki busy');
      }
      throw err;
    }
  });

  app.post('/maintenance/reembed', async (req, reply) => {
    try {
      await wikiMemory.runReembed(entityId);
      reply.redirect('/status');
    } catch (err: any) {
      if (err.name === 'WikiBusyError') {
        return reply.status(409).send('Wiki busy');
      }
      throw err;
    }
  });
}
```

- [ ] **Step 7: Commit**

```bash
git add src/http/routes/
git commit -m "feat: Fastify routes for HTML UI"
```

---

## Task 12: Fastify server setup

**Files:**
- Create: `src/http/server.ts`

- [ ] **Step 1: Implement Fastify server**

```typescript
// src/http/server.ts
import Fastify from 'fastify';
import multipart from '@fastify/multipart';
import type { WikiMemory } from '@equationalapplications/core-llm-wiki';
import type { Reconciler } from '../reconciler/index.js';
import { indexRoute } from './routes/index.js';
import { factsRoute } from './routes/facts.js';
import { searchRoute } from './routes/search.js';
import { uploadRoute } from './routes/upload.js';
import { statusRoute } from './routes/status.js';
import { maintenanceRoute } from './routes/maintenance.js';

export async function createHTTPServer(
  port: number,
  wikiMemory: WikiMemory,
  entityId: string,
  adapter: any,
  reconciler: Reconciler,
  immutablePath: string,
  llmReachableFn: () => Promise<boolean>
) {
  const app = Fastify({ logger: false });

  await app.register(multipart);

  await indexRoute(app, wikiMemory, entityId, adapter);
  await factsRoute(app, adapter);
  await searchRoute(app, wikiMemory, entityId);
  await uploadRoute(app, immutablePath);
  await statusRoute(app, adapter, reconciler, llmReachableFn);
  await maintenanceRoute(app, wikiMemory, entityId);

  await app.listen({ port, host: '127.0.0.1' });

  console.log(`HTTP server listening on http://127.0.0.1:${port}`);

  return app;
}
```

- [ ] **Step 2: Commit**

```bash
git add src/http/server.ts
git commit -m "feat: Fastify server with all routes"
```

---

## Task 13: MCP server

**Files:**
- Create: `src/mcp/server.ts`

- [ ] **Step 1: Implement MCP server**

```typescript
// src/mcp/server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { SSEServerTransport } from '@modelcontextprotocol/sdk/server/sse.js';
import {
  CallToolRequestSchema,
  ListResourcesRequestSchema,
  ListToolsRequestSchema,
  ReadResourceRequestSchema
} from '@modelcontextprotocol/sdk/types.js';
import type { WikiMemory } from '@equationalapplications/core-llm-wiki';
import Fastify from 'fastify';
import { writeFileSync } from 'fs';
import { join } from 'path';

export async function createMCPServer(
  port: number,
  wikiMemory: WikiMemory,
  entityId: string,
  adapter: any,
  immutablePath: string
) {
  const server = new Server(
    {
      name: 'llm-wiki',
      version: '1.0.0'
    },
    {
      capabilities: {
        tools: {},
        resources: {}
      }
    }
  );

  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: [
      {
        name: 'wiki_read',
        description: 'Semantic + keyword search',
        inputSchema: {
          type: 'object',
          properties: {
            query: { type: 'string' }
          },
          required: ['query']
        }
      },
      {
        name: 'wiki_ingest_text',
        description: 'Write synthetic document to immutable/',
        inputSchema: {
          type: 'object',
          properties: {
            title: { type: 'string' },
            body: { type: 'string' }
          },
          required: ['title', 'body']
        }
      },
      {
        name: 'wiki_list_facts',
        description: 'List all facts',
        inputSchema: {
          type: 'object',
          properties: {
            limit: { type: 'number' },
            offset: { type: 'number' }
          }
        }
      },
      {
        name: 'wiki_list_tasks',
        description: 'List tasks',
        inputSchema: {
          type: 'object',
          properties: {
            status: { type: 'string' }
          }
        }
      },
      {
        name: 'wiki_create_task',
        description: 'Create a new task',
        inputSchema: {
          type: 'object',
          properties: {
            description: { type: 'string' },
            priority: { type: 'number' }
          },
          required: ['description']
        }
      },
      {
        name: 'wiki_complete_task',
        description: 'Mark task as complete',
        inputSchema: {
          type: 'object',
          properties: {
            id: { type: 'number' }
          },
          required: ['id']
        }
      },
      {
        name: 'wiki_run_librarian',
        description: 'Run librarian maintenance',
        inputSchema: { type: 'object', properties: {} }
      },
      {
        name: 'wiki_run_heal',
        description: 'Run heal maintenance',
        inputSchema: { type: 'object', properties: {} }
      }
    ]
  }));

  server.setRequestHandler(CallToolRequestSchema, async request => {
    const { name, arguments: args } = request.params;

    switch (name) {
      case 'wiki_read': {
        const results = await wikiMemory.read(entityId, args.query as string);
        return { content: [{ type: 'text', text: JSON.stringify(results, null, 2) }] };
      }

      case 'wiki_ingest_text': {
        const filename = `${args.title}.md`.replace(/[^a-z0-9._-]/gi, '_');
        const targetPath = join(immutablePath, filename);
        writeFileSync(targetPath, args.body as string);
        return { content: [{ type: 'text', text: `Wrote ${filename}` }] };
      }

      case 'wiki_list_facts': {
        const limit = (args.limit as number) || 50;
        const offset = (args.offset as number) || 0;
        const facts = await adapter.getAllAsync(
          'SELECT * FROM llm_wiki_entries WHERE deleted_at IS NULL ORDER BY created_at DESC LIMIT ? OFFSET ?',
          [limit, offset]
        );
        return { content: [{ type: 'text', text: JSON.stringify(facts, null, 2) }] };
      }

      case 'wiki_list_tasks': {
        const status = args.status || 'pending';
        const tasks = await adapter.getAllAsync(
          'SELECT * FROM llm_wiki_tasks WHERE status = ? ORDER BY created_at DESC',
          [status]
        );
        return { content: [{ type: 'text', text: JSON.stringify(tasks, null, 2) }] };
      }

      case 'wiki_create_task': {
        await wikiMemory.createTask(entityId, {
          description: args.description as string,
          priority: (args.priority as number) || 1
        });
        return { content: [{ type: 'text', text: 'Task created' }] };
      }

      case 'wiki_complete_task': {
        await wikiMemory.completeTask(entityId, args.id as number);
        return { content: [{ type: 'text', text: 'Task completed' }] };
      }

      case 'wiki_run_librarian': {
        await wikiMemory.runLibrarian(entityId);
        return { content: [{ type: 'text', text: 'Librarian complete' }] };
      }

      case 'wiki_run_heal': {
        await wikiMemory.runHeal(entityId);
        return { content: [{ type: 'text', text: 'Heal complete' }] };
      }

      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  });

  server.setRequestHandler(ListResourcesRequestSchema, async () => {
    const facts = await adapter.getAllAsync<{ id: number }>(
      'SELECT id FROM llm_wiki_entries WHERE deleted_at IS NULL'
    );
    const tasks = await adapter.getAllAsync<{ id: number }>(
      'SELECT id FROM llm_wiki_tasks'
    );
    return {
      resources: [
        ...facts.map(f => ({
          uri: `wiki://fact/${f.id}`,
          name: `Fact ${f.id}`,
          mimeType: 'application/json'
        })),
        ...tasks.map(t => ({
          uri: `wiki://task/${t.id}`,
          name: `Task ${t.id}`,
          mimeType: 'application/json'
        }))
      ]
    };
  });

  server.setRequestHandler(ReadResourceRequestSchema, async request => {
    const { uri } = request.params;

    const factMatch = uri.match(/^wiki:\/\/fact\/(\d+)$/);
    if (factMatch) {
      const row = await adapter.getFirstAsync(
        'SELECT * FROM llm_wiki_entries WHERE id = ?',
        [factMatch[1]]
      );
      return {
        contents: [{ uri, mimeType: 'application/json', text: JSON.stringify(row, null, 2) }]
      };
    }

    const taskMatch = uri.match(/^wiki:\/\/task\/(\d+)$/);
    if (taskMatch) {
      const row = await adapter.getFirstAsync(
        'SELECT * FROM llm_wiki_tasks WHERE id = ?',
        [taskMatch[1]]
      );
      return {
        contents: [{ uri, mimeType: 'application/json', text: JSON.stringify(row, null, 2) }]
      };
    }

    throw new Error(`Unknown resource: ${uri}`);
  });

  // HTTP+SSE transport on 127.0.0.1:MCP_PORT
  const httpApp = Fastify({ logger: false });
  let transport: SSEServerTransport | null = null;

  httpApp.get('/sse', async (req, reply) => {
    transport = new SSEServerTransport('/messages', reply.raw);
    await server.connect(transport);
  });

  httpApp.post('/messages', async (req, reply) => {
    if (!transport) {
      return reply.status(400).send('No active SSE session');
    }
    await transport.handlePostMessage(req.raw, reply.raw, req.body);
  });

  await httpApp.listen({ port, host: '127.0.0.1' });

  console.log(`MCP server listening on http://127.0.0.1:${port}/sse`);

  return server;
}
```

- [ ] **Step 2: Commit**

```bash
git add src/mcp/server.ts
git commit -m "feat: MCP server with tools and resources"
```

---

## Task 14: Main entrypoint

**Files:**
- Create: `src/index.ts`

- [ ] **Step 1: Implement main entrypoint**

```typescript
// src/index.ts
import { WikiMemory } from '@equationalapplications/core-llm-wiki';
import Database from 'better-sqlite3';
import { loadConfig } from './config.js';
import { OutboxSQLiteAdapter } from './adapter/outbox.js';
import { LLMClient } from './llm/client.js';
import { Reconciler } from './reconciler/index.js';
import { runBootRescan } from './boot/rescan.js';
import { startWatchers } from './watcher/index.js';
import { createHTTPServer } from './http/server.js';
import { createMCPServer } from './mcp/server.js';
import { mkdirSync, existsSync } from 'fs';

async function main() {
  const config = loadConfig();

  console.log('LLM Wiki Docker — starting');

  [config.paths.db, config.paths.immutable, config.paths.wiki].forEach(dir => {
    const parent = dir.split('/').slice(0, -1).join('/');
    if (!existsSync(parent)) {
      mkdirSync(parent, { recursive: true });
    }
  });

  const db = new Database(config.paths.db);
  const adapter = new OutboxSQLiteAdapter(db);

  const llmClient = new LLMClient(config.llm);

  const llmProvider = {
    async generateText(opts: { systemPrompt: string; userPrompt: string }) {
      return llmClient.generateText(opts);
    },
    async embed(text: string) {
      return llmClient.embed(text);
    }
  };

  const wikiMemory = new WikiMemory(adapter, llmProvider);

  await wikiMemory.migrate();

  const reconciler = new Reconciler(
    wikiMemory,
    config.entityId,
    config.paths.wiki,
    config.paths.immutable,
    adapter
  );

  await runBootRescan(
    config.paths.wiki,
    config.paths.immutable,
    adapter,
    reconciler
  );

  const stopWatchers = startWatchers(
    config.paths.wiki,
    config.paths.immutable,
    reconciler
  );

  const llmReachableFn = async () => {
    try {
      await llmClient.generateText({ systemPrompt: 'ping', userPrompt: 'pong' });
      return true;
    } catch {
      return false;
    }
  };

  const httpServer = await createHTTPServer(
    config.port,
    wikiMemory,
    config.entityId,
    adapter,
    reconciler,
    config.paths.immutable,
    llmReachableFn
  );

  const mcpServer = await createMCPServer(
    config.mcpPort,
    wikiMemory,
    config.entityId,
    adapter,
    config.paths.immutable
  );

  console.log('LLM Wiki Docker — ready');

  process.on('SIGINT', async () => {
    console.log('Shutting down...');
    stopWatchers();
    await httpServer.close();
    db.close();
    process.exit(0);
  });
}

main().catch(err => {
  console.error('Fatal error:', err);
  process.exit(1);
});
```

- [ ] **Step 2: Commit**

```bash
git add src/index.ts
git commit -m "feat: main entrypoint with full bootstrap"
```

---

## Task 15: Docker setup

**Files:**
- Create: `Dockerfile`
- Create: `docker-compose.yml`

- [ ] **Step 1: Create Dockerfile**

```dockerfile
# Dockerfile — multi-stage. better-sqlite3 needs build toolchain on alpine.
FROM node:24-alpine AS build
WORKDIR /app
RUN apk add --no-cache python3 make g++ libc6-compat
COPY package*.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

FROM node:24-alpine AS runtime
WORKDIR /app
RUN apk add --no-cache libc6-compat
COPY package*.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
RUN npm prune --omit=dev
RUN mkdir -p /app/data /app/immutable /app/wiki
EXPOSE 8080 8081
CMD ["node", "dist/index.js"]
```

- [ ] **Step 2: Create docker-compose.yml**

```yaml
# docker-compose.yml — Docker Compose v5.1.3+
services:
  llm-wiki:
    build: .
    container_name: llm-wiki
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:8081:8081"
    volumes:
      - ./data:/app/data
      - ./immutable:/app/immutable
      - ./wiki:/app/wiki
    environment:
      LLM_BASE_URL: ${LLM_BASE_URL:-http://host.docker.internal:11434/v1}
      LLM_API_KEY: ${LLM_API_KEY:-}
      LLM_MODEL: ${LLM_MODEL:-llama3.2}
      EMBED_MODEL: ${EMBED_MODEL:-nomic-embed-text}
      PORT: ${PORT:-8080}
      MCP_PORT: ${MCP_PORT:-8081}
      WIKI_ENTITY_ID: ${WIKI_ENTITY_ID:-default}
    restart: unless-stopped
```

- [ ] **Step 3: Commit**

```bash
git add Dockerfile docker-compose.yml
git commit -m "feat: Docker setup with compose"
```

---

## Task 16: Integration tests

**Files:**
- Create: `tests/integration/ingest-flow.test.ts`
- Create: `tests/integration/boot-rescan.test.ts`

- [ ] **Step 1: Write ingest flow integration test**

```typescript
// tests/integration/ingest-flow.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { WikiMemory } from '@equationalapplications/core-llm-wiki';
import Database from 'better-sqlite3';
import { OutboxSQLiteAdapter } from '../../src/adapter/outbox';
import { Reconciler } from '../../src/reconciler/index';
import { createStubLLMProvider } from '../fixtures/stub-llm';
import { mkdtempSync, rmSync, writeFileSync, mkdirSync } from 'fs';
import { tmpdir } from 'os';
import { join } from 'path';

describe('Ingest flow integration', () => {
  let tmpDir: string;
  let wikiPath: string;
  let immutablePath: string;
  let db: Database.Database;
  let adapter: OutboxSQLiteAdapter;
  let wikiMemory: WikiMemory;
  let reconciler: Reconciler;

  beforeEach(async () => {
    tmpDir = mkdtempSync(join(tmpdir(), 'ingest-test-'));
    wikiPath = join(tmpDir, 'wiki');
    immutablePath = join(tmpDir, 'immutable');

    mkdirSync(wikiPath);
    mkdirSync(immutablePath);

    db = new Database(join(tmpDir, 'wiki.db'));
    adapter = new OutboxSQLiteAdapter(db);

    wikiMemory = new WikiMemory(adapter, createStubLLMProvider());
    await wikiMemory.migrate();

    db.exec(`
      CREATE TABLE IF NOT EXISTS outbox (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        op TEXT NOT NULL,
        table_name TEXT NOT NULL,
        row_id TEXT,
        payload_json TEXT NOT NULL,
        created_at INTEGER NOT NULL
      );
      CREATE TABLE IF NOT EXISTS ingest_failures (
        path TEXT PRIMARY KEY,
        hash TEXT NOT NULL,
        error TEXT NOT NULL,
        ts INTEGER NOT NULL
      );
    `);

    reconciler = new Reconciler(
      wikiMemory,
      'default',
      wikiPath,
      immutablePath,
      adapter
    );
  });

  afterEach(() => {
    db.close();
    rmSync(tmpDir, { recursive: true, force: true });
  });

  it('ingests new file and creates facts', async () => {
    writeFileSync(join(wikiPath, 'test.md'), 'Test content');

    reconciler.add({ op: 'add', sourceDir: 'wiki', relpath: 'test.md' });
    await reconciler.drain();

    const facts = await adapter.getAllAsync(
      'SELECT * FROM llm_wiki_entries WHERE deleted_at IS NULL'
    );

    expect(facts.length).toBeGreaterThan(0);
  });

  it('updates facts when file changes', async () => {
    writeFileSync(join(wikiPath, 'test.md'), 'Original content');

    reconciler.add({ op: 'add', sourceDir: 'wiki', relpath: 'test.md' });
    await reconciler.drain();

    const originalFacts = await adapter.getAllAsync(
      'SELECT * FROM llm_wiki_entries WHERE deleted_at IS NULL'
    );

    writeFileSync(join(wikiPath, 'test.md'), 'Updated content');

    reconciler.add({ op: 'change', sourceDir: 'wiki', relpath: 'test.md' });
    await reconciler.drain();

    const deletedCount = await adapter.getFirstAsync<{ cnt: number }>(
      'SELECT COUNT(*) as cnt FROM llm_wiki_entries WHERE deleted_at IS NOT NULL'
    );

    expect(deletedCount?.cnt).toBeGreaterThan(0);
  });

  it('soft-deletes facts when file removed', async () => {
    writeFileSync(join(wikiPath, 'test.md'), 'Content');

    reconciler.add({ op: 'add', sourceDir: 'wiki', relpath: 'test.md' });
    await reconciler.drain();

    reconciler.add({ op: 'unlink', sourceDir: 'wiki', relpath: 'test.md' });
    await reconciler.drain();

    const activeFacts = await adapter.getAllAsync(
      'SELECT * FROM llm_wiki_entries WHERE deleted_at IS NULL'
    );

    expect(activeFacts.length).toBe(0);
  });

  it('mirrors wiki writes to outbox', async () => {
    writeFileSync(join(wikiPath, 'test.md'), 'Content');

    reconciler.add({ op: 'add', sourceDir: 'wiki', relpath: 'test.md' });
    await reconciler.drain();

    const outboxRows = await adapter.getAllAsync('SELECT * FROM outbox');

    expect(outboxRows.length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 2: Write boot rescan integration test**

```typescript
// tests/integration/boot-rescan.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { WikiMemory } from '@equationalapplications/core-llm-wiki';
import Database from 'better-sqlite3';
import { OutboxSQLiteAdapter } from '../../src/adapter/outbox';
import { Reconciler } from '../../src/reconciler/index';
import { runBootRescan } from '../../src/boot/rescan';
import { createStubLLMProvider } from '../fixtures/stub-llm';
import { mkdtempSync, rmSync, writeFileSync, mkdirSync } from 'fs';
import { tmpdir } from 'os';
import { join } from 'path';

describe('Boot rescan integration', () => {
  let tmpDir: string;
  let wikiPath: string;
  let immutablePath: string;
  let db: Database.Database;
  let adapter: OutboxSQLiteAdapter;
  let wikiMemory: WikiMemory;
  let reconciler: Reconciler;

  beforeEach(async () => {
    tmpDir = mkdtempSync(join(tmpdir(), 'rescan-test-'));
    wikiPath = join(tmpDir, 'wiki');
    immutablePath = join(tmpDir, 'immutable');

    mkdirSync(wikiPath);
    mkdirSync(immutablePath);

    db = new Database(join(tmpDir, 'wiki.db'));
    adapter = new OutboxSQLiteAdapter(db);

    wikiMemory = new WikiMemory(adapter, createStubLLMProvider());
    await wikiMemory.migrate();

    reconciler = new Reconciler(
      wikiMemory,
      'default',
      wikiPath,
      immutablePath,
      adapter
    );
  });

  afterEach(() => {
    db.close();
    rmSync(tmpDir, { recursive: true, force: true });
  });

  it('creates tables on first run', async () => {
    await runBootRescan(wikiPath, immutablePath, adapter, reconciler);

    const tables = db
      .prepare(
        "SELECT name FROM sqlite_master WHERE type='table' AND name IN ('outbox', 'source_ref_map', 'ingest_failures')"
      )
      .all();

    expect(tables.length).toBe(3);
  });

  it('reconciles files present in FS but not in DB', async () => {
    writeFileSync(join(wikiPath, 'new.md'), 'New file');

    await runBootRescan(wikiPath, immutablePath, adapter, reconciler);

    const facts = await adapter.getAllAsync(
      'SELECT * FROM llm_wiki_entries WHERE deleted_at IS NULL'
    );

    expect(facts.length).toBeGreaterThan(0);
  });

  it('reconciles files with changed hashes', async () => {
    writeFileSync(join(wikiPath, 'test.md'), 'Original');

    reconciler.add({ op: 'add', sourceDir: 'wiki', relpath: 'test.md' });
    await reconciler.drain();

    writeFileSync(join(wikiPath, 'test.md'), 'Updated');

    await runBootRescan(wikiPath, immutablePath, adapter, reconciler);

    const deletedCount = await adapter.getFirstAsync<{ cnt: number }>(
      'SELECT COUNT(*) as cnt FROM llm_wiki_entries WHERE deleted_at IS NOT NULL'
    );

    expect(deletedCount?.cnt).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 3: Run integration tests**

Run: `npm test tests/integration/`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add tests/integration/
git commit -m "test: integration tests for ingest and boot rescan"
```

---

## Task 17: README documentation

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README**

```markdown
# LLM Wiki Docker

Privacy-first single-user wiki that extracts facts from markdown/text files and code repositories using a local LLM. All inference runs against a user-supplied OpenAI-compatible endpoint (e.g., Ollama).

## Features

- File watching with automatic fact extraction
- Semantic + keyword search
- 1980s BBS-style HTML UI
- MCP server for agent integration
- SQLite storage with outbox pattern (v2 sync ready)
- Docker Desktop native (macOS)

## Prerequisites

- Docker Desktop for Mac 4.72.0+ (bundles Docker Compose v5.1.3)
- Ollama running locally (or another OpenAI-compatible LLM endpoint)

## Quick Start

1. **Start Ollama with required models:**

```bash
ollama run llama3.2
ollama pull nomic-embed-text
```

2. **Clone and configure:**

```bash
git clone <repo-url>
cd llm-wiki-docker
cp .env.example .env
# Edit .env if needed (defaults point to Ollama on host)
```

3. **Start the wiki:**

```bash
docker compose up
```

4. **Open browser:**

http://127.0.0.1:8080

5. **Add content:**

- Drop `.md` or `.txt` files into `./wiki/` directory
- Upload via web UI to `./immutable/`

## Configuration

Environment variables (set in `.env`):

| Variable | Default | Description |
|---|---|---|
| `LLM_BASE_URL` | `http://host.docker.internal:11434/v1` | LLM endpoint |
| `LLM_API_KEY` | (unset) | Optional bearer token |
| `LLM_MODEL` | `llama3.2` | Chat model |
| `EMBED_MODEL` | `nomic-embed-text` | Embeddings model |
| `PORT` | `8080` | HTML UI port |
| `MCP_PORT` | `8081` | MCP server port |

## Directory Structure

- `./wiki/` — live watched repository (supports `.wikiignore`)
- `./immutable/` — uploaded documents (never auto-deleted)
- `./data/` — SQLite database

## Development

```bash
npm install
npm run dev
npm test
```

## Testing

```bash
# Unit tests
npm test tests/unit/

# Integration tests
npm test tests/integration/

# All tests
npm test
```

## MCP Integration

Connect Claude Desktop or other MCP clients to `http://127.0.0.1:8081`.

Available tools:
- `wiki_read` — search facts
- `wiki_ingest_text` — add synthetic document
- `wiki_list_facts` — list all facts
- `wiki_create_task` / `wiki_complete_task` — task management
- `wiki_run_librarian` / `wiki_run_heal` — maintenance

## License

MIT
```

- [ ] **Step 2: Create .env.example**

```bash
# .env.example
LLM_BASE_URL=http://host.docker.internal:11434/v1
LLM_API_KEY=
LLM_MODEL=llama3.2
EMBED_MODEL=nomic-embed-text
PORT=8080
MCP_PORT=8081
WIKI_ENTITY_ID=default
```

- [ ] **Step 3: Commit**

```bash
git add README.md .env.example
git commit -m "docs: README with setup and usage"
```

---

## Self-Review

### Spec Coverage

✅ **OutboxSQLiteAdapter** — Task 4 implements mirroring, collision handling via source_ref_map, prune at 100k
✅ **LLMClient** — Task 3 implements OpenAI-compat with retry/backoff
✅ **Watcher** — Task 7 implements dual chokidar with .wikiignore, path.relative for cross-platform
✅ **Reconciler** — Task 8 implements FIFO queue, hash dedup, collision handling with source_ref_map writes, statSync before read
✅ **BootRescan** — Task 9 implements full hash diff, binary file UTF-8 test, statSync before read, path.relative
✅ **Fastify HTML** — Tasks 10-12 implement BBS UI with all routes
✅ **MCP server** — Task 13 implements HTTP+SSE (not stdio), per-item resources (wiki://fact/{id}, wiki://task/{id})
✅ **Docker** — Task 15 implements multi-stage Dockerfile with alpine build deps, Compose v5.1.3
✅ **Tests** — Task 16 integration tests with mkdirSync imports
✅ **README** — Task 17 implements setup docs

### Breaking Changes from Initial Plan

- **MCP Transport**: Changed from stdio to SSE over HTTP on 127.0.0.1:MCP_PORT per spec requirement
- **Resources**: Changed from collection (wiki://facts) to per-item (wiki://fact/{id}, wiki://task/{id})
- **source_ref_map**: Now used for collision handling; reconciler persists mappings on collision
- **Dockerfile**: Multi-stage to support better-sqlite3 build deps on alpine
- **Path handling**: All path operations use path.relative() instead of string replace

### Verification Needed at Implementation

- SSEServerTransport API (@modelcontextprotocol/sdk) — endpoint structure
- core-llm-wiki ingestDocument source_type default behavior
- better-sqlite3 prebuilt binary availability for node:24-alpine

All tasks implement complete, runnable code with exact paths and commands.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-08-llm-wiki-docker.md`. Two execution options:

**1. Subagent-Driven (recommended)** - Fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
