# MemoryModule-grounding.md

## Memory Module — Technical Grounding Document

A comprehensive reference for the architecture, API surface, pipelines, and operational semantics of Memory Module.

This document describes the *why* behind each design decision in the companion TINS specification (`MemoryModule-dev-TINS.md`). It is platform-neutral: it commits to algorithms, pipeline structure, and contracts, but not to any specific managed-cloud or vendor stack.

---

## 1. What Memory Module Is

Memory Module is a persistent memory layer for AI agents. It solves the problem of **context compaction** — the moment in an agent's lifecycle when the harness shortens the context window (to stay within model limits or to avoid context rot) and information would otherwise be permanently discarded.

Instead of discarding, Memory Module **preserves knowledge on compaction** and makes it retrievable across sessions, agents, and users. It is designed to slot into existing agent loops without modifying the agent's core logic.

### Core value proposition

- **Persistence across sessions** — memory survives restarts, compactions, and session boundaries.
- **Shared memory across agents, people, and tools** — a profile doesn't have to belong to a single agent; teams, review bots, and coding agents can share the same store.
- **Portability** — every memory is exportable. Vendor lock-in is explicitly avoided; the data belongs to the operator.
- **Complementary to file search** — Memory Module is for *context recall* (derived from sessions), not for indexing files. A document search primitive handles the latter. They are designed to interoperate.

---

## 2. The Profile Abstraction

A **profile** is the top-level container — an isolated memory store addressable by name. Profiles are the unit of isolation: in the reference standalone implementation, each profile maps to its own SQLite file holding both the relational tables (messages, memories, FTS indexes) and the vector index.

A profile exposes five operations:

| Operation | Purpose | Typical caller |
|-----------|---------|----------------|
| `ingest`  | Bulk extraction of memories from a conversation | Harness, at compaction time |
| `remember` | Store a single memory explicitly | Model, via tool use |
| `recall`  | Run the retrieval pipeline, return a synthesized answer | Model or application |
| `list`    | Enumerate stored memories | Model, via tool use |
| `forget`  | Mark a specific memory as no longer true/important | Model, via tool use |

### Canonical usage pattern (SDK)

```ts
import { MemoryClient } from "@memory-module/sdk";

const client = new MemoryClient({
  baseUrl: "http://127.0.0.1:8787",
  getAuthToken: async () => process.env.MEMORY_TOKEN!,
});

const profile = await client.getProfile("my-project");

// Bulk: extract memories from a conversation (typically at compaction)
await profile.ingest([
  { role: "user",      content: "Set up the project with React and TypeScript." },
  { role: "assistant", content: "Done. Scaffolded a React + TS project." },
  { role: "user",      content: "Use pnpm, not npm. And dark mode by default." },
  { role: "assistant", content: "Got it — pnpm and dark mode as default." },
], { sessionId: "session-001" });

// Targeted: store a single memory on the spot
const memory = await profile.remember({
  content: "API rate limit was raised to 10,000 req/s per zone after the April 10 incident.",
  sessionId: "session-001",
});

// Retrieve a synthesized answer
const results = await profile.recall("What package manager does the user prefer?");
console.log(results.result); // "The user prefers pnpm over npm."
```

### Access surfaces

- **SDK** (`@memory-module/sdk`) — the primary, first-class access path for any host chat system or agent harness. A thin HTTP client that exposes the `Profile` surface.
- **REST API** — for agents, tools, or scripts that prefer to hit the server directly. Every operation has a corresponding endpoint under `/v1/profiles/:name/…`.
- **Model tool-use surface** — four tools (`recall`, `remember`, `forget`, `list`) that a chat model can call during a session. `ingest` is deliberately *not* exposed as a model-facing tool — it's a harness operation.

---

## 3. Architectural Overview

Memory Module is a single coordinator process that fronts three backing primitives: persistent storage, a vector index, and a pluggable set of model providers. A caller invokes it through the SDK or REST; the coordinator dispatches to the right profile's storage, to its vector index, and to the configured model providers for LLM and embedding calls.

