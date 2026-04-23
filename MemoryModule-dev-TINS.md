<!-- TINS Specification v1.0 -->
<!-- ZS:COMPLEXITY:HIGH -->
<!-- ZS:PRIORITY:HIGH -->
<!-- ZS:PLATFORM:NODE+WEB -->
<!-- ZS:LANGUAGE:TYPESCRIPT -->
<!-- ZS:FRONTEND:VITE+REACT -->
<!-- ZS:BACKEND:NODE+SQLITE -->
<!-- ZS:AUTH:PLUGGABLE -->
<!-- ZS:DEPLOY:STANDALONE -->
<!-- ZS:LICENSE:MIT -->

# Memory Module

## Description

Memory Module is a model-agnostic, platform-agnostic agent memory harness that preserves agent knowledge across sessions, restarts, and context compactions. It implements the five-channel retrieval pipeline with Reciprocal Rank Fusion, a multi-stage ingestion pipeline with SHA-256 content-addressed idempotency, four classified memory types with supersession chains, and HyDE-augmented embedding described in the companion grounding document `CF-AgentMemory-grounding.md`.

The module is designed for local development and whitelabel deployment. Everything runs on a developer machine or a single server: a Node.js backend with SQLite for per-profile storage and `sqlite-vec` for vector search, a Vite/React/TypeScript chat foundation with a pluggable chat-model adapter, and a minimal npm SDK that hides the transport layer. There are no managed-cloud dependencies — no Workers, no Durable Objects, no managed vector database, no specific auth provider. Production deployment targets any host that runs Node.js.

The architecture is deliberately model-agnostic and adapter-driven. Every LLM binding, every chat model, and every authentication scheme is swappable. The same module can back any future model choice without touching pipeline code, and any operator can rebrand and redeploy without touching the pipeline layer.

---

## Functionality

### Core Features

- **Profile-scoped memory stores**, each backed by a single SQLite file, addressable by name and fully isolated from other profiles
- **Five memory operations**: `ingest`, `remember`, `recall`, `list`, `forget`
- **Ingestion pipeline**: SHA-256 ID generation → parallel full/detail extraction → 8-check verification → four-way classification → SQLite write → async vectorization
- **Retrieval pipeline**: query analysis + embedding → five parallel channels → Reciprocal Rank Fusion → synthesis
- **Four memory types**: Fact, Event, Instruction, Task — each with distinct indexing rules
- **Supersession chains** for keyed types (Fact, Instruction) — versions tracked via forward pointer, never deleted
- **HyDE-augmented retrieval** — hypothetical answer statements improve recall on abstract queries
- **Deterministic date math** — temporal computation bypasses the LLM
- **Standalone React chat foundation** for isolated development, evaluation, and as a reference integration surface
- **Pluggable chat-model adapter** (`ChatAdapter` interface) with reference implementations for OpenAI-compatible endpoints (OpenAI, Ollama, LM Studio, vLLM, LocalAI), Anthropic, and a deterministic mock
- **Pluggable auth adapter** (`AuthAdapter` interface) with reference implementations for a dev shared-secret, a generic JWT verifier, and an open (no-auth) local mode
- **Harness SDK** for Vite/React/TypeScript integration with any host chat system
- **Full export** of all memories per profile (portability guarantee)
- **Benchmark harness** against LoCoMo, LongMemEval, and BEAM (corpora not redistributed; fetched by the operator)
- **Local-first deployment** via `pnpm dev`; production deployment via `pnpm build && pnpm start` on any Node-capable host
- **Whitelabel-friendly** — no project-specific branding, colours, names, or URLs in shipped code

### System Boundaries and Division of Concerns

The Memory Module is a closed system with exactly three external interfaces:

```
+------------------------------------------------------------------+
|                      Existing Chat System                         |
|                      (host, untouched)                            |
+--------+---------------------------------------------------------+
         |
         | Interface 1: Harness SDK (npm package, HTTP under the hood)
         v
+------------------------------------------------------------------+
|                       Memory Module                               |
|                                                                   |
|  +-----------------+     +-------------------+                    |
|  | Harness SDK     |     | Chat Foundation   |  Interface 2:      |
|  | (TS client)     |     | (Vite/React SPA)  |  REST API          |
|  +--------+--------+     +---------+---------+                    |
|           |                        |                              |
|           +------------+-----------+                              |
|                        v                                          |
|           +--------------------------+                            |
|           | Node.js Server           |                            |
|           | (Fastify + Pipelines)    |                            |
|           +--------------------------+                            |
|                        |                                          |
|       +----------------+----------------+                         |
|       v                v                v                         |
|  +---------+      +----------+     +----------+                   |
|  | SQLite  |      |sqlite-vec|     | Model    |                   |
|  | per     |      |(same DB) |     | Providers|                   |
|  | Profile |      |          |     |(pluggable|                   |
|  +---------+      +----------+     +----------+                   |
+------------------------------------------------------------------+
         ^
         | Interface 3: Benchmark runner (dev only)
         |
+------------------------------------------------------------------+
| LoCoMo / LongMemEval / BEAM corpora (operator-supplied)           |
+------------------------------------------------------------------+
```

**Cross-pollination prohibitions** (enforced by code structure, not convention):

1. Memory pipeline modules must not import from chat-foundation modules. Directory-level ESLint boundaries enforce this.
2. Memory types must not extend or reuse chat message types. They are defined independently even if shapes overlap.
3. The chat model is swappable — its adapter sits behind a `ChatAdapter` interface. No pipeline code and no chat-foundation code imports a chat-model SDK directly.
4. The auth scheme is swappable — it sits behind an `AuthAdapter` interface. No pipeline code imports an auth SDK directly.
5. Model bindings (extractor, verifier, classifier, synthesizer, embedder) are resolved through a `ModelRegistry` — no pipeline code imports a specific model SDK.
6. Persistence bindings (database, vector store) sit behind `Storage` and `VectorStore` interfaces. The reference implementations use SQLite + sqlite-vec; the interfaces accommodate other backends without pipeline changes.

### Operations and Contracts

Each operation is defined by a precise input/output contract.

#### `ingest(messages, options)`

Bulk extraction from a conversation. Called by a harness at compaction time.

**Input:**
```typescript
{
  messages: Array<{
    role: "user" | "assistant" | "system" | "tool",
    content: string,
    timestamp?: string  // ISO 8601, defaults to now
  }>,
  options: {
    sessionId: string,              // Required, groups messages
    ingestedAt?: string,            // ISO 8601, defaults to now; used for relative date resolution
    classifierHint?: "default" | "bias-facts" | "bias-events"  // optional, default "default"
  }
}
```

**Output:**
```typescript
{
  ingestId: string,                 // UUID for this ingest operation
  messagesSeen: number,
  memoriesCreated: number,
  memoriesSuperseded: number,
  memoriesDropped: number,          // Dropped by verifier
  durationMs: number,
  vectorizationQueued: boolean      // true if async vectorization dispatched
}
```

**Behavior:**
- Idempotent on re-ingestion (content-addressed IDs + `INSERT OR IGNORE`)
- Returns to caller before vectorization completes
- Does not block on LLM extraction of other concurrent ingests for the same profile (a per-profile async mutex serializes writes)

#### `remember(input)`

