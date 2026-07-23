# Private RAG System — Progress Tracker

Task breakdown of [MASTER-PLAN.md](MASTER-PLAN.md). Check a box only when its **verification** passes.
Mark a phase "Done" when every task in it is checked and its gate criteria are met.

Status legend: `[ ]` todo · `[~]` in progress · `[x]` done · `[!]` blocked

**This tracker is in two parts — finish Part A before starting Part B:**
- **Part A — Development** (Phases 0–5, on the dev VM): build and *functionally* validate the whole stack.
- **Part B — Deployment** (Phases 6–9, on the R360): stand it up on real hardware, run the performance gate, and harden for production.

---

## Environments

Two machines. **Develop here, deploy to the R360.** MASTER-PLAN.md's hardware assessment
and RAM budget describe the **deploy target**, not this dev host.

| | **Dev host (this box)** | **Deploy target (R360)** |
|---|---|---|
| CPU | AMD EPYC 9354P, **4 vCPUs** (VM) | Xeon E-2468, 8C/16T |
| RAM | **15GB, no swap** (~11GB free) | 32GB DDR5 |
| Storage | single `/`, **108GB free, no LVM** | 5× 1.2TB SAS + LVM |
| Docker | 28.3.3 + Compose v2.39.1 ✅ | to install |
| GPU | none | none |

**What this means:**

- **The dev host builds and validates *function*, not *performance*.** Its 4 shared
  vCPUs make any prefill/generation number here meaningless for the go/no-go decision.
  That gate lives in **Part B, Phase 8** and runs on the R360.
- **Memory ceiling is hard.** No swap → the full stack + an 8B model here will OOM-kill.
  Development uses a ≤4B model and never runs reranker + LLM + embedder together.

---

## Development → Deployment workflow

Two tracks, one codebase:

- **Development (this VM) — Part A / Phases 0–5.** Build and functionally validate the
  whole pipeline against the **`dev` profile**. Proves the system *works*.
- **Deployment (R360) — Part B / Phases 6–9.** Promote the same codebase to the **`prod`
  profile**, run the performance gate, and harden. Proves it's *fast enough* and *durable*.

**One compose file, two profiles.** Keep a single `docker-compose.yml` (shared base +
`dev`/`prod` overlay via Compose `profiles:` or a `docker-compose.prod.yml` override).
The two environments differ **only** in env vars, `mem_limit`s, model, and volume
source — **never in service topology**. That invariant is what makes "works on dev"
mean "same graph on prod."

| Setting | `dev` profile (this VM) | `prod` profile (R360) |
|---|---|---|
| Generation model | Qwen3-4B Q4_K_M | Qwen3-8B Q4_K_M |
| Embedder | nomic-embed-text | bge-m3 |
| Reranker | off (isolation test only) | on (bge-reranker-v2-m3) |
| `OLLAMA` threads | 4 | 8 |
| `/data/rag` volume | plain dir on `/` | LVM mount |
| `mem_limit`s | sized for 15GB, no swap | sized for 32GB |
| Corpus | ~20–50 docs | few hundred → full |

**Cutover — what moves vs. what is rebuilt on the R360:**

| Transfers as-is (git / snapshot) | Rebuilt on the R360 |
|---|---|
| App code (FastAPI, ingestion worker) | Model 4B → 8B, threads 4 → 8 |
| Compose base + `prod` overlay | Embedder nomic → bge-m3; reranker on |
| Prompt templates, chunking config | Storage: plain dir → LVM `/data/rag` |
| Golden set + eval harness | Governor, `mem_limit`s, performance benchmarks |
| Qdrant snapshot (optional index seed) | Viability decision (interactive/async/GPU) |

---
---

# PART A — DEVELOPMENT (dev VM)

Build and functionally validate the full stack under the `dev` profile. Nothing here
measures performance — that is Part B. **Definition of done:** every Phase 0–5 gate green.

---

## Phase 0 — Platform prep (dev)

**Goal:** a clean Docker host with a working directory, compose profiles, and a repo.

- [x] **0.1 Record dev host baseline** — capture CPU/RAM/swap/disk.
  - _Verify:_ documented in the Environments table (4 vCPU EPYC 9354P, 15GB no swap, 108GB free on `/`).