```
+------------------+      SDK / REST      +---------------------------------------+
|  Caller          |                      |  Memory Module Server                 |
|  (host chat /    |--------------------->|                                       |
|   agent harness) |                      |    Profile Manager                    |
+------------------+                      |           |                           |
                                          |           v                           |
                                          |    +-----------------+                |
                                          |    | Profile         |                |
                                          |    | (per-name)      |                |
                                          |    +-----------------+                |
                                          |           |                           |
                                          |    +------+------+------+             |
                                          |    v             v      v             |
                                          |  +-------+   +-------+ +-----------+  |
                                          |  | SQLite|   |Vector | | Model     |  |
                                          |  | (per  |   |index  | | Registry  |  |
                                          |  | prof) |   |(same  | |(pluggable |  |
                                          |  |       |   | file) | | providers)|  |
                                          |  +-------+   +-------+ +-----------+  |
                                          +---------------------------------------+
```

### Component responsibilities

| Component | Role |
|-----------|------|
| **Server** | Stateless HTTP layer. Routes requests, validates inputs, orchestrates ingest/recall pipelines, invokes model providers. |
| **Profile Manager** | Lazily resolves named profiles to backing resources. Caches `Profile` instances in memory. |
| **Profile (per name)** | Owns the SQLite handle and the per-profile write mutex. Handles transactional writes, FTS indexing, and supersession chains. |
| **Vector Index (per profile)** | Vector index over embedded memories. Co-located with the profile's relational storage for tenant isolation. |
| **Model Registry** | Abstracts every LLM and embedding call. Pipeline code references *roles* (extractor, verifier, classifier, query analyzer, synthesizer, embedder) — never specific providers. Providers are swappable via configuration. |

### Why this shape

- **Tenant isolation by default** — one profile = one storage unit. Memories from one profile cannot accidentally surface in another.
- **Purpose-built storage per workload** — a relational store with FTS for transactional, queryable memory state; a vector index for ANN search; file-based snapshots for portability. Nothing forced into a single shape.
- **Addressability by name** — any request, from anywhere, reaches the right profile through its name. No profile ID registry, no provisioning step.
- **Platform neutrality** — the same pipeline code runs on a developer laptop, a single production server, a container orchestrator, or any managed-cloud composition of equivalent primitives. The SDK and REST contracts don't change.

---

## 4. The Ingestion Pipeline

Ingestion is invoked in bulk, typically at compaction. The conversation flows through a multi-stage pipeline: ID generation → parallel extraction (full pass + detail pass) → merge → verification → classification → storage write → asynchronous vectorization.

### Stage 1 — Deterministic ID Generation

Each message gets a **content-addressed ID**: `SHA-256(sessionId || role || content)`, truncated to 128 bits. Re-ingesting the same conversation produces identical IDs, so re-ingestion is **idempotent** (this later combines with `INSERT OR IGNORE` at the storage layer).

### Stage 2 — Parallel Extraction

Two passes run concurrently:

**Full Pass.** Chunks messages at roughly 10K characters with a **two-message overlap**, and processes up to four chunks concurrently. Each chunk is given to the extractor as a structured transcript with:

- role labels on every turn,
- relative dates resolved to absolutes (e.g. "yesterday" → `2026-04-14`),
- line indices for source provenance.

**Detail Pass.** Runs alongside the full pass for longer conversations (9+ messages). Uses overlapping windows tuned to surface concrete values the broad pass tends to smooth over: names, prices, version numbers, entity attributes.

The two result sets are then **merged**.

### Stage 3 — Verification

Every extracted memory candidate is checked against the source transcript through **eight verification checks**:

1. Entity identity
2. Object identity
3. Location context
4. Temporal accuracy
5. Organizational context
6. Completeness
7. Relational context
8. Whether inferred facts are actually supported by the conversation

Each candidate is then **passed, corrected, or dropped**.

### Stage 4 — Classification

Verified memories are classified into one of four types:

| Type | Meaning | Indexing properties |
|------|---------|---------------------|
| **Fact** | Atomic, stable, currently-true knowledge. E.g. "the project uses GraphQL", "the user prefers dark mode." | **Keyed** + supersedes prior versions. Vector-indexed. |
| **Event** | Something that happened at a specific time. E.g. a deployment, a decision. | Time-anchored. Vector-indexed. |
| **Instruction** | How-to knowledge — procedures, workflows, runbooks. | **Keyed** + supersedes prior versions. Vector-indexed. |
| **Task** | Ephemeral in-flight work item. | **Excluded from vector index** to keep it lean. Still discoverable via FTS. |

**Supersession chains.** For keyed types (facts and instructions), an incoming memory with a matching normalized topic key **supersedes** the prior version rather than deleting it. A forward pointer is written from the old memory to the new, preserving the version history. This means a recall can trace "what did we used to think?" if needed, but will surface the current value by default.

### Stage 5 — Storage

Writes use `INSERT OR IGNORE`. Combined with content-addressed IDs, this silently skips exact duplicates. The response is returned to the harness **before** vectorization runs — vectorization is asynchronous and does not block the ingest call.

### Stage 6 — Asynchronous Vectorization

After the response returns, the pipeline:

1. Generates the embedding text for each new memory by **prepending 3–5 search queries** (produced during classification) to the memory content. This is the key trick for bridging the gap between how memories are *written* (declaratively: "user prefers dark mode") and how they are *searched* (interrogatively: "what theme does the user want?"). Without this, the embedding is biased toward the answer's vocabulary and misses questions phrased differently.
2. Upserts new vectors into the profile's vector index.
3. Deletes vectors for any superseded memories in parallel with the upserts.

### Pipeline diagram

```
              +------------------------+
              |  Conversation Input    |
              +-----------+------------+
                          v
              +------------------------+
              |  ID Generation         |
              |  SHA-256 content-addr. |
              +-----------+------------+
                          v
              +----------+----------+
              v                     v
       +-------------+       +-------------+
       |  Full Pass  |       | Detail Pass |
       | 10K chunks, |       | overlapping |
       | concurrent  |       | windows for |
       |             |       | concrete    |
       |             |       | values      |
       +------+------+       +------+------+
              +--------+   +--------+
                       v   v
                   +---merge---+
                       |
                       v
              +--------------------+
              | Verification       |
              | 8 checks ->        |
              | pass/correct/drop  |
              +----------+---------+
                         v
              +--------------------+
              |   Classification   |
              +----------+---------+
                         |
          +------+-------+-------+------+
          v      v       v       v
        Fact   Event  Instruction  Task
       (keyed) (time) (keyed)    (no vec)
        supersedes   supersedes   index
          |      |       |       |
          +------+---+---+-------+
                    v
         +----------------------+
         |   Storage Write      |
         |   INSERT OR IGNORE   |
         +----------+-----------+
                    | async
                    v
         +----------------------+
         |    Vector Upsert     |
         | search queries pre-  |
         | pended to content;   |
         | superseded vectors   |
         | pruned in parallel   |
         +----------------------+
```

---

## 5. The Retrieval Pipeline

Retrieval runs when the agent calls `recall(query)`. No single retrieval method works well across all query shapes, so Memory Module runs five channels in parallel and fuses them with Reciprocal Rank Fusion.

### Stage 1 — Query Analysis + Embedding (concurrent)

**Query analysis** produces:

- **Ranked topic keys** — used to drive fact-key lookup.
- **FTS terms with synonyms** — used for the full-text channel.
- **A HyDE (Hypothetical Document Embedding)** — a *declarative* statement phrased as if it were the answer to the question. Embedded and used in the HyDE vector channel.

**Embedding** embeds the raw query directly (this vector feeds the direct vector channel).

Both embeddings go downstream.

### Stage 2 — Five Parallel Retrieval Channels