Model-driven single-memory write.

**Input:**
```typescript
{
  content: string,                  // 1-4000 chars
  type?: "fact" | "event" | "instruction" | "task",  // Optional, auto-classified if omitted
  topicKey?: string,                // Optional, auto-generated for keyed types if omitted
  sessionId: string,                // Required
  timestamp?: string                // ISO 8601, defaults to now
}
```

**Output:**
```typescript
{
  memoryId: string,                 // SHA-256 content-addressed
  type: "fact" | "event" | "instruction" | "task",
  topicKey: string | null,
  supersededMemoryId: string | null // If this replaced a prior keyed memory
}
```

#### `recall(query, options?)`

Run the five-channel retrieval pipeline and return a synthesized answer.

**Input:**
```typescript
{
  query: string,                    // 1-1000 chars
  options?: {
    maxCandidates?: number,         // default 20
    includeRawCandidates?: boolean, // default false; if true, include ranked candidates in output
    sessionFilter?: string | null,  // optional, scope to a single session
    temporalNow?: string            // ISO 8601, for deterministic date math; defaults to now
  }
}
```

**Output:**
```typescript
{
  queryId: string,
  result: string,                   // Natural-language synthesized answer
  candidates?: Array<{              // Only if includeRawCandidates true
    memoryId: string,
    content: string,
    type: "fact" | "event" | "instruction" | "task",
    rrfScore: number,
    channelHits: Array<"fts" | "fact-key" | "direct-vec" | "hyde-vec" | "raw-msg">
  }>,
  durationMs: number
}
```

#### `list(options?)`

Enumerate memories for inspection.

**Input:**
```typescript
{
  options?: {
    type?: "fact" | "event" | "instruction" | "task",
    sessionId?: string,
    includeSuperseded?: boolean,    // default false
    limit?: number,                 // default 100, max 1000
    cursor?: string                 // opaque pagination cursor
  }
}
```

**Output:**
```typescript
{
  memories: Array<MemoryRecord>,    // See Data Models below
  nextCursor: string | null
}
```

#### `forget(memoryId)`

Mark a memory as no longer true/important. Does not delete — writes a tombstone so supersession chains remain intact.

**Input:**
```typescript
{ memoryId: string }
```

**Output:**
```typescript
{ forgotten: boolean, memoryId: string }
```

#### `export(options?)`

Portability guarantee: full dump of profile contents.

**Input:**
```typescript
{
  options?: {
    format?: "json" | "ndjson",     // default "json"
    includeRawMessages?: boolean,    // default true
    includeVectors?: boolean         // default false
  }
}
```

**Output:** A streamable response (application/json or application/x-ndjson) containing all memories, messages, and optionally vectors for the profile.

### User Flows

#### Flow 1 — Harness-driven compaction ingest

1. Host chat system detects context budget pressure and decides to compact
2. Host calls `profile.ingest(messagesToCompact, { sessionId })` via the SDK
3. Memory Module's server receives the request, resolves the profile's SQLite file, acquires the per-profile write mutex
4. Ingestion pipeline runs (ID gen → parallel extraction → verification → classification → SQLite write)
5. Response returned to host; vectorization runs asynchronously in the background job runner
6. Host discards the compacted messages from its live context
7. Later, when the model needs context, host surfaces `recall` as a tool or preloads via a synthetic system message

#### Flow 2 — Model-driven remember-during-task

1. Model encounters a concrete fact it judges worth persisting
2. Model calls the `remember` tool with `{ content, sessionId }`
3. Tool dispatches to the server
4. Memory is classified, keyed if appropriate, stored, vectorized async
5. Response includes memoryId; host may surface briefly to user

#### Flow 3 — Model-driven recall

1. Model calls the `recall` tool with a natural-language query
2. Pipeline runs the five parallel channels, fuses, synthesizes
3. Model receives a natural-language answer string
4. Model uses the answer in its next output

#### Flow 4 — Benchmark run

1. Operator runs `pnpm bench --suite locomo --profile bench-run-042`
2. Runner spins up an isolated profile (new SQLite file), ingests benchmark conversations, issues benchmark queries, scores responses
3. Results written to `./bench-results/` as JSON; diffed against the previous run
4. Trend analysis surfaces regressions

#### Flow 5 — Portability export

1. User requests their data
2. `profile.export()` streams NDJSON of all memories + raw messages
3. File is deliverable; importable into any future store that honours the schema

### Edge Cases and Error States

| Condition | Behavior |
|-----------|----------|
| Empty `messages` array on ingest | Return `memoriesCreated: 0`, no LLM calls made |
| Message content exceeds 50K chars | Split at sentence boundaries; preserve chunking overlap |
| Duplicate ingest (same content hashes) | `INSERT OR IGNORE` skips silently; response shows `memoriesCreated: 0` if all duplicates |
| Verifier drops all extracted candidates | Return normally with `memoriesDropped: N, memoriesCreated: 0` |
| `recall` on empty profile | Return `result: "No relevant memories found."`, empty candidates |
| `recall` query exceeds 1000 chars | Return 400 error with message `"Query exceeds 1000 character limit"` |
| Vector upsert failure | Log error, retain SQLite row, retry via job runner (max 5 attempts); memory remains FTS-discoverable throughout |
| LLM provider rate limit hit | Exponential backoff with jitter, max 3 retries, then surface `{ error: "rate-limited" }` |
| Supersession target already forgotten | New memory stored normally; no pointer written |
| `forget` on non-existent memoryId | Return `{ forgotten: false, memoryId }` |
| Profile does not exist | Auto-created on first access (new SQLite file allocated on demand) |
| Concurrent ingests to same profile | Per-profile async mutex serializes; earlier one completes first |
| Topic key collision across sessions | Same key = same logical fact, supersession applies globally within profile |
| `temporalNow` in the future | Accepted; used only for relative-date resolution |
| Model returns malformed JSON in extraction | Retry once with `"repair this JSON"` prompt; if still bad, drop the chunk and log |
| Model returns zero extractions on valid input | Trust it — some chunks legitimately contain nothing memorable |
| SQLite file locked by external process | Retry with exponential backoff, 3 attempts, then 503 |
| Disk full during write | Surface `{ error: "storage-unavailable" }`; pipeline state unchanged |

---

## Prerequisites

A fresh developer machine needs the following before `pnpm dev` works end-to-end:

- **Node.js** ≥ 20 LTS
- **pnpm** ≥ 9
- A C toolchain for native modules (`better-sqlite3` and `sqlite-vec` compile on install):
  - macOS: Xcode Command Line Tools
  - Debian/Ubuntu: `build-essential`, `python3`
  - Windows: windows-build-tools or Visual Studio Build Tools
- **One embedding model source** — either:
  - An OpenAI-compatible HTTP endpoint (OpenAI, Ollama with `nomic-embed-text` or similar, LM Studio, vLLM, LocalAI), OR
  - Anthropic API access (for LLM only) paired with a separate embedding source
- **One chat model source** — same list as above
- Roughly 500 MB of free disk for `node_modules`, plus whatever the operator allocates for profile SQLite files (a typical profile with 10K memories occupies 50–200 MB depending on message volume)