- [x] **0.2 Verify AVX2** — `lscpu | grep -i avx2` returns a match.
  - _Verify:_ `avx2` present in flags. ✅ (confirmed on this host).
- [x] **0.3 Create `/data/rag`** — plain dir on `/` (108GB free) with subdirs `models/ qdrant/ corpus/ snapshots/`, owned `ubuntu:ubuntu`, mode 775.
  - _Verify:_ path exists; host write OK; **container bind-mount write OK as both root and `--user 1000:1000`** (tested with `alpine`). ✅
- [x] **0.4 Docker + Compose present** — engine + compose plugin available.
  - _Verify:_ `docker run hello-world` + `docker compose version` v2+. ✅ (28.3.3 + v2.39.1).
- [x] **0.5 Create compose skeleton** — `docker-compose.yml` + `.env` (dev knobs). ollama/qdrant/open-webui defined (app services stubbed for Phase 3); volumes at `/data/rag`, `restart: unless-stopped`, json-file log rotation (10m×3), localhost-only ports, `mem_limit`s from `.env` summing ~9g of 15g.
  - _Verify:_ `docker compose config` validates ✅; env substitution resolves (`OLLAMA_NUM_THREAD=4`, mem_limits 6g/2g/1g, volumes → `/data/rag/*`).
- [x] **0.6 Split into `dev`/`prod` profiles** — shared base + `.env` (dev) / `.env.prod` (prod) overlay. The two differ **only** in model, embedder, threads, volume source, and `mem_limit`s (see workflow table) — identical service topology. (Env-file overlay, not Compose `--profile`: `--profile` gates *which services run*, not env *values*, so it can't express model/threads/mem_limit deltas — CLAUDE.md's "parameterize differences through `.env` / the prod overlay.")
  - _Verify:_ `docker compose config` (dev) and `docker compose --env-file .env.prod config` (prod) both validate ✅; rendered diff is only threads (4→8) and mem_limits (ollama 6g→14g, qdrant 2g→6g, webui 1g→2g). Model/embedder deltas land once the app services are uncommented (Phase 3); volume source is the same `/data/rag` path (LVM vs plain dir is host fstab, not compose). ✅
- [x] **0.7 Init git repo + README** — repo already initialized; added [README.md](README.md) (architecture, prerequisites, two-environment model, dev quick-start, prod overlay, repo layout) and a `.gitignore`. Versions the compose files, env overlays, and this tracker.
  - _Verify:_ README documents fresh clone → `docker compose config` → `docker compose up -d` (dev; env-file overlay, not `--profile`) with health checks and model pulls ✅; `git status` clean after the commit landing these files. ✅

**Gate:** `/data/rag` writable, compose skeleton + dev/prod overlays validate, repo initialized. ✅ **Phase 0 DONE.**

---

## Phase 1 — Inference baseline (functional)

