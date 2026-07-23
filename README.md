# enclave-rag

A fully-private, self-hosted **RAG** (retrieval-augmented generation) stack for a small
team. CPU-only, runs via Docker Compose, **no data leaves the box**. Every model is
Apache-2.0/MIT (Open WebUI is the one source-available exception).

**Status: Phase 0 (platform prep).** The off-the-shelf services (Ollama, Qdrant, Open WebUI)
are wired in [docker-compose.yml](docker-compose.yml); the application services (`rag-api`,
`ingestion-worker`) are stubbed and land in Phase 3. See progress below.

- **[MASTER-PLAN.md](MASTER-PLAN.md)** — the design: hardware, chosen stack, architecture, build phases, performance expectations.
- **[PROGRESS.md](PROGRESS.md)** — the task tracker; current state and what's next. Every task has a verification step.

## Architecture

```
Open WebUI ──▶ RAG API (FastAPI) ──▶ Ollama        (generation)
                     │
                     └────────────▶ Qdrant         (hybrid dense+sparse retrieval)

Ingestion worker ──▶ Docling parse ──▶ chunk ──▶ embed ──▶ upsert to Qdrant (dedup by doc hash)
```

Query flow: embed → Qdrant top-20 → (Phase 2: rerank to top-5) → citation-forcing prompt → streamed answer with sources.

| Service | Port (localhost-only) | Role |
|---|---|---|
| ollama | `11434` | LLM runtime + embeddings (OpenAI-compatible) |
| qdrant | `6333` REST / `6334` gRPC | vector DB |
| open-webui | `3000` | chat UI |
| rag-api | `8000` | retrieval + answer API *(stubbed until Phase 3)* |
| ingestion-worker | — | watch folder / upload → parse → embed → upsert *(stubbed until Phase 3)* |

All ports bind `127.0.0.1` only. External exposure is hardened to Open-WebUI-behind-TLS in Phase 4/7.

## Prerequisites

- **Docker Engine + Compose v2** (`docker compose version` ≥ v2).
- **AVX2 CPU** (`lscpu | grep -i avx2` returns a match) — required for CPU inference.
- A host data directory at `${DATA_DIR:-/data/rag}` with subdirs `models/ qdrant/ corpus/ snapshots/ open-webui/`, writable by uid 1000. The Qdrant index is disposable (rebuildable from corpus); the corpus is not.

## The two-environment model

**One [docker-compose.yml](docker-compose.yml), two env-file overlays.** Dev and prod share an
**identical service graph**; they differ **only** in env values (model, embedder, threads, volume
source, `mem_limit`s). Differences are parameterized through `.env` / `.env.prod` — a service is
never added to one environment and not the other.

> Note: this uses an **env-file overlay**, not Compose `--profile`. `--profile` gates *which
> services run*, not env *values*, so it can't express the model/threads/`mem_limit` deltas.

| Setting | dev (`.env`, this VM) | prod (`.env.prod`, R360) |
|---|---|---|
| Generation model | `qwen3:4b` | `qwen3:8b` |
| Embedder | `nomic-embed-text` | `bge-m3` |
| Reranker | off | on (`bge-reranker-v2-m3`) |
| Ollama threads | 4 | 8 |
| `mem_limit`s | sized for 15GB, no swap | sized for 32GB |
| `/data/rag` | plain dir | LVM mount |

## Quick start (dev)

From a fresh clone on the dev VM:

```bash
git clone <repo-url> enclave-rag && cd enclave-rag

# 1. Validate the compose file + env substitution (no containers started)
docker compose config          # dev; auto-loads .env

# 2. Bring up the off-the-shelf services (dev profile = auto-loaded .env)
docker compose up -d
docker compose ps              # all should be running/healthy

# 3. Health checks
curl localhost:11434/api/tags  # Ollama up (200)
curl localhost:6333/healthz    # Qdrant up

# 4. Pull the dev models (≤4B — an 8B + full stack OOMs 15GB)
docker exec rag-ollama ollama pull qwen3:4b           # generation
docker exec rag-ollama ollama pull nomic-embed-text   # embeddings

# 5. Open the chat UI
#    http://localhost:3000
```

Tear down with `docker compose down` (add `-v` only if you also want to drop named volumes;
the host data at `/data/rag` is untouched either way).

## Running the prod overlay (R360)

`--env-file` **replaces** the default `.env`, so `.env.prod` is self-contained:

```bash
docker compose --env-file .env.prod config     # validate
docker compose --env-file .env.prod up -d       # bring up with prod values
```

Full R360 bring-up (LVM storage, prod models, TLS, performance gate) is Part B, Phases 6–9 in [PROGRESS.md](PROGRESS.md).

## Common operations

```bash
docker compose logs -f <service>    # tail a service (ollama | qdrant | open-webui)
docker compose restart <service>
docker compose down && docker compose up -d   # recreate
```

## Repo layout

```
docker-compose.yml   one file, both environments (env-driven)
.env                 dev overlay (this VM: 4 vCPU / 15GB / no swap)
.env.prod            prod overlay (R360: 8C/16T / 32GB / LVM)
MASTER-PLAN.md       design & rationale
PROGRESS.md          verification-gated task tracker
R360-hw-spec.txt     raw lshw dump backing MASTER-PLAN.md §1
CLAUDE.md            guidance for Claude Code in this repo
services/            app service build contexts (rag-api, ingestion) — added in Phase 3
```

## Conventions

- **Verification-gated progress.** A [PROGRESS.md](PROGRESS.md) box is checked only when its `_Verify:_` step passes; a phase is "Done" only when its gate criteria are met.
- **Performance numbers on the dev VM are meaningless** (4 shared vCPUs). The prefill/generation benchmark and the interactive-vs-async decision run only on the R360 (Part B, Phase 8).
- Record irreversible/expensive choices in PROGRESS.md's **Cross-phase decisions log**.