Optional:
- **Docker** — a reference `Dockerfile` and `docker-compose.yml` are provided for one-command deployment
- **Benchmark corpora** — LoCoMo / LongMemEval / BEAM are not bundled. Acquire them from their upstream sources under their respective licenses and place them at `./bench/corpora/{locomo,longmemeval,beam}/`

No cloud accounts, no managed services, and no vendor-specific CLIs are required to run the system locally or in a single-box production deployment.

---

## Technical Implementation

### Repository Layout

```
memory-module/
├── apps/
│   ├── server/                      # Node.js backend
│   │   ├── src/
│   │   │   ├── index.ts             # entrypoint, config load, Fastify boot
│   │   │   ├── server.ts            # Fastify app factory
│   │   │   ├── config.ts            # env + config validation via Zod
│   │   │   ├── rest/                # REST API handlers
│   │   │   │   ├── ingest.ts
│   │   │   │   ├── remember.ts
│   │   │   │   ├── recall.ts
│   │   │   │   ├── list.ts
│   │   │   │   ├── forget.ts
│   │   │   │   └── export.ts
│   │   │   ├── profile/             # Per-profile coordinator
│   │   │   │   ├── profile-manager.ts  # resolves & caches Profile instances
│   │   │   │   ├── profile.ts          # Profile class (all 5 ops)
│   │   │   │   ├── mutex.ts            # per-profile async mutex
│   │   │   │   └── sqlite-schema.ts    # DDL bootstrap
│   │   │   ├── storage/             # Persistence abstraction
│   │   │   │   ├── Storage.ts          # interface
│   │   │   │   ├── VectorStore.ts      # interface
│   │   │   │   └── sqlite/
│   │   │   │       ├── SqliteStorage.ts
│   │   │   │       └── SqliteVectorStore.ts  # uses sqlite-vec
│   │   │   ├── auth/                # Auth abstraction
│   │   │   │   ├── AuthAdapter.ts      # interface
│   │   │   │   ├── shared-secret.ts    # dev default
│   │   │   │   ├── jwt.ts              # generic JWT verifier
│   │   │   │   └── none.ts             # localhost-only open mode
│   │   │   ├── jobs/                # Async vectorization runner
│   │   │   │   ├── job-runner.ts
│   │   │   │   └── vectorize-job.ts
│   │   │   ├── pipelines/
│   │   │   │   ├── ingest/          # Ingestion pipeline
│   │   │   │   │   ├── id-gen.ts
│   │   │   │   │   ├── extract-full.ts
│   │   │   │   │   ├── extract-detail.ts
│   │   │   │   │   ├── merge.ts
│   │   │   │   │   ├── verify.ts
│   │   │   │   │   ├── classify.ts
│   │   │   │   │   └── vectorize.ts
│   │   │   │   └── recall/          # Retrieval pipeline
│   │   │   │       ├── query-analyze.ts
│   │   │   │       ├── embed.ts
│   │   │   │       ├── channel-fts.ts
│   │   │   │       ├── channel-fact-key.ts
│   │   │   │       ├── channel-direct-vec.ts
│   │   │   │       ├── channel-hyde-vec.ts
│   │   │   │       ├── channel-raw-msg.ts
│   │   │   │       ├── rrf.ts
│   │   │   │       ├── temporal.ts
│   │   │   │       └── synthesize.ts
│   │   │   ├── models/              # Model-agnostic bindings
│   │   │   │   ├── registry.ts      # ModelRegistry
│   │   │   │   ├── roles.ts         # Extractor, Verifier, Classifier, Synthesizer, Embedder
│   │   │   │   └── providers/
│   │   │   │       ├── openai-compat.ts   # OpenAI, Ollama, LM Studio, vLLM, LocalAI
│   │   │   │       ├── anthropic.ts
│   │   │   │       └── mock.ts            # For tests
│   │   │   ├── types.ts             # Shared types (Memory, Message, etc.)
│   │   │   ├── constants.ts         # Chunk sizes, RRF weights, retry caps, timeouts
│   │   │   └── prompts/             # Prompt templates, one file per role
│   │   │       ├── extract.ts
│   │   │       ├── verify.ts
│   │   │       ├── classify.ts
│   │   │       ├── query-analyze.ts
│   │   │       └── synthesize.ts
│   │   ├── .env.example
│   │   └── package.json
│   │
│   ├── chat-foundation/             # Standalone Vite/React/TS chat (reference UI)
│   │   ├── src/
│   │   │   ├── main.tsx
│   │   │   ├── App.tsx
│   │   │   ├── adapters/
│   │   │   │   ├── ChatAdapter.ts          # interface
│   │   │   │   ├── openai-compat.ts        # default
│   │   │   │   ├── anthropic.ts
│   │   │   │   └── mock.ts
│   │   │   ├── components/
│   │   │   │   ├── ChatWindow.tsx
│   │   │   │   ├── MessageList.tsx
│   │   │   │   ├── Composer.tsx
│   │   │   │   ├── MemoryPanel.tsx         # recall results / memory inspector
│   │   │   │   └── Settings.tsx            # adapter + endpoint config
│   │   │   ├── hooks/
│   │   │   │   └── useMemoryTools.ts       # binds memory ops as model tools
│   │   │   └── config.ts                   # loads settings from localStorage
│   │   ├── index.html
│   │   ├── vite.config.ts
│   │   └── package.json
│   │
│   └── bench/                       # Benchmark runner
│       ├── src/
│       │   ├── run.ts
│       │   ├── suites/
│       │   │   ├── locomo.ts
│       │   │   ├── longmemeval.ts
│       │   │   └── beam.ts
│       │   └── scorers/
│       └── package.json
│
├── packages/
│   ├── sdk/                         # @memory-module/sdk (npm package)
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── client.ts            # MemoryClient
│   │   │   ├── profile.ts           # Profile class (thin HTTP wrapper)
│   │   │   ├── tools.ts             # Model tool definitions (model-agnostic)
│   │   │   └── types.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   └── shared-types/                # Shared types; no runtime dependencies
│       └── src/index.ts
│
├── profiles/                        # Runtime SQLite files (gitignored)
│   └── .gitkeep
│
├── scripts/
│   ├── dev.sh                       # runs server + chat-foundation concurrently
│   ├── build.sh
│   ├── start.sh                     # production start
│   └── reset-profile.sh             # nukes a profile SQLite file
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── .eslintrc.cjs                    # enforces directory boundaries
├── pnpm-workspace.yaml
├── package.json
├── LICENSE                          # MIT
└── README.md
```

### Architecture

Memory Module is a pnpm monorepo with three apps and two shared packages. The only import graph allowed between them is:

```
chat-foundation  ───imports──▶  @memory-module/sdk
server           ───imports──▶  @memory-module/shared-types
sdk              ───imports──▶  @memory-module/shared-types
bench            ───imports──▶  @memory-module/sdk
```

No other cross-app imports are permitted. This is enforced by the ESLint `no-restricted-imports` rule configured in `.eslintrc.cjs` (see Division-of-Concerns Enforcement below).

### Runtime Shape

A single Node.js process:

1. **Fastify HTTP server** listening on `HOST:PORT` (default `127.0.0.1:8787`).
2. **Profile Manager** — lazily instantiates `Profile` objects keyed by profile name. Each `Profile` owns exactly one `better-sqlite3` database handle and one per-profile async mutex for write serialization. Profile instances are cached in memory.
3. **Background job runner** — a single `setInterval` loop (default tick 500ms) that drains the `vectorize_queue` for every active profile in batches of up to 50 rows per profile per tick.
4. **Graceful shutdown** — on `SIGINT` / `SIGTERM`: stop accepting new requests, flush in-flight jobs up to 30s, close all SQLite handles, exit.

This fits on any machine that can run Node 20. For higher throughput or multi-tenant deployment, operators run multiple instances behind a reverse proxy keyed by profile name (the per-profile SQLite file becomes the sharding unit).

### Model-Agnostic Binding

Every LLM-or-embedding touchpoint in the pipeline resolves through `ModelRegistry`. A model is referenced by its **role**, not its identity.

```typescript
// apps/server/src/models/roles.ts
export interface ExtractorModel {
  extract(chunk: TranscriptChunk, opts: ExtractOpts): Promise<ExtractedMemory[]>;
}
export interface VerifierModel {
  verify(candidate: ExtractedMemory, source: TranscriptChunk): Promise<VerifyResult>;
}
export interface ClassifierModel {
  classify(memory: VerifiedMemory): Promise<ClassificationResult>;
}
export interface QueryAnalyzerModel {
  analyze(query: string): Promise<QueryAnalysis>;
}
export interface SynthesizerModel {
  synthesize(query: string, candidates: RankedCandidate[], precomputedFacts: PrecomputedFact[]): Promise<string>;
}
export interface EmbedderModel {
  embed(texts: string[]): Promise<number[][]>;
}

// apps/server/src/models/registry.ts
export class ModelRegistry {
  constructor(private cfg: ResolvedConfig) {}
  extractor(): ExtractorModel     { /* resolves via cfg.models.extractor */ }
  verifier(): VerifierModel       { /* ... */ }
  classifier(): ClassifierModel   { /* ... */ }
  queryAnalyzer(): QueryAnalyzerModel { /* ... */ }
  synthesizer(): SynthesizerModel { /* ... */ }
  embedder(): EmbedderModel       { /* ... */ }
}
```

Swapping providers is a `config.ts` edit plus an optional environment variable change. Pipeline code imports interfaces only. Per-role provider selection means the operator can, for example, use a cheap local model for extraction/classification and a stronger hosted model for synthesis.

### Persistence — SQLite per Profile

One SQLite file per profile, stored at `./profiles/<name>.db`. `better-sqlite3` is used for synchronous, transactional access; SQLite's FTS5 virtual table provides full-text search; `sqlite-vec` provides the vector index inside the same file.

**Rationale for one-file-per-profile:**
- Strong tenant isolation without any provisioning layer
- Easy backup and relocation (copy the file)
- No cross-profile query surface by construction
- Zero-cost profile creation (file allocated on first write)

#### Schema

```sql
-- Raw conversation messages (content-addressed)
CREATE TABLE IF NOT EXISTS messages (
  id              TEXT PRIMARY KEY,
  session_id      TEXT NOT NULL,
  role            TEXT NOT NULL,
  content         TEXT NOT NULL,
  timestamp       TEXT NOT NULL,
  ingested_at     TEXT NOT NULL,
  ingest_id       TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_messages_session ON messages(session_id);
CREATE INDEX IF NOT EXISTS idx_messages_ingest  ON messages(ingest_id);

-- Classified memories
CREATE TABLE IF NOT EXISTS memories (
  id                   TEXT PRIMARY KEY,
  type                 TEXT NOT NULL,
  content              TEXT NOT NULL,
  topic_key            TEXT,
  session_id           TEXT NOT NULL,
  timestamp            TEXT NOT NULL,
  ingested_at          TEXT NOT NULL,
  ingest_id            TEXT NOT NULL,
  superseded_by        TEXT,
  forgotten_at         TEXT,
  search_queries       TEXT NOT NULL,       -- JSON array
  source_line_indices  TEXT                 -- JSON array
);
CREATE INDEX IF NOT EXISTS idx_memories_topic
  ON memories(topic_key)
  WHERE topic_key IS NOT NULL
    AND superseded_by IS NULL
    AND forgotten_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_memories_type          ON memories(type);
CREATE INDEX IF NOT EXISTS idx_memories_session       ON memories(session_id);
CREATE INDEX IF NOT EXISTS idx_memories_superseded_by ON memories(superseded_by);

-- FTS5 over memory content (Porter-stemmed)
CREATE VIRTUAL TABLE IF NOT EXISTS memories_fts USING fts5(
  content, topic_key, search_queries,
  content='memories',
  content_rowid='rowid',
  tokenize='porter unicode61'
);

-- FTS5 over raw messages (safety-net channel)
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
  content,
  content='messages',
  content_rowid='rowid',
  tokenize='porter unicode61'
);

-- Vector index over memory embeddings (sqlite-vec)
-- Dimension must match the embedder's output (default 768; set from config).
CREATE VIRTUAL TABLE IF NOT EXISTS memories_vec USING vec0(
  memory_id TEXT PRIMARY KEY,
  embedding FLOAT[768]
);

-- Bookkeeping
CREATE TABLE IF NOT EXISTS ingest_log (
  ingest_id        TEXT PRIMARY KEY,
  session_id       TEXT NOT NULL,
  started_at       TEXT NOT NULL,
  finished_at      TEXT,
  messages_seen    INTEGER NOT NULL DEFAULT 0,
  memories_created INTEGER NOT NULL DEFAULT 0,
  memories_dropped INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS vectorize_queue (
  memory_id        TEXT PRIMARY KEY,
  queued_at        TEXT NOT NULL,
  attempts         INTEGER NOT NULL DEFAULT 0,
  last_error       TEXT
);

-- Triggers keep FTS mirrors in sync with base tables
CREATE TRIGGER IF NOT EXISTS memories_ai AFTER INSERT ON memories BEGIN
  INSERT INTO memories_fts(rowid, content, topic_key, search_queries)
    VALUES (new.rowid, new.content, new.topic_key, new.search_queries);
END;
CREATE TRIGGER IF NOT EXISTS memories_ad AFTER DELETE ON memories BEGIN
  INSERT INTO memories_fts(memories_fts, rowid, content, topic_key, search_queries)
    VALUES('delete', old.rowid, old.content, old.topic_key, old.search_queries);
END;
CREATE TRIGGER IF NOT EXISTS messages_ai AFTER INSERT ON messages BEGIN
  INSERT INTO messages_fts(rowid, content) VALUES (new.rowid, new.content);
END;
```

Writes use `INSERT OR IGNORE` for idempotency. Multi-row writes run in a transaction via `better-sqlite3`'s `.transaction()` helper.

### Data Models