| Channel | What it's good at | Weight (RRF) |
|---------|-------------------|--------------|
| **Fact-Key Lookup** | Exact topic match — strongest signal, zero ambiguity | **Highest** |
| **Full-Text Search** (Porter stemming) | Keyword precision when you know the exact term but not the context | Mid-high |
| **HyDE Vector Search** | Semantic similarity between "what the answer looks like" and stored memories — wins on abstract or multi-hop queries where question and answer use different vocabulary | Mid |
| **Direct Vector Search** | Semantic similarity from the raw query — the general-purpose ANN path | Mid |
| **Raw Message Search** (FTS over stored turns) | Safety net for verbatim details the extraction pipeline may have generalized away | **Lowest** |

### Stage 3 — Reciprocal Rank Fusion (RRF)

Each candidate receives a **weighted score** based on where it ranked within each channel. The highest-weighted channel is fact-key (exact topic match is the most reliable signal); raw-message matches are included at low weight as a fallback. **Ties are broken by recency** — newer results ranked higher.

### Stage 4 — Synthesis

The top candidates are passed to a synthesis model that produces a **natural-language answer** to the original query.

### Deterministic overrides

Some query types bypass the LLM. Notably, **temporal computation** (date math) is handled with regex + arithmetic, and the precomputed results are injected into the synthesis prompt as facts. Models are unreliable at date math — so they're not asked to do it.

### Retrieval pipeline diagram

```
                      +----------------+
                      |  Search Query  |
                      +-------+--------+
                              |
                +-------------+-------------+
                v                           v
       +-----------------+         +-----------------+
       | Query Analysis  |         |   Embedding     |
       | topic keys, FTS |         | raw query -> vec|
       | + synonyms,     |         |                 |
       | HyDE answer     |         |                 |
       +--------+--------+         +--------+--------+
                +--------------+------------+
                               v
                  +--------------------------+
                  |    Parallel Retrieval    |
                  +------------+-------------+
                               |
   +-----------+---------------+---------------+--------------+
   v           v               v               v              v
+------+   +------+        +--------+      +--------+     +------+
| FTS  |   | Fact |        | Direct |      |  HyDE  |     | Raw  |
|Porter|   | Key  |        | Vector |      | Vector |     | Msgs |
|      |   |Lookup|        | Search |      | Search |     |      |
+---+--+   +--+---+        +---+----+      +---+----+     +--+---+
    +---------+----------------+----------------+------------+
                               v
                  +--------------------------+
                  |  Reciprocal Rank Fusion  |
                  |  fact-key highest,       |
                  |  raw-msgs lowest;        |
                  |  ties broken by recency  |
                  +------------+-------------+
                               v
                  +--------------------------+
                  |        Synthesis         |
                  |  natural-language answer |
                  |  from ranked candidates  |
                  +------------+-------------+
                               v
                  +--------------------------+
                  |      Search Results      |
                  +--------------------------+
```

---

## 6. Memory Types — Detailed Semantics

### Fact

- **Shape:** atomic, currently-true assertion.
- **Storage:** row in the memories table with a normalized topic key.
- **Update semantics:** if a new fact arrives with the same key, the old fact is marked superseded and a forward pointer is written. The new fact becomes the canonical answer for that key.
- **Retrieval:** surfaces via fact-key lookup (highest RRF weight), FTS, and vector search.
- **Example:** `topic_key: "package_manager_preference"` → `"The user prefers pnpm over npm."`

### Event

