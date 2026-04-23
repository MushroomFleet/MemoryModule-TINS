# Memory Module (TINS)

**A model-agnostic, platform-agnostic agent memory harness — distributed as TINS specifications, not source code.**

Memory Module is a persistent memory layer for AI agents. It preserves what an agent learns across sessions, restarts, and context compactions, and makes that knowledge retrievable through a five-channel retrieval pipeline with Reciprocal Rank Fusion, HyDE-augmented embedding, supersession chains, and deterministic date math.

This repository does not ship source code. It ships two TINS-compliant Markdown specifications detailed enough for a capable coding assistant to generate a complete, working implementation on demand.

---

## What's in this repository

| File | Purpose |
|------|---------|
| [`MemoryModule-dev-TINS.md`](./MemoryModule-dev-TINS.md) | The implementation specification. Standalone Node.js + SQLite + Vite/React. Runs locally, deploys anywhere Node runs. Whitelabel-friendly. |
| [`MemoryModule-grounding.md`](./MemoryModule-grounding.md) | The architectural grounding document. Describes *why* each pipeline decision was made. Read it before editing pipeline code. |

Pick **one** of the TINS files depending on where you want to run the system, then generate an implementation from it.

---

## What is TINS?

[TINS (There Is No Source)](https://thereisnosource.com) is a distribution paradigm: software is shipped as a sufficiently-detailed README, and an AI model generates the implementation on demand. The same README produces better code as models improve.

For Memory Module this means:

- The specification is the canonical artifact.
- You can regenerate the code at any time with a more capable model.
- The distribution is tiny — no build artifacts, no lockfiles, no vendored dependencies.
- Platform independence is built in — the spec is prescriptive about behaviour and loose about incidentals.

---

## What Memory Module does

Memory Module solves the problem of **context compaction**: the moment in an agent's lifecycle when the harness shortens the context window and information would otherwise be permanently discarded. Instead of discarding, Memory Module preserves that knowledge and makes it retrievable across sessions, agents, and users.

### Five operations

| Operation | Purpose | Typical caller |
|-----------|---------|----------------|
| `ingest`   | Bulk extraction of memories from a conversation | Harness, at compaction time |
| `remember` | Store a single memory explicitly | Model, via tool use |
| `recall`   | Run the retrieval pipeline, return a synthesized natural-language answer | Model or application |
| `list`     | Enumerate stored memories | Model, via tool use |
| `forget`   | Mark a memory as no longer true or important | Model, via tool use |

### Four memory types

Extracted memories are classified into one of four types, each with different indexing and supersession rules:

- **Fact** — atomic, currently-true assertion. Keyed; new versions supersede old ones.
- **Event** — something that happened at a specific time. Time-anchored.
- **Instruction** — how-to knowledge, runbooks, procedures. Keyed; supersedes prior versions.
- **Task** — ephemeral in-flight work item. Excluded from the vector index to keep it lean.

### Five-channel retrieval

Every `recall` query runs five channels in parallel and fuses them with Reciprocal Rank Fusion:

1. **Fact-Key Lookup** — exact topic match (highest weight)
2. **Full-Text Search** — Porter-stemmed keyword match
3. **HyDE Vector Search** — semantic match against a hypothetical-answer embedding
4. **Direct Vector Search** — semantic match against the raw query embedding
5. **Raw Message Search** — FTS over stored turns, lowest weight, safety net

Ties are broken by recency. Temporal questions bypass the LLM and are answered with deterministic date arithmetic injected into the synthesis prompt as precomputed facts.

For the full pipeline, schemas, and rationale, read [`CF-AgentMemory-grounding.md`](./CF-AgentMemory-grounding.md).

---

## Architecture at a glance (standalone build)

```
+-------------------------------------------------+
|           Host Chat System (untouched)          |
+----------------------+--------------------------+
                       | @memory-module/sdk
                       v
+-------------------------------------------------+
|        Node.js Server (Fastify + pipelines)     |
|                                                 |
|  +--------------+   +--------------+            |
|  |  Ingestion   |   |  Retrieval   |            |
|  |  pipeline    |   |  pipeline    |            |
|  +------+-------+   +------+-------+            |
|         \                  /                    |
|          v                v                     |
|    +--------------------------+                 |
|    |  SQLite per profile      |                 |
|    |  + sqlite-vec + FTS5     |                 |
|    +--------------------------+                 |
|                                                 |
|  Pluggable ModelRegistry (OpenAI-compat,        |
|  Anthropic, Ollama, LM Studio, vLLM, mock)      |
|                                                 |
|  Pluggable AuthAdapter (shared-secret, JWT,     |
|  or loopback-only open mode)                    |
+-------------------------------------------------+

         Optional: Vite/React chat foundation
         (reference integration surface, white-label ready)
```

**Key properties:**

- **One SQLite file per profile** — strong tenant isolation, trivial backup, zero provisioning.
- **Model-agnostic** — every LLM and embedding touchpoint resolves through a `ModelRegistry`. Swap providers by editing config.
- **Platform-agnostic** — runs on any host that runs Node 20+.
- **Whitelabel-friendly** — no branding, no project-specific strings, overridable title.
- **Division of concerns enforced by ESLint** — ingest and recall pipelines cannot import from each other; chat foundation cannot import from the server; everything goes through the SDK.

---

## Quick start — generating an implementation

You will need a coding assistant capable of reading long specifications and writing a monorepo from them — e.g. Claude Code, OpenCode, or any equivalent agent harness.

### 1. Clone the repository

```bash
git clone https://github.com/MushroomFleet/MemoryModule-TINS.git
cd MemoryModule-TINS
```

### 2. Point your coding assistant at the specs

Open a session with your coding assistant in the repository root and give it:

```
Read CF-AgentMemory-grounding.md first, then MemoryModule-dev-TINS.md.
Generate a complete implementation in this directory, following the repository
layout and all invariants in the spec.
```

### 3. Install and run

```bash
pnpm install
cp apps/server/.env.example apps/server/.env
pnpm dev
```

Server defaults to `http://127.0.0.1:8787`; chat foundation to `http://127.0.0.1:5173`.

### Prerequisites for the generated implementation

- Node.js ≥ 20 LTS
- pnpm ≥ 9
- A C toolchain (for `better-sqlite3` and `sqlite-vec`):
  - macOS: Xcode Command Line Tools
  - Debian/Ubuntu: `build-essential`, `python3`
  - Windows: Visual Studio Build Tools
- At least one chat model source (OpenAI-compatible endpoint, Anthropic, Ollama, LM Studio, vLLM, LocalAI)
- At least one embedding model source (same list as above)

No cloud accounts are required. If you prefer the Cloudflare stack instead, use `MemoryModule-TINS.md` and follow the platform-specific prerequisites listed inside that document.

---

## Using Memory Module from a host chat system

Once an implementation has been generated, integration is a thin SDK import:

```ts
import { MemoryClient, buildMemoryTools } from "@memory-module/sdk";

const client = new MemoryClient({
  baseUrl: "http://127.0.0.1:8787",
  getAuthToken: async () => process.env.MEMORY_TOKEN!,
});

const profile = await client.getProfile("my-project");

// At compaction time:
await profile.ingest(messagesToCompact, { sessionId: "session-001" });

// Expose the four model-facing tools:
const tools = buildMemoryTools(profile);
// → recall, remember, forget, list

// Or call recall directly:
const { result } = await profile.recall("what package manager do we use?");
console.log(result); // "You use pnpm."
```

---

## Why these design choices

A few invariants matter more than anything else. They are enforced in the spec and should be preserved in any implementation:

- **Pipeline code never calls a specific model directly** — always through `ModelRegistry`.
- **Ingest and recall pipelines never share utility modules** — division of concerns over DRY.
- **Search queries are always prepended to embedding text** — the single biggest lever on recall quality.
- **The synthesizer never does date math** — the deterministic regex + precomputed-fact path exists for a reason.
- **Task-type memories are ephemeral** — never back-filled into the vector index.
- **`INSERT OR IGNORE` is load-bearing** — idempotent re-ingestion depends on it.

Full rationale for each: see the grounding document.

---

## Portability

A first-class guarantee:

- Every memory is exportable via `profile.export()` as JSON or NDJSON.
- Profile SQLite files are the unit of backup — copy the file, you have a complete portable snapshot.
- The export schema is documented and stable; importing into an alternative store that honours the schema is straightforward.

Your data is yours. Leaving is easy by design.

---

## Benchmarks

The spec includes a benchmark runner targeting **LoCoMo**, **LongMemEval**, and **BEAM**. These corpora are not redistributed — operators fetch them from their upstream sources under their respective licenses and drop them into `./bench/corpora/`. Benchmark variance is real; the spec mandates N=3 runs with trend analysis before concluding any improvement.

---

## License

MIT — see `LICENSE` inside the generated implementation. The specifications in this repository are released under the same terms.

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{memory_module_tins,
  title = {Memory Module: a model-agnostic, platform-agnostic agent memory harness distributed as TINS specifications},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/MemoryModule-TINS},
  version = {1.0.0}
}
```

### Donate

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