```typescript
// Message ingested into the pipeline
type Message = {
  role: "user" | "assistant" | "system" | "tool";
  content: string;
  timestamp?: string;      // ISO 8601
};

// Transcript chunk for extraction
type TranscriptChunk = {
  chunkId: string;
  messages: Array<Message & { lineIndex: number }>;
  temporalAnchor: string;  // The "now" used to resolve relative dates
};

// Raw extraction output before verification
type ExtractedMemory = {
  tentativeContent: string;
  tentativeType?: "fact" | "event" | "instruction" | "task";
  sourceLineIndices: number[];
  extractionPass: "full" | "detail";
};

// After verification
type VerifiedMemory = ExtractedMemory & {
  content: string;
  verified: true;
};

// After classification
type ClassificationResult = {
  type: "fact" | "event" | "instruction" | "task";
  topicKey: string | null;
  searchQueries: string[];
  timestamp: string | null;
};

// Final stored form
type MemoryRecord = {
  id: string;
  type: "fact" | "event" | "instruction" | "task";
  content: string;
  topicKey: string | null;
  sessionId: string;
  timestamp: string;
  ingestedAt: string;
  ingestId: string;
  supersededBy: string | null;
  forgottenAt: string | null;
  searchQueries: string[];
  sourceLineIndices: number[];
};

type QueryAnalysis = {
  topicKeys: string[];
  ftsTerms: string[];
  hydeStatement: string;
};

type RankedCandidate = {
  memory: MemoryRecord;
  rrfScore: number;
  channelHits: Array<"fts" | "fact-key" | "direct-vec" | "hyde-vec" | "raw-msg">;
};

type PrecomputedFact = {
  kind: "temporal-diff" | "temporal-resolve";
  value: string;
};
```

### Ingestion Pipeline — Algorithms

#### Stage 1: ID Generation

```typescript
// apps/server/src/pipelines/ingest/id-gen.ts
import { createHash } from "node:crypto";

export function generateMessageId(sessionId: string, role: string, content: string): string {
  const input = `${sessionId}\x1f${role}\x1f${content}`;
  return createHash("sha256").update(input).digest("hex").slice(0, 32); // 128 bits
}

export function generateMemoryId(sessionId: string, content: string, type: string): string {
  const input = `${sessionId}\x1f${type}\x1f${content}`;
  return createHash("sha256").update(input).digest("hex").slice(0, 32);
}
```

#### Stage 2: Chunking and Parallel Extraction

- **Full pass**: chunk at **≤10,000 characters** with **two-message overlap**, process **up to 4 chunks concurrently**.
- **Detail pass**: only for conversations with **9 or more messages**. Uses overlapping windows of **~5 messages** targeted at concrete-value extraction.
- Both passes run in parallel. Results are merged by `generateMemoryId` — duplicates collapse.

The extraction prompt receives:
- role-labelled transcript
- relative date resolution (e.g. `"yesterday" → "2026-04-14"` using `ingestedAt` as anchor)
- line indices for provenance

#### Stage 3: Verification — Eight Checks

Each candidate goes through all eight checks in a single LLM call. The call returns one of three outcomes per check: `pass`, `correct` (with a corrected version), `drop`.

| # | Check | What it catches |
|---|-------|-----------------|
| 1 | **Entity identity** | "John" when transcript said "Jon" |
| 2 | **Object identity** | Confused noun referents |
| 3 | **Location context** | Wrong geographic attribution |
| 4 | **Temporal accuracy** | "yesterday" resolved to wrong day |
| 5 | **Organizational context** | Company/team mix-ups |
| 6 | **Completeness** | Partial extraction of multi-part facts |
| 7 | **Relational context** | Wrong actor ↔ action pairing |
| 8 | **Inference support** | Claim not actually supported by transcript |

If any check returns `drop`, the candidate is dropped. If any returns `correct`, the corrected content flows forward.

#### Stage 4: Classification

Classifier outputs, for each verified memory:

- **type**: fact | event | instruction | task
- **topicKey**: normalized key (lowercase, snake_case, stopwords removed) — only for fact and instruction
- **searchQueries**: 3–5 interrogative phrasings of what this memory answers
- **timestamp**: only for events; ISO 8601

Keyed types enter supersession logic: if `topicKey` matches an existing non-superseded, non-forgotten memory, the old memory's `superseded_by` is set to the new memory's id.

#### Stage 5: Storage

Write order inside a single SQLite transaction:

1. `INSERT OR IGNORE` new messages into `messages` (triggers mirror to `messages_fts`)
2. `INSERT OR IGNORE` new memories into `memories` (triggers mirror to `memories_fts`)
3. For each keyed memory with an existing predecessor: `UPDATE memories SET superseded_by = ? WHERE topic_key = ? AND superseded_by IS NULL AND forgotten_at IS NULL AND id != ?`
4. `INSERT` each non-task memoryId into `vectorize_queue` (tasks are excluded from the vector index)
5. `INSERT` the ingest log row with final counts

The background job runner picks up `vectorize_queue` entries on its next tick.

#### Stage 6: Asynchronous Vectorization

The job runner ticks every `VECTORIZE_TICK_MS` (default 500ms). Per tick, per profile:

1. Select up to 50 rows from `vectorize_queue`
2. For each, construct embedding text:
   ```
   [search_queries joined with " ? "] ? [content]
   ```
   This prepends the 3–5 interrogative queries to the content, bridging declarative-write vs interrogative-read vocabulary.
3. Call embedder in batch
4. Upsert into `memories_vec`
5. For any superseded memories (`superseded_by IS NOT NULL`), delete their vectors
6. Delete successful rows from `vectorize_queue`
7. Increment `attempts` and set `last_error` for failures; drop rows after 5 failed attempts

### Retrieval Pipeline — Algorithms

#### Stage 1: Query Analysis + Embedding (concurrent)

Query analyzer LLM returns:

```json
{
  "topicKeys": ["package_manager_preference", "build_tool_choice"],
  "ftsTerms": ["pnpm", "npm", "package", "manager"],
  "hydeStatement": "The user prefers pnpm over npm as their package manager."
}
```

Embedder is called **twice in parallel**:
- Once on the raw query → `directVec`
- Once on `hydeStatement` → `hydeVec`

#### Stage 2: Five Parallel Channels

Each channel returns a ranked list of up to `maxCandidates` memoryIds.

**Channel 1 — Full-Text Search (Porter stemming)**
```sql
SELECT memories.id, memories_fts.rank
FROM memories_fts
JOIN memories ON memories.rowid = memories_fts.rowid
WHERE memories_fts MATCH ?
  AND memories.superseded_by IS NULL
  AND memories.forgotten_at IS NULL
ORDER BY rank
LIMIT ?
```
Match expression constructed from `ftsTerms` with OR joins.

**Channel 2 — Fact-Key Lookup**
```sql
SELECT id FROM memories
WHERE topic_key IN (?, ?, ?)
  AND superseded_by IS NULL
  AND forgotten_at IS NULL
LIMIT ?
```
Input order of topic keys is preserved in application code (not SQL), ranked highest-first from `QueryAnalysis.topicKeys`.

**Channel 3 — Direct Vector Search**
```sql
SELECT memory_id, distance
FROM memories_vec
WHERE embedding MATCH ?             -- directVec
  AND k = ?                          -- maxCandidates
ORDER BY distance;
```
Results are filtered in the application layer against active memories (non-superseded, non-forgotten, non-task).

**Channel 4 — HyDE Vector Search**
Same as Channel 3 but queried with `hydeVec`.

**Channel 5 — Raw Message Search**
```sql
SELECT messages.id FROM messages_fts
JOIN messages ON messages.rowid = messages_fts.rowid
WHERE messages_fts MATCH ?
ORDER BY rank
LIMIT ?
```
Returns messageIds, not memoryIds. Included as a safety net with the lowest RRF weight.