- **Shape:** something that happened at a specific time.
- **Storage:** row in the memories table, time-anchored, no key (events don't supersede each other).
- **Retrieval:** FTS + vector search. Temporal queries get deterministic date-math treatment at synthesis.
- **Example:** `"API rate limit was increased to 10,000 req/s per zone after the April 10 incident."`

### Instruction

- **Shape:** procedure, workflow, runbook — how to do something.
- **Storage:** row in the memories table with a normalized topic key.
- **Update semantics:** supersedes prior versions with the same key (same as facts).
- **Retrieval:** same channels as facts.
- **Example:** `topic_key: "deploy_to_staging"` → step-by-step instructions.

### Task

- **Shape:** ephemeral in-flight work item.
- **Storage:** row in the memories table, **excluded from vector index** to keep the index lean.
- **Retrieval:** FTS + raw-message search only. No vector channel.
- **Intended lifecycle:** surfaced via `list`, marked via `forget` when completed.

---

## 7. Model-Facing Tool Surface

The model interacts directly with memory through a small, deliberately constrained set of tools. The design principle is that **the primary agent should never burn context on storage strategy** — no query design, no index management, no retrieval tuning exposed to the model.

| Tool | What the model uses it for |
|------|----------------------------|
| `recall(query)` | Retrieve information. Runs the full five-channel + RRF + synthesis pipeline. |
| `remember(content)` | Store something important encountered mid-task. |
| `forget(memoryId)` | Mark a memory as no longer true or relevant. |
| `list()` | See what's currently stored. |

Ingestion (`ingest`) is **not** a model-facing tool — it's a harness operation invoked at compaction boundaries. This separation keeps the bulk extraction pipeline out of the model's tool loop.

---

## 8. Integration Patterns

### Pattern A — Memory for individual agents

Works transparently with any agent harness that supports model tool-use. Typical shapes include human-in-the-loop coding agents, self-hosted agent frameworks, and managed agent services. No changes to the agent's core loop: the harness is configured to call `ingest` at compaction and to expose the four model-facing tools.

### Pattern B — Custom agent harnesses

For teams building their own infrastructure — autonomous background agents, batch-review pipelines, long-running research agents — Memory Module gives those agents durable memory that **survives restarts** and persists across sessions. The harness owns when and what to ingest; Memory Module owns how to store, index, and retrieve.

### Pattern C — Shared memory across agents, people, and tools

Profiles are not bound to a single principal. Useful shapes:

- **Team profile** — a whole engineering team's coding agents share a profile. Conventions, architectural decisions, and tribal knowledge accumulate centrally instead of being repeatedly re-explained or lost.
- **Cross-agent profile** — a code review bot and a coding agent share a profile so review feedback shapes future code generation.
- **Human + agent profile** — a human tool that writes via `remember` and agents that read via `recall` share the same store.

The result: knowledge that would otherwise be ephemeral becomes a durable team asset.

### Relation to document search

Memory Module and document-search systems solve **distinct problems** and compose:

- **Document search** — finds results across unstructured and structured **files**. A document/object search primitive.
- **Memory Module** — surfaces **context derived from sessions**. Not file-backed.

An agent can use both, and they're designed to interoperate.

---

## 9. Data Ownership & Portability

A first-class commitment: the data belongs to the operator, not the infrastructure.

- Every memory is **exportable** via `profile.export()` as JSON or NDJSON.
- The reference implementation stores each profile in a single file, which is itself a complete portable snapshot — copy the file, you have the profile.
- The export schema is documented and stable; importing into an alternative store that honours the schema is straightforward.

The principle: make leaving easy, and keep building something good enough that operators don't want to leave. As agents embed further into business processes, the memory they accumulate becomes genuine institutional knowledge — tying that knowledge to a single proprietary store is a material risk.

---

## 10. Development & Tuning Methodology

A deliberate methodology for evolving the pipeline safely, without overfitting to any one benchmark or workload.

1. **Start from a working baseline.** Run the existing pipeline against the full benchmark suite to establish a reference score per suite.
2. **Change one thing at a time.** Changing prompts, weights, and chunking sizes simultaneously makes improvements impossible to attribute.
3. **Human review of proposed changes.** When an agent or developer proposes a change, review it for *generality* before acceptance: would this change also help on queries shaped differently than the benchmark that motivated it? Changes that only help one benchmark are usually overfitting.
4. **Multi-suite benchmarking.** Test against **LoCoMo**, **LongMemEval**, and **BEAM** — different benchmarks stress the system in different ways, reducing the risk of overfitting any one of them. Each has its own failure modes worth reading about before using the scores.
5. **Iterate.** Rerun the suite, compare, keep what generalizes, discard what doesn't.

### The stochasticity problem

Even with `temperature=0`, LLM calls are not perfectly deterministic. This makes benchmark scores noisy across runs. Mitigations:

- Average over multiple runs (N=3 minimum for any conclusion about an improvement; N=5 for small suites where variance is wider).
- Rely on **trend analysis** alongside raw scores to judge whether a change actually helped — a 1-point single-run bump is almost certainly noise.
- Guard carefully against overfitting to benchmark specifics (e.g. memorising the exact phrasing of benchmark questions in a prompt) in ways that wouldn't generalize to real workloads.

### Mock-first testing

The reference implementation ships a `mock` model provider that returns deterministic outputs. Running the pipeline against `mock` isolates pipeline-logic bugs from model-stochasticity noise; running against real providers then validates the end-to-end behaviour. Tests should live at both layers.

---

## 11. Key Design Decisions & Their Rationale

| Decision | Why |
|----------|-----|
| **SHA-256 content-addressed IDs** | Idempotent re-ingestion. Safe to retry or replay without duplication. |
| **Full pass + detail pass in parallel** | Broad extraction smooths over concrete values (numbers, names, versions); the detail pass recovers them. |
| **Eight-check verification** | Extraction alone hallucinates or over-infers; verification filters or corrects before anything is stored. |
| **Four memory types, not one** | Different shapes need different indexing (keyed vs time-anchored vs ephemeral) and different supersession rules. A single generic type would force compromises. |
| **Supersession chains, not deletion** | Preserves history for auditing and "what did we used to believe" queries while keeping the current answer canonical. |
| **Tasks excluded from vector index** | Ephemeral items would bloat the index without helping recall quality. FTS is enough for task-shaped queries. |
| **Async vectorization** | Keeps `ingest` latency low. The response returns once the storage write commits. |
| **Prepending search queries to embedding text** | Bridges the declarative/interrogative vocabulary gap. The single biggest lever on recall quality. |
| **Five-channel parallel retrieval + RRF** | No single method wins across query shapes; fusion is a principled way to combine complementary signals. |
| **Deterministic date math** | LLMs are unreliable at arithmetic on dates. Compute it, inject the result into the synthesis prompt as a fact. |
| **One storage unit per profile** | Strong tenant isolation, automatic addressing by name, no provisioning step. |
| **Model-agnostic binding** | Pipeline code references roles (extractor, verifier, classifier, synthesizer, embedder), not providers. Swap any provider by editing configuration. |
| **Prompt-cache reuse where supported** | When a model provider supports prompt caching, repeated calls against the same profile should route consistently to benefit from it. The pipeline exposes a stable session-identity handle to providers that can use it. |

---

## 12. Glossary

- **Profile** — named, isolated memory store. One backing storage unit + one vector index per profile.
- **Compaction** — the harness operation that shortens an agent's context window; the canonical trigger for `ingest`.
- **Ingest** — bulk extraction of memories from a conversation into a profile.
- **Recall** — retrieval + synthesis; returns a natural-language answer.
- **Remember / Forget / List** — model-facing tools for targeted memory operations.
- **Fact / Event / Instruction / Task** — the four classified memory types.
- **Keyed memory** — a memory with a normalized topic key (facts and instructions); new versions supersede old ones.
- **Supersession** — replacing an old keyed memory with a new one, via forward pointer, without deletion.
- **HyDE (Hypothetical Document Embedding)** — a declarative statement phrased as the *answer* to the query, embedded and used as a retrieval signal.
- **RRF (Reciprocal Rank Fusion)** — fusion method combining multiple ranked retrieval channels into a single weighted ranking.
- **Content-addressed ID** — `SHA-256(sessionId || role || content)`, truncated to 128 bits. Ensures idempotent re-ingestion.
- **Model Registry** — the abstraction layer that resolves pipeline *roles* to concrete providers. The sole place provider identity lives.
- **Chat Adapter** — the interface by which the reference chat foundation talks to a chat model. Swap adapters to change which model powers the reference UI without touching any pipeline code.

---

*End of grounding document.*
