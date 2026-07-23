# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fully-private, self-hosted RAG (retrieval-augmented generation) stack for a small team,
running CPU-only via Docker Compose. No data leaves the box; every model is Apache-2.0/MIT
(Open WebUI is the one source-available exception). The project is **early** — currently in
Phase 0. The application services (`rag-api`, `ingestion-worker`) are not written yet; only
the compose skeleton for the off-the-shelf services exists.

Two documents are the source of truth and must stay authoritative — read them before acting:

- **[PLAN.md](PLAN.md)** — the design: hardware assessment, chosen stack, architecture,
  build phases, performance expectations, GPU/RAM upgrade analysis. The "why" behind every choice.
- **[PROGRESS.md](PROGRESS.md)** — the task tracker. Every task has a `_Verify:_` line; a box is
  checked **only** when that verification passes, and a phase is "Done" only when its gate is met.
  This is where you find current state and what to do next. **Keep it updated as work lands.**

`R360-hw-spec.txt` is the raw `lshw` dump backing PLAN.md §1 (the deploy target's hardware).

## The two-environment model (most important thing to understand)

Work happens on a **dev VM** (this box: 4 vCPU / 15GB / no swap) and deploys to the **R360**
(Xeon 8C/16T / 32GB / LVM). PROGRESS.md splits into **Part A (dev, Phases 0–5)** — build and
*functionally* validate — and **Part B (R360, Phases 6–9)** — run the performance gate and harden.

**Central invariant:** one `docker-compose.yml`, two profiles. Dev and prod differ **only** in
env vars, `mem_limit`s, model, embedder, threads, and volume source — **never in service topology.**
That invariant is what makes "works on dev" mean "same graph on prod." Do not add a service to one
environment and not the other; parameterize differences through `.env` / the prod overlay instead.

Consequences to respect on the **dev VM**:
- **15GB, no swap → hard memory ceiling.** Use a ≤4B model (`qwen3:4b`); never run reranker + LLM +
  embedder together. `mem_limit`s are hard caps so a runaway container OOM-kills itself, not the host.
- **Performance numbers here are meaningless** (4 shared vCPUs). The prefill/generation benchmark and
  the interactive-vs-async viability decision live in **Part B / Phase 8** and run only on the R360.

dev→prod deltas (from PROGRESS.md): model `qwen3:4b`→`qwen3:8b`, embedder `nomic-embed-text`→`bge-m3`,
reranker off→on, threads 4→8, `/data/rag` plain dir→LVM mount, mem_limits sized 15GB→32GB.

## Architecture

Request path: **Open WebUI → FastAPI RAG API → Ollama** for generation, with the RAG API calling
**Qdrant** (hybrid dense+sparse search) for retrieval. An **ingestion worker** watches a folder /
exposes an upload endpoint, parses documents with **Docling**, chunks (~500–800 tokens, 15% overlap,
headings kept in metadata), embeds, and upserts to Qdrant with a doc hash for dedup. Query flow:
embed → Qdrant top-20 → (phase 2: rerank to top-5) → citation-forcing prompt → streamed answer with sources.

Services in [docker-compose.yml](docker-compose.yml):
- **ollama** (`:11434`) — LLM runtime, OpenAI-compatible API. `OLLAMA_KEEP_ALIVE=-1` pins the model
  in RAM; `OLLAMA_NUM_PARALLEL=1` (one CPU generation at a time).
- **qdrant** (`:6333` REST / `:6334` gRPC) — vector DB; scalar-quantized vectors in RAM, payloads on disk.
- **open-webui** (`:3000`) — chat UI; points at Ollama today, will point at the RAG API (`/v1`) in Phase 4.
- **rag-api** and **ingestion-worker** — **stubbed/commented** in the compose file. Their topology and
  memory budget are declared now so the profile split stays honest; uncomment and add build contexts
  under `./services/` in Phase 3.

All ports bind `127.0.0.1` only; external exposure is hardened to Open-WebUI-behind-TLS in Phase 4/7.
Persistent data lives on the host at `${DATA_DIR:-/data/rag}` with subdirs `models/ qdrant/ corpus/
snapshots/ open-webui/`. The Qdrant index is disposable (rebuildable from corpus); the corpus is not.

## Working in this repo

There is no build/lint/test tooling yet (app code doesn't exist). Current work is compose + infra:

```bash
docker compose config                 # validate the compose file + .env substitution
docker compose --profile dev config   # (after task 0.6) validate the dev profile
docker compose up -d                   # bring up ollama/qdrant/open-webui
docker compose ps                      # service status
docker compose logs -f <service>       # tail logs

# Health checks used as task verifications:
curl localhost:11434/api/tags          # Ollama up
curl localhost:6333/healthz            # Qdrant up

docker exec rag-ollama ollama pull qwen3:4b            # dev generation model (≤4B)
docker exec rag-ollama ollama pull nomic-embed-text    # dev embedder
```

Conventions to follow:
- **Verification-gated progress.** When you complete a PROGRESS.md task, run its `_Verify:_` step and
  only then check the box. Don't mark a phase Done until its gate criteria pass.
- **This is not yet a git repo** (task 0.7 initializes it). Don't assume git history exists.
- Record irreversible/expensive choices in PROGRESS.md's **Cross-phase decisions log** (model choice,
  reranker kept?, RAM upgrade, GPU procurement) so PLAN.md §7's upgrade path stays honest.