#### Stage 3: Reciprocal Rank Fusion

```typescript
const CHANNEL_WEIGHTS: Record<Channel, number> = {
  "fact-key":  1.00,
  "fts":       0.70,
  "hyde-vec":  0.65,
  "direct-vec": 0.60,
  "raw-msg":   0.30,
};
const K = 60;

function rrf(channelResults: Record<Channel, string[]>, maxCandidates: number): RankedCandidate[] {
  const scores = new Map<string, { score: number; hits: Set<Channel> }>();
  for (const [channel, ranking] of Object.entries(channelResults) as [Channel, string[]][]) {
    const w = CHANNEL_WEIGHTS[channel];
    ranking.forEach((id, idx) => {
      const rank = idx + 1;
      const delta = w * (1 / (K + rank));
      const entry = scores.get(id) ?? { score: 0, hits: new Set() };
      entry.score += delta;
      entry.hits.add(channel);
      scores.set(id, entry);
    });
  }
  return Array.from(scores.entries())
    .map(([id, { score, hits }]) => ({ id, score, hits }))
    .sort((a, b) => b.score - a.score || tieBreakByRecency(a.id, b.id))
    .slice(0, maxCandidates);
}
```

Ties are broken by recency (newer `ingestedAt` ranks higher).

#### Stage 4: Temporal Precomputation (deterministic)

Regex scan the query for temporal patterns:
- `"X days ago"`, `"last week"`, `"how long since"`, `"how many days between X and Y"`
- ISO dates: `YYYY-MM-DD`

For each match, compute the answer deterministically using `temporalNow` as the anchor. Produce `PrecomputedFact[]`.

Example:
- Query: *"how long since the last deploy?"*
- Retrieved event: `"Deployed v2.4.1 on 2026-04-10"`
- Precomputed fact: `{ kind: "temporal-diff", value: "10 days (between 2026-04-10 and 2026-04-20)" }`

#### Stage 5: Synthesis

The synthesizer LLM receives:
- Original query
- Top candidates (with full content, type, timestamp)
- `PrecomputedFact[]`
- Instruction: *"Use precomputed facts verbatim where relevant; never do date arithmetic yourself."*

Returns the natural-language answer. That string is returned in `result`.

### REST API Surface

Base path: `/v1`

| Method | Path | Body | Response |
|--------|------|------|----------|
| POST | `/profiles/:name/ingest`   | `{ messages, options }` | Ingest result |
| POST | `/profiles/:name/remember` | remember input          | `{ memoryId, type, topicKey, supersededMemoryId }` |
| POST | `/profiles/:name/recall`   | `{ query, options? }`   | recall result |
| GET  | `/profiles/:name/memories?type=&session=&cursor=&limit=` | — | list result |
| POST | `/profiles/:name/forget`   | `{ memoryId }`          | `{ forgotten, memoryId }` |
| GET  | `/profiles/:name/export?format=&includeRawMessages=&includeVectors=` | — | streamed JSON/NDJSON |
| GET  | `/healthz` | — | `{ ok: true }` (no auth) |

All non-`/healthz` requests pass through the configured `AuthAdapter`. Profile names are validated as `^[a-z0-9][a-z0-9_-]{0,62}$` to ensure safe filesystem mapping.

### Pluggable Auth

```typescript
// apps/server/src/auth/AuthAdapter.ts
export interface AuthAdapter {
  /** Returns a principal identifier on success; throws on failure. */
  verify(req: { headers: Record<string, string | undefined> }): Promise<{ principal: string }>;
}
```

Three reference implementations ship with the module:

1. **`shared-secret`** (default for dev) — single token in `AUTH_TOKEN` env var. Client sends `Authorization: Bearer <token>`.
2. **`jwt`** — verifies a JWT against a configurable JWKS URL or a static public key (PEM). Works with any OIDC-compliant issuer. Caches keys in memory with TTL.
3. **`none`** — no auth. Only permitted when `HOST` binds to loopback (`127.0.0.1` or `::1`); otherwise the server refuses to start.

### Pluggable Chat Adapter (Chat Foundation)

```typescript
// apps/chat-foundation/src/adapters/ChatAdapter.ts
export interface ChatAdapter {
  send(params: {
    messages: Array<{ role: "user" | "assistant" | "system" | "tool"; content: string }>;
    tools?: ToolDefinition[];
    onToolCall?: (name: string, input: unknown) => Promise<unknown>;
  }): Promise<{ content: string; toolCalls: Array<{ name: string; input: unknown }> }>;
}
```

Reference implementations:

- **`openai-compat`** — works with OpenAI, Ollama, LM Studio, vLLM, LocalAI, and any other OpenAI-compatible chat-completions endpoint. Configured by `endpoint` (base URL), `apiKey`, and `model`.
- **`anthropic`** — Anthropic Messages API.
- **`mock`** — deterministic scripted replies for UI tests.

Operators white-labelling the chat foundation can add new adapters without touching any other file.

### Harness SDK — `@memory-module/sdk`

```typescript
// packages/sdk/src/client.ts
export class MemoryClient {
  constructor(private cfg: {
    baseUrl: string;
    getAuthToken?: () => Promise<string>;  // omitted in open/none-auth mode
  }) {}

  async getProfile(name: string): Promise<Profile> {
    return new Profile(name, this.cfg);
  }
}

// packages/sdk/src/profile.ts
export class Profile {
  constructor(private name: string, private cfg: ClientConfig) {}

  async ingest(messages: Message[], options: IngestOptions): Promise<IngestResult>   { return this.post("ingest", { messages, options }); }
  async remember(input: RememberInput): Promise<RememberResult>                      { return this.post("remember", input); }
  async recall(query: string, options?: RecallOptions): Promise<RecallResult>        { return this.post("recall", { query, options }); }
  async list(options?: ListOptions): Promise<ListResult>                              { return this.get(`memories${qs(options)}`); }
  async forget(memoryId: string): Promise<ForgetResult>                               { return this.post("forget", { memoryId }); }
  async export(options?: ExportOptions): Promise<ReadableStream>                      { return this.getStream(`export${qs(options)}`); }

  private async post<T>(path: string, body: unknown): Promise<T> { /* fetch + auth */ }
  private async get<T>(path: string): Promise<T> { /* fetch + auth */ }
  private async getStream(path: string): Promise<ReadableStream> { /* fetch + auth */ }
}

// packages/sdk/src/tools.ts — model-agnostic tool definitions
export function buildMemoryTools(profile: Profile) {
  return [
    {
      name: "recall",
      description: "Retrieve information from long-term memory.",
      inputSchema: { type: "object", properties: { query: { type: "string" } }, required: ["query"] },
      execute: async ({ query }: { query: string }) => (await profile.recall(query)).result,
    },
    {
      name: "remember",
      description: "Store something important in long-term memory.",
      inputSchema: {
        type: "object",
        properties: { content: { type: "string" }, sessionId: { type: "string" } },
        required: ["content", "sessionId"],
      },
      execute: async (input) => profile.remember(input),
    },
    {
      name: "forget",
      description: "Mark a memory as no longer true or relevant.",
      inputSchema: {
        type: "object",
        properties: { memoryId: { type: "string" } },
        required: ["memoryId"],
      },
      execute: async (input) => profile.forget(input.memoryId),
    },
    {
      name: "list",
      description: "List stored memories for inspection.",
      inputSchema: { type: "object", properties: {} },
      execute: async () => profile.list(),
    },
  ];
}
```