**Goal:** Ollama serves a small model + embeddings that the pipeline can call. **No
benchmarking here** — the prefill/generation numbers and the viability decision are
Part B, Phase 8 (they're only meaningful on R360 hardware).

- [ ] **1.1 Deploy Ollama** — add Ollama service to compose; container starts and stays up.
  - _Verify:_ `curl localhost:11434/api/tags` returns 200.
- [ ] **1.2 Pull dev generation model** — `ollama pull` **Qwen3-4B Q4_K_M** (≤4B; an 8B + full stack OOMs 15GB).
  - _Verify:_ model listed in `/api/tags`; a trivial completion returns text.
- [ ] **1.3 Pull embedding model** — nomic-embed-text (~550MB) to wire the pipeline; switch to bge-m3 when Phase 3 hybrid search needs sparse vectors.
  - _Verify:_ embedding request returns a vector of expected dimension.
- [ ] **1.4 Set runtime tuning** — `OLLAMA_NUM_PARALLEL=1`, `OLLAMA_KEEP_ALIVE=-1`, **threads=4** (4 vCPUs).
  - _Verify:_ env vars present in compose; model stays resident (no reload between calls).

**Gate:** Ollama serves a ≤4B model + embeddings; model stays resident; the pipeline can call it.

---

## Phase 2 — Vector store + ingestion

**Goal:** documents flow from a folder/upload into Qdrant as retrievable, deduplicated chunks.

- [ ] **2.1 Deploy Qdrant** — add service; scalar-quantized dense vectors in RAM, payloads on disk (under `/data/rag`).
  - _Verify:_ `curl localhost:6333/healthz` OK; collection config shows quantization + on-disk payload.
- [ ] **2.2 Create collection** — dense (+ sparse for hybrid) vector params matching the embedder's dims.
  - _Verify:_ `GET /collections/<name>` returns expected vector config.
- [ ] **2.3 Docling parse step** — parse PDF/DOCX/PPTX incl. tables; unstructured fallback wired.
  - _Verify:_ a sample PDF and DOCX each produce clean structured text.
- [ ] **2.4 Structure-aware chunking** — ~500–800 tokens, 15% overlap, headings kept in metadata.
  - _Verify:_ chunk token counts within range; heading metadata present on sample chunks.
- [ ] **2.5 Embed + upsert** — embed chunks, upsert to Qdrant.
  - _Verify:_ Qdrant point count matches produced chunk count.
- [ ] **2.6 Doc-hash dedup / re-ingest** — store doc hash; re-ingesting an unchanged file is a no-op.
  - _Verify:_ re-running ingest on same file adds 0 new points; changed file updates.
- [ ] **2.7 Watched folder + upload endpoint** — drop-in folder and HTTP upload both trigger ingestion.
  - _Verify:_ file placed in watched folder appears as points within a worker cycle.
- [ ] **2.8 Ingest a tiny dev corpus** — ~20–50 docs, enough to exercise parse/chunk/embed/upsert.
  - _Verify:_ ingestion completes without errors; point count logged. (The few-hundred-doc pilot runs on the R360, Phase 7.)
- [ ] **2.9 Retrieval spot-check** — run 5–10 known queries against raw vector search.
  - _Verify:_ top hits are on-topic for each; obvious misses noted.

**Gate:** tiny corpus ingested, dedup works, spot-check retrieval looks sane.

---

## Phase 3 — RAG API

**Goal:** a FastAPI service that answers with citations from retrieved context.

- [ ] **3.1 Scaffold FastAPI service** — add to compose; `/health` returns 200.
  - _Verify:_ `curl localhost:<port>/health` OK.
- [ ] **3.2 `/ingest` endpoint** — accepts docs, invokes the Phase 2 pipeline.
  - _Verify:_ POSTing a doc increases Qdrant point count.
- [ ] **3.3 Hybrid retrieval** — dense + BM25/sparse, top-20 candidates. (Needs bge-m3 for sparse — switch embedder here if still on nomic.)
  - _Verify:_ retrieval returns 20 scored candidates; hybrid beats dense-only on a test query.
- [ ] **3.4 Top-20 → top-5 selection** — narrow to 5 for the prompt.
  - _Verify:_ assembled context contains 5 chunks with metadata.
- [ ] **3.5 Citation-forcing prompt template** — "answer only from context; cite [1][2]".
  - _Verify:_ answers contain inline citation markers tied to sources.
- [ ] **3.6 `/query` streamed** — token streaming end to end.
  - _Verify:_ client receives incremental tokens, not one blob.
- [ ] **3.7 Source metadata in response** — doc, page, heading per answer.
  - _Verify:_ response payload includes source list resolvable back to documents.
- [ ] **3.8 Context budget guard** — assembled prompt ≤4K tokens.
  - _Verify:_ prompt token count logged and capped; oversize context is trimmed, not sent.

**Gate:** `/query` streams a cited answer with resolvable sources under the 4K prompt cap.

---

## Phase 4 — UI + access

**Goal:** a chat UI with auth; internal services not exposed. (Production TLS with the
real hostname/cert is finalized in Part B, Phase 7 — here, validate the path locally.)

- [ ] **4.1 Deploy Open WebUI** — add to compose, pointed at the RAG API (OpenAI-compatible endpoint).
  - _Verify:_ UI loads; a question returns a cited answer through the RAG API.
- [ ] **4.2 Enable local accounts** — auth on; signup/login works.
  - _Verify:_ anonymous access blocked; a test account can log in.
- [ ] **4.3 Network isolation** — bind Ollama/Qdrant/FastAPI to an internal bridge only.
  - _Verify:_ those ports are not reachable from outside the host.
- [ ] **4.4 Reverse-proxy + TLS path** — front with Caddy / reverse proxy; validate the TLS termination path locally (self-signed OK).
  - _Verify:_ HTTPS works locally; HTTP redirects; only Open WebUI is exposed.

**Gate:** only Open WebUI is reachable, behind login, over a working TLS path.

---

## Phase 5 — Quality pass (build + eval harness)

**Goal:** the retrieval/answer quality tooling exists and runs. (The *definitive* A/B
with the reranker co-resident happens on the R360, Phase 8 — 15GB won't hold both here.)

- [ ] **5.1 Reranker integration** — wire bge-reranker-v2-m3 between retrieval and generation (behind the `prod` profile / a flag).
  - _Verify:_ with the LLM stopped, the reranker returns a reordered top-k for a test query (isolation test — won't co-reside with the LLM in 15GB).
- [ ] **5.2 Build golden set** — 30–50 Q/A pairs from the corpus.
  - _Verify:_ golden set file committed with expected answers/sources.
- [ ] **5.3 Eval harness** — script computes retrieval hit-rate + answer faithfulness.
  - _Verify:_ running it emits both metrics as numbers.
- [ ] **5.4 A/B harness (rerank on/off)** — the comparison runs and reports a delta.
  - _Verify:_ harness produces a metrics table for both configs. (Definitive run on prod, Phase 8.)
- [ ] **5.5 Tune chunk size / top-k / prompt** — iterate against dev eval results.
  - _Verify:_ each change logged with before/after metrics; best config promoted (re-confirmed on prod numbers).

**Gate:** golden set + eval harness committed and runnable; reranker validated in isolation.

---
---

# PART B — DEPLOYMENT (R360)

Promote the identical codebase to the `prod` profile on the real hardware, run the
performance gate the dev host could not, and harden for production. **Prereq:** all
Part A gates green — see the *Development → Deployment workflow* cutover table.

---

## Phase 6 — R360 platform prep

**Goal:** the real Phase 0 on real hardware: dedicated LVM storage, tuned CPU, Docker.

- [ ] **6.0 Pre-flight** — Part A gates green; latest code + `prod` overlay pushed to the R360; cutover table reviewed.
  - _Verify:_ `git log` on the R360 matches dev HEAD; `docker compose --profile prod config` validates on the R360.
- [ ] **6.1 Confirm OS & virtualization load** — record specs; list KVM guests + their RAM/CPU commitments.
  - _Verify:_ free RAM/CPU headroom documented; RAM-upgrade-before-reranker decision recorded.
- [ ] **6.2 Verify AVX2** — `lscpu | grep -i avx2`.
  - _Verify:_ `avx2` present.
- [ ] **6.3 Check free LVM capacity** — `vgs`/`lvs`/`pvs` show ≥500GB free.
  - _Verify:_ free extents ≥ 500GB confirmed.
- [ ] **6.4 Carve dedicated LV** — create ~500GB LV, format, mount at `/data/rag`, add to `/etc/fstab`.
  - _Verify:_ `df -h /data/rag` shows expected size; survives remount.
- [ ] **6.5 Install Docker + Compose** — engine + compose plugin.
  - _Verify:_ `docker run hello-world` succeeds; `docker compose version` v2+.
- [ ] **6.6 Set CPU governor to `performance`** — apply and persist across reboot.
  - _Verify:_ `cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` all read `performance`.

**Gate:** Docker healthy, `/data/rag` LV mounted, AVX2 confirmed, governor set, `prod` profile validates on R360.

---

## Phase 7 — Promote the stack (prod profile)

**Goal:** the full production stack running on the R360, serving cited answers, with the real corpus and TLS.

- [ ] **7.1 Pull prod models** — Qwen3-8B Q4_K_M, bge-m3, bge-reranker-v2-m3.
  - _Verify:_ all three listed/available; a trivial completion + embedding + rerank each return.
- [ ] **7.2 Apply prod tuning** — `threads=8`, `OLLAMA_NUM_PARALLEL=1`, `OLLAMA_KEEP_ALIVE=-1`, `mem_limit`s for 32GB, reranker enabled/co-resident.
  - _Verify:_ env/limits present in the `prod` overlay; models stay resident.
- [ ] **7.3 Bring up the stack** — `docker compose --profile prod up`.
  - _Verify:_ all services healthy; `/query` returns a cited answer end to end.
- [ ] **7.4 Migrate/re-ingest corpus** — restore a Qdrant snapshot from dev or re-ingest the real (few-hundred → full) corpus.
  - _Verify:_ point count matches expectation; golden-set eval (Phase 5) reruns green.
- [ ] **7.5 Finalize production TLS** — real hostname + valid cert via Caddy / existing reverse proxy; expose only Open WebUI.
  - _Verify:_ HTTPS with a valid cert; HTTP redirects; internal services unreachable from outside the host.

**Gate:** prod stack healthy on the R360, real corpus loaded, only Open WebUI exposed over valid TLS.

---

## Phase 8 — Performance gate (the real go/no-go)

**Goal:** the numbers the dev host could not produce, and the decision they drive.

- [ ] **8.1 Benchmark prefill (primary gate)** — `llama-bench` for `pp2048`, `pp4096`, `pp8192`.
  - _Verify:_ `pp4096` tok/s recorded in the Results table below.
- [ ] **8.2 Benchmark generation** — record `tg` tok/s.
  - _Verify:_ generation tok/s recorded.
- [ ] **8.3 Benchmark embedding throughput** — measure chunks/sec.
  - _Verify:_ chunks/sec recorded.
- [ ] **8.4 Make the viability call** — apply §4 thresholds to `pp4096`.
  - _Verify:_ decision written: ≥150 → interactive; 30–150 → 4B + 2K cap or async UX; <30 → async/batch or GPU. Logged in the decisions log.
- [ ] **8.5 Confirm the RAM budget holds** — full stack + 8B + reranker resident.
  - _Verify:_ peak RAM leaves headroom on 32GB; no OOM under a concurrent-query test.

### Performance Results (fill in on the R360)

| Metric | Value | Notes |
|---|---|---|
| pp2048 (tok/s) | | |
| pp4096 (tok/s) | | primary gate |
| pp8192 (tok/s) | | |
| tg (tok/s) | | |
| Embedding (chunks/s) | | |
| **Decision** | | interactive / async / GPU |

**Gate:** benchmarks recorded, RAM budget verified, interactive-vs-async decision made on real data.

---

## Phase 9 — Production hardening + cutover

**Goal:** the stack survives reboots, disk loss of the index, and gets watched.

- [ ] **9.1 Nightly Qdrant snapshots** — scheduled snapshot job.
  - _Verify:_ a snapshot file appears on schedule; restore tested once.
- [ ] **9.2 Corpus backup** — rsync corpus to a second LV / off-box.
  - _Verify:_ backup runs on schedule; restore of a sample file works.
- [ ] **9.3 Restart policies** — `restart: unless-stopped` on all services.
  - _Verify:_ `docker compose kill` of a service → it comes back; survives host reboot.
- [ ] **9.4 Log rotation** — rotation configured for containers.
  - _Verify:_ log size capped; old logs rotate.
- [ ] **9.5 Monitoring + alerts** — Prometheus + node_exporter; RAM/latency alerts.
  - _Verify:_ metrics scraped; a test alert fires.
- [ ] **9.6 Re-ingest runbook** — document rebuild-index-from-corpus procedure.
  - _Verify:_ following the doc from scratch reproduces a working index.
- [ ] **9.7 Smoke test + rollback plan** — post-deploy smoke (health, `/query`, `/ingest`, UI login); documented rollback if a gate fails.
  - _Verify:_ smoke checklist passes; rollback steps written and dry-run once.

**Gate:** backups verified by a real restore, services self-heal, monitoring live, runbook proven, smoke green.

---
---

## Cross-phase decisions log

Record irreversible/expensive decisions as they're made (keeps §7 upgrade path honest).

- [ ] Interactive vs async UX (from Phase 8 gate): _____
- [ ] Generation model choice (Qwen3-8B vs Mistral-7B): _____
- [ ] Reranker kept? (from 5.4 / 8.5): _____
- [ ] RAM upgrade needed? (KVM load / corpus size): _____
- [ ] GPU (L4/T4) procurement decision: _____