The tool definitions are intentionally free of any specific model-vendor's tool schema. Host chat systems translate to Anthropic / OpenAI / Google / other tool formats as needed.

### Division-of-Concerns Enforcement

`.eslintrc.cjs`:

```javascript
module.exports = {
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        { group: ['**/pipelines/**'], message: 'Pipeline internals are private. Use the Profile interface.' },
      ],
    }],
  },
  overrides: [
    {
      files: ['apps/server/src/pipelines/ingest/**/*.ts'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['**/pipelines/recall/**'], message: 'Ingest pipeline must not import from recall pipeline.' },
            { group: ['**/chat-foundation/**'], message: 'Server must not import from frontend.' },
          ],
        }],
      },
    },
    {
      files: ['apps/server/src/pipelines/recall/**/*.ts'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['**/pipelines/ingest/**'], message: 'Recall pipeline must not import from ingest pipeline.' },
          ],
        }],
      },
    },
    {
      files: ['apps/chat-foundation/**/*.{ts,tsx}'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['**/apps/server/**'], message: 'Frontend must not import from server. Use the SDK.' },
            { group: ['**/pipelines/**'],   message: 'Frontend must not import pipelines. Use the SDK.' },
          ],
        }],
      },
    },
  ],
};
```

### Configuration

Configuration lives in `apps/server/.env` (for secrets) plus `apps/server/src/config.ts` (for structured defaults). Everything is validated on boot via Zod.

**`.env.example`:**

```bash
# Network
HOST=127.0.0.1
PORT=8787

# Storage
PROFILES_DIR=./profiles
EMBEDDING_DIMENSIONS=768

# Auth: one of shared-secret | jwt | none
AUTH_ADAPTER=shared-secret
AUTH_TOKEN=change-me-in-production
# If AUTH_ADAPTER=jwt:
# AUTH_JWKS_URL=https://issuer.example/.well-known/jwks.json
# AUTH_JWT_AUDIENCE=memory-module
# AUTH_JWT_ISSUER=https://issuer.example

# Models — per-role provider selection
EXTRACTOR_PROVIDER=openai-compat
EXTRACTOR_ENDPOINT=http://localhost:11434/v1
EXTRACTOR_MODEL=llama3.1:8b
EXTRACTOR_API_KEY=ollama

VERIFIER_PROVIDER=openai-compat
VERIFIER_ENDPOINT=http://localhost:11434/v1
VERIFIER_MODEL=llama3.1:8b
VERIFIER_API_KEY=ollama

CLASSIFIER_PROVIDER=openai-compat
CLASSIFIER_ENDPOINT=http://localhost:11434/v1
CLASSIFIER_MODEL=llama3.1:8b
CLASSIFIER_API_KEY=ollama

QUERY_ANALYZER_PROVIDER=openai-compat
QUERY_ANALYZER_ENDPOINT=http://localhost:11434/v1
QUERY_ANALYZER_MODEL=llama3.1:8b
QUERY_ANALYZER_API_KEY=ollama

SYNTHESIZER_PROVIDER=openai-compat
SYNTHESIZER_ENDPOINT=http://localhost:11434/v1
SYNTHESIZER_MODEL=llama3.1:70b
SYNTHESIZER_API_KEY=ollama

EMBEDDER_PROVIDER=openai-compat
EMBEDDER_ENDPOINT=http://localhost:11434/v1
EMBEDDER_MODEL=nomic-embed-text
EMBEDDER_API_KEY=ollama

# Jobs
VECTORIZE_TICK_MS=500
VECTORIZE_BATCH_SIZE=50
```

**`config.ts`** validates the above and exposes a typed, frozen `ResolvedConfig` object consumed by the `ModelRegistry`, the auth layer, the profile manager, and the job runner.

### Deployment

#### Local development

```bash
pnpm install
cp apps/server/.env.example apps/server/.env
pnpm dev        # runs server (http://127.0.0.1:8787) and chat-foundation (http://127.0.0.1:5173) concurrently
```

#### Single-host production

```bash
pnpm install
pnpm -r build
pnpm --filter @memory-module/server start
# Chat foundation is a static bundle — serve apps/chat-foundation/dist/ from any static host or reverse-proxy it through the server.
```

#### Docker

```bash
docker compose up -d
```

The bundled `docker/Dockerfile` produces a single image containing server + static chat-foundation assets. `docker-compose.yml` mounts `./profiles` as a volume so SQLite files persist across container rebuilds.

No CI/CD is mandated in v1. The deployment surface is small enough that CLI-first keeps iteration speed high; operators who want CI can wire their preferred runner to `pnpm -r build && pnpm -r test`.

### Code Conventions

1. **Each pipeline stage is a single file** with one exported function. No deep class hierarchies.
2. **Every prompt lives in `prompts/*.ts` as a tagged template literal** with typed input/output via Zod. Prompts are the tunable surface and should not be buried inside implementation files.
3. **Zod schemas co-located with types.** Every LLM response is parsed through Zod. Malformed responses throw typed errors that the retry logic handles.
4. **Tests live next to source**: `extract-full.ts` + `extract-full.test.ts`. Vitest. One file per behavior.
5. **`pnpm typecheck` as a precommit hook.**
6. **No barrel files.** Direct imports from files to minimize surprise.
7. **Constant values live in `apps/server/src/constants.ts`** (chunk sizes, RRF weights, retry caps, timeouts). Never inline magic numbers.
8. **Agentic-coding readme at repo root** — a top-level `AGENTS.md` summarizes this doc's key points for any coding assistant (human-in-loop coding agents frequently look for such files).

---

## Style Guide

### Code style

- **TypeScript strict mode** everywhere, `noUncheckedIndexedAccess: true`
- **ES2022 target**, ESM only
- **Prettier** with 2-space indent, semicolons, single quotes
- **Zod** for every external boundary (LLM responses, REST inputs, SDK inputs, config)
- **Functional-first** where practical; classes reserved for the `Profile`, `MemoryClient`, and adapter implementations

### Frontend style (chat foundation)

- Tailwind with the default palette; no custom design tokens in v1
- Dark mode by default, light-mode toggle available
- No project-specific branding in shipped code — logo slot in header is a neutral placeholder, titled "Memory Module" by default and overridable via `VITE_APP_TITLE`
- Memory inspector panel shows, per memory: id (truncated 8 chars), type (coloured badge), content, topic key, superseded-by link if any
- Colour cues: Fact = sky-blue, Event = amber, Instruction = emerald, Task = rose

### Typography

- `ui-sans-serif, system-ui` for body
- `ui-monospace, SFMono-Regular` for IDs, topic keys, and raw memory content in the inspector

---

## Testing Scenarios

The implementation must pass every scenario below.

### Unit — Ingestion

1. **Idempotency** — Ingest the same 4-message conversation twice. Second ingest returns `memoriesCreated: 0`.
2. **Content-addressed IDs** — Same `(sessionId, role, content)` on two different ingests produces identical messageId.
3. **Chunking** — A 25K-character conversation produces ≥3 chunks with two-message overlap.
4. **Detail pass skipped on short conversations** — 8-message conversation runs full pass only.
5. **Verifier drop** — Mocked verifier returning `drop` on all candidates yields `memoriesCreated: 0, memoriesDropped: N`.
6. **Classification routing** — Memories of type `task` are not added to `vectorize_queue`.
7. **Supersession** — Ingest A with topic_key `X`, then ingest B with topic_key `X`. `list({ includeSuperseded: true })` shows A's `supersededBy === B.id`.

### Unit — Retrieval

8. **Fact-key exact match outranks** — A fact-key hit beats a direct-vector hit on the same query.
9. **Tasks not in vector channels** — Recall over a profile with only task-type memories returns no direct-vec or hyde-vec hits.
10. **HyDE helps multi-hop** — Query `"what's the cost implication?"` retrieves a memory `"enterprise tier is $10K/month"` that direct-vec misses but hyde-vec catches.
11. **Raw-message safety net** — A verbatim phrase present only in messages (not extracted as a memory) is still recallable.
12. **Temporal precompute** — Query `"how long since the April 10 incident?"` with `temporalNow = 2026-04-20` returns an answer containing "10 days".
13. **Recency tiebreaker** — Two memories tied on RRF score: the one with newer `ingestedAt` ranks first.
14. **Forgotten memories excluded** — A memory with `forgotten_at IS NOT NULL` never appears in recall results.

### Integration

15. **End-to-end ingest → recall** — Ingest the four-message setup conversation from grounding §2; `recall("What package manager does the user prefer?")` returns a synthesized string containing "pnpm".
16. **Profile isolation** — Write memory M to profile A; `recall` on profile B cannot surface M (separate SQLite file; profile-B DB has no knowledge of M).
17. **Async vectorization** — After ingest completes, vectors appear in `memories_vec` within 5 seconds (polling test).
18. **Auth enforced (shared-secret)** — REST call without Bearer token returns 401; with wrong token returns 403.
19. **Auth enforced (jwt)** — With a valid JWT for configured audience/issuer, request succeeds; with an expired JWT, returns 403.
20. **Auth bypass guard** — Starting the server with `AUTH_ADAPTER=none` and a non-loopback `HOST` refuses to start.
21. **Export round-trip** — Export NDJSON, reimport into a fresh profile via ingest + explicit remember calls; recall parity on 5 sample queries.

### Benchmark

22. **LoCoMo score** — Recorded per run; trend-line check on CI (no single-run pass/fail gate).
23. **LongMemEval score** — Same.
24. **BEAM score** — Same.

### Division-of-concerns (static)

25. **ESLint clean** — `pnpm lint` passes with zero `no-restricted-imports` violations.
26. **No cross-app imports** — Grep confirmation: `apps/server/src/` contains no imports from `apps/chat-foundation/` and vice versa.

### Whitelabel

27. **No project-specific strings** — `grep -ri` for operator-specific names, URLs, or branding in `apps/chat-foundation/src/` and `apps/server/src/` returns zero matches.
28. **Title override** — Setting `VITE_APP_TITLE="Acme Memory"` at build time replaces the default "Memory Module" header text.

---

## Accessibility Requirements

(Applies to the chat foundation only; the server has no UI.)

- Full keyboard navigation of composer, message list, and memory inspector
- ARIA roles: `role="log"` on message list with `aria-live="polite"`
- Colour cues on memory type always paired with a text label (colour is not sole indicator)
- WCAG AA contrast minimum in both themes
- Focus trap in any modal
- Screen-reader announcement on new assistant message arrival

---

## Performance Goals

### Backend (local, commodity dev machine — 8-core CPU, 16 GB RAM, SSD)

| Operation | p50 target | p99 target |
|-----------|-----------|-----------|
| `ingest` (10 messages) | < 3s   | < 8s |
| `remember`              | < 500ms | < 1.2s |
| `recall`                | < 1.5s | < 4s |
| `list` (100 items)      | < 50ms  | < 150ms |
| `forget`                | < 20ms  | < 80ms |

(Ingest and recall latencies are dominated by LLM call time and therefore depend on the chosen model provider. Targets assume a local Ollama 8B-class model or a hosted small model. Local SQLite operations themselves — list, forget, the FTS and vec queries inside recall — are sub-millisecond.)

(Ingest latency excludes vectorization, which is async.)

### Frontend (chat foundation)

- Initial load (cold): < 2s on 4G
- Interaction-to-response: bounded by chat model, not SDK; SDK overhead < 50ms per call
- Smooth 60fps scroll on 1000-message transcripts

---

## Extended Features

Out of scope for v1, but the architecture must not foreclose:

- **WAL-mode replication** for multi-reader single-writer scale-out on a single host
- **Profile archive/restore** as a single tarball (includes SQLite file + optional vector re-seed manifest)
- **Multi-tenant profile sharing** with per-memory ACLs
- **Memory versioning UI** — visual supersession chain browser
- **Model-swap tooling** — switch `ModelRegistry` providers via the chat foundation settings page without restart
- **Streaming recall** — server-sent synthesis tokens
- **Cross-profile federated recall** — query a set of profiles in a single call
- **Import from third-party memory stores** via the export schema
- **Alternative storage backends** — Postgres + pgvector adapter behind the same `Storage` / `VectorStore` interfaces
- **Alternative auth adapters** — OAuth provider adapter, session-cookie adapter

---

## Implementation Notes

1. **The companion grounding document** `CF-AgentMemory-grounding.md` describes the architectural "why" in more depth than this spec. Operators and contributors should read it once before editing pipeline code.
2. **Never let pipeline code call a specific model directly.** Always go through `ModelRegistry`. This is the single most important architectural invariant.
3. **Never let recall and ingest share utility modules.** If a helper looks shared, duplicate it. Division-of-concerns over DRY.
4. **Always prepend search queries to embedding text.** This is the single biggest lever on recall quality per the grounding document.
5. **Do not let the synthesizer do date math.** The regex + precomputed-fact path is in the pipeline for a reason.
6. **Treat task-type memories as ephemeral.** Do not back-fill them into the vector index later.
7. **Idempotency is load-bearing.** Never swap `INSERT OR IGNORE` for plain `INSERT` without very deliberate reasoning.
8. **Benchmark variance is real.** Any score change smaller than 2 points across a single run is noise. Require N=3 runs with trend analysis before concluding an improvement.
9. **Test against the mock provider first, then the real provider.** The `mock` provider in `models/providers/mock.ts` returns deterministic outputs so the pipeline logic can be tested independently of model stochasticity.
10. **Benchmark corpora are not redistributed.** The runner expects them at `./bench/corpora/{locomo,longmemeval,beam}/`. Operators must acquire them from upstream under their respective licenses.
11. **Profile files are the unit of backup.** Copying `./profiles/<name>.db` produces a complete, portable snapshot. There is no other piece of state to reconcile.
12. **One Node process, one machine, by default.** The spec is designed to be boringly simple to run. Multi-instance scale-out is an Extended Feature, not a v1 requirement.

---

## License

MIT.

---

*End of Memory Module TINS specification.*
