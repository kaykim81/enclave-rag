# Private RAG System — Build Plan (Dell PowerEdge R360)

> **Deploy target vs. dev host.** §1 and the RAM budget below describe the **R360
> deploy target**. Active development happens on a smaller VM (4 vCPU / 15GB / no
> LVM) — see the *Environments* section and `> [dev]` deltas in [PROGRESS.md](PROGRESS.md).

## 1. Hardware Assessment

| Component | Spec | Implication for RAG |
|---|---|---|
| CPU | Intel Xeon E-2468 (8C/16T, AVX2) | CPU-only inference; llama.cpp-class runtimes required |
| RAM | 32GB DDR5-5600, dual-channel (~90 GB/s) | Memory bandwidth caps generation at ~8–12 tok/s for an 8B Q4 model; 2 DIMM slots free (upgrade path to 64/128GB) |
| GPU | None (Matrox G200eW3 is BMC video only) | No CUDA; all model choices must be CPU-viable |
| Storage | 5× 1.2TB 10K SAS (MegaRAID SAS39xx), LVM | Ample space for corpus, models, and vector indexes; spinning disks → keep hot index in RAM |
| Network | 8× 1GbE (BCM5719) + 2× 1GbE (BCM5720), Linux bridges + virbr0 present | Box appears to be a Linux/KVM host already; 1GbE is fine for internal RAG traffic |

**Bottom line:** this box comfortably runs a fully private RAG stack for a small team
(~5–15 concurrent users for retrieval; 1–2 concurrent LLM generations) using a
quantized 7–8B model. It will not serve 70B-class models at usable speed.

## 2. Recommended Stack

All open-source, all local, no data leaves the box. Every model in the stack is
Apache 2.0 or MIT — no attribution, naming, or MAU obligations. One exception to
"open-source": Open WebUI is source-available (modified BSD-3), free for
commercial use but its branding may not be altered above 50 end users in a
rolling 30-day period without an enterprise license.

| Layer | Choice | Why |
|---|---|---|
| OS / runtime | Existing Linux + Docker Compose | Host already runs Linux (LVM, bridges); Compose keeps the POC reproducible |
| LLM runtime | **Ollama** (llama.cpp under the hood) | Simplest CPU serving, OpenAI-compatible API, easy model swaps |
| Generation model | **Qwen3-8B Q4_K_M** (~5GB) — fallback: Mistral-7B-Instruct-v0.3 Q4_K_M | Best quality/speed at 8B on CPU; strong RAG/citation behavior; both Apache 2.0 |
| Fast-lane model (optional) | Qwen3-4B Q4_K_M (~2.5GB) | ~2× faster when latency matters more than quality |
| Embeddings | **bge-m3** (multilingual, ~2GB) or nomic-embed-text (English, ~550MB) | Fast on CPU; bge-m3 also gives sparse vectors for hybrid search |
| Reranker (phase 2) | bge-reranker-v2-m3 | Big relevance win for ~200–400ms extra per query on CPU |
| Vector DB | **Qdrant** | Lightweight, hybrid dense+sparse search, on-disk payloads, snapshot backups |
| Document parsing | **Docling** (PDF/DOCX/PPTX/tables) + fallback to unstructured | Best open-source layout-aware parsing, CPU-friendly |
| RAG orchestration | **FastAPI + LlamaIndex** (thin) | Keep orchestration inspectable; avoid framework lock-in |
| UI | **Open WebUI** | Chat UI with auth, RAG citations, connects directly to Ollama/OpenAI-compatible APIs |
| Observability | Prometheus + node_exporter (optional for POC) | Watch RAM headroom and token throughput |

### RAM budget (32GB total)

| Item | RAM |
|---|---|
| OS + Docker + services | ~3GB |
| Qwen3-8B Q4_K_M + 8K context KV cache | ~7GB |
| Embedding model (bge-m3) | ~2GB |
| Reranker (when enabled) | ~2GB |
| Qdrant (1M chunks, quantized vectors in RAM) | ~2–3GB |
| Open WebUI + FastAPI + ingestion worker | ~2GB |
| **Headroom / page cache** | **~13GB** |

Fits with margin. If the box must keep running existing KVM guests, do the RAM
upgrade (see §7) before adding the reranker.

## 3. Architecture

```
                        ┌─────────────────────────────────────────┐
                        │            PowerEdge R360               │
  Users ──1GbE──▶ Open WebUI ──▶ FastAPI RAG API ──▶ Ollama (Qwen3-8B)
                        │             │    ▲                      │
                        │             ▼    │ top-k chunks         │
                        │           Qdrant ◀── bge-m3 embeddings  │
                        │             ▲                           │
                        │      Ingestion worker (Docling →       │
                        │      chunk → embed → upsert)           │
                        │             ▲                           │
                        │       Watched folder / upload API       │
                        └─────────────────────────────────────────┘
```

Query flow: question → embed (bge-m3) → Qdrant hybrid search (dense + sparse,
top-20) → \[phase 2: rerank to top-5\] → prompt assembly with citations →
Qwen3-8B generation → streamed answer with source links.

## 4. Build Phases

### Phase 0 — Platform prep (½ day)
- Confirm OS, free LVM capacity; carve a dedicated LV (e.g. 500GB `/data/rag`)
  for models, corpus, and Qdrant storage.
- Install Docker + Compose; pin everything into one `docker-compose.yml`.
- Set CPU governor to `performance`; verify AVX2 with `lscpu`.

### Phase 1 — Inference baseline (½ day)
- Deploy Ollama; pull Qwen3-8B Q4_K_M and bge-m3.
- Benchmark with `llama-bench`: **prefill (pp) throughput at 2K/4K/8K** and
  generation (tg) throughput, plus embedding chunks/sec.
- **Primary gate is prefill, not generation** — prefill dominates RAG query
  latency. Measure `pp4096` first; it determines whether interactive use is
  viable at all.
  - ≥150 tok/s prefill → interactive CPU-only RAG is workable.
  - 30–150 tok/s → expect 30s–2min TTFT: drop to the 4B model, cap context at
    2K, or scope to async/batch UX (see §5).
  - Generation ≥8 tok/s is a secondary check; it is rarely the binding constraint.
- Set `OLLAMA_NUM_PARALLEL=1`, `OLLAMA_KEEP_ALIVE=-1` (pin model in RAM),
  threads = 8 (physical cores — hyperthreads hurt llama.cpp).

### Phase 2 — Vector store + ingestion (1–2 days)
- Deploy Qdrant with scalar-quantized dense vectors (in-RAM) + on-disk payloads.
- Build ingestion worker: watched folder + upload endpoint → Docling parse →
  structure-aware chunking (~500–800 tokens, 15% overlap, keep headings in
  metadata) → embed → upsert. Store doc hash for dedup/re-ingest.
- Ingest a pilot corpus (a few hundred documents); spot-check retrieval quality.

### Phase 3 — RAG API (1–2 days)
- FastAPI service: `/query` (streamed), `/ingest`, `/health`.
- Hybrid retrieval (dense + BM25/sparse via bge-m3), top-20 → top-5 into a
  citation-forcing prompt template ("answer only from context; cite [1][2]").
- Return source metadata (doc, page, heading) with every answer.

### Phase 4 — UI + access (½ day)
- Deploy Open WebUI pointed at the RAG API (OpenAI-compatible endpoint).
- Enable local accounts; bind services to an internal bridge, expose only
  Open WebUI (TLS via Caddy or existing reverse proxy).

### Phase 5 — Quality pass (ongoing)
- Add bge-reranker-v2-m3 between retrieval and generation; A/B against Phase 3.
- Build a 30–50 question golden set from the corpus; track answer faithfulness
  and retrieval hit-rate on every config change.
- Tune chunk size, top-k, and prompt from eval results — not vibes.

### Phase 6 — Ops hardening
- Nightly Qdrant snapshots + corpus rsync to a second LV or off-box.
- `docker compose` restart policies; log rotation; Prometheus RAM/latency alerts.
- Document the re-ingest-from-scratch procedure (index is disposable, corpus is not).

## 5. Expected Performance

> **Revised.** An earlier draft of this table badly understated CPU
> time-to-first-token (claimed 3–8s). Prompt processing is compute-bound, and
> on 8 AVX2 cores it is the dominant cost of a RAG query — not generation.
> Treat all figures below as estimates to be replaced by Phase 1 benchmarks.

### CPU-only (8B Q4_K_M)

| Metric | Expectation |
|---|---|
| Prefill throughput | ~30–50 tok/s (8 cores × AVX2 ≈ 970 GFLOPS peak, ~30–50% realized, ~16 GFLOP/token) |
| Time to first token | ~40–70s at 2K context; **~80–130s at 4K**; ~3min at 8K |
| Generation speed | 8–12 tok/s (bandwidth-bound: ~90 GB/s ÷ ~5GB weights) |
| Embedding throughput | ~5–15 chunks/sec → ~1M chunks ingested overnight |
| Query latency end-to-end | **~2 min** (4K context, 300-token answer) |
| Concurrency | Retrieval: dozens; generation: queue beyond 1–2 simultaneous |

### With a T4 / L4

| Metric | CPU-only | + T4 | Gain |
|---|---|---|---|
| Prefill throughput | ~30–50 tok/s | ~600–900 tok/s | ~15–20× |
| TTFT @ 4K context | 80–130s | ~5–7s | ~15× |
| Generation (8B Q4) | 8–12 tok/s | ~30–40 tok/s | ~3–4× |
| End-to-end query | ~2 min | ~15s | ~7–8× |
| Concurrent generations | 1–2 | 6–8 | ~4× |

**The asymmetry matters.** Generation scales only with the bandwidth ratio
(320 vs ~90 GB/s) and stops at 3–4×. Prefill is compute-bound, where the gap
between tensor cores and AVX2 is far wider. RAG is prefill-heavy by
construction, so the workload sits on the axis where the GPU wins biggest.

**Implication for the POC:** at ~2 minutes to first token, CPU-only is
demonstrable but not adoptable for interactive use. Either budget for a GPU
(§8), or scope the CPU-only phase to batch/async use (submit question →
notification when answered) rather than live chat. Prefix caching does not
rescue this — retrieved chunks differ every query, so only the static system
prompt is reused.

## 6. Risks & Mitigations

- **Long-context prompt processing is slow on CPU** → keep assembled prompts ≤4K
  tokens (tight top-k + reranker beats stuffing context).
- **Concurrent users queue on generation** → fast-lane 4B model, streaming
  responses, and honest UX (queue position) for the POC.
- **10K SAS disks are slow for random reads** → keep vector index in RAM
  (quantized), payloads on disk; page cache absorbs the rest.
- **Box may already host KVM guests** → check existing RAM/CPU commitments
  before Phase 0; cgroup-limit the RAG stack (`mem_limit` per service).

## 7. Upgrade Path (in ROI order)

> **Reordered.** An earlier draft ranked RAM first. That was wrong: RAM adds
> capacity, and the binding constraint is *speed* (specifically prefill).

1. **T4 / L4 GPU** — see §8. Fixes the actual bottleneck: TTFT ~2min → ~15s.
2. **RAM → 64GB** — see §7.1. **No speed benefit.** Buy only if capacity is
   genuinely constrained (KVM guests, corpus >2–3M chunks, or running
   embedder + reranker + a 14B model concurrently).
3. **NVMe via PCIe adapter** for Qdrant if corpus grows past ~5M chunks.

### 7.1 RAM: capacity only, and mind the 2DPC downclock

The E-2468 is **dual-channel**. The 4 DIMM slots are 2 channels × 2 slots, so
filling the empty pair adds no channels and no bandwidth:

```
DDR5-5600 × 8 bytes × 2 channels = 89.6 GB/s   (unchanged by adding DIMMs)
```

Generation is bandwidth-bound, so 8–12 tok/s stays 8–12 tok/s.

**Worse — the naive upgrade is a regression.** Populating 2 DIMMs per channel
on DDR5 UDIMM typically forces a BIOS downclock to ~4400 MT/s:

```
DDR5-4400 × 8 bytes × 2 channels = 70.4 GB/s   (−21% → ~6.5–9.5 tok/s)
```

**Correct approach if 64GB is needed:** buy **2× 32GB ECC UDIMM and replace**
the existing pair, staying at 1 DIMM per channel and full 5600 MT/s. Platform
maximum is 128GB, leaving a path to 4× 32GB later.

Confirm 2DPC speed behavior against Dell's R360 memory population guidelines
before ordering.

**Larger models are a quality/speed trade, not a gain:**

| Model | Weights (Q4) | Est. generation @ 90 GB/s |
|---|---|---|
| 8B | ~5GB | 8–12 tok/s |
| 14B | ~9GB | 4–6 tok/s |
| 32B | ~19GB | 2–3 tok/s |

## 8. GPU Selection

### Form-factor constraints (R360 is 1U)

Any candidate must be **single-slot**, **low-profile/half-height**, and
**≤75W slot-powered** (1U risers rarely expose 8-pin aux power). This
eliminates most consumer and workstation cards regardless of performance.

### Viable options

| | **NVIDIA L4** (recommended) | **NVIDIA T4** (budget) |
|---|---|---|
| VRAM | 24GB GDDR6 | 16GB GDDR6 |
| Bandwidth | ~300 GB/s | ~320 GB/s |
| Power / form | 72W, single-slot LP | 70W, single-slot LP |
| Architecture | Ada, CC 8.9, FP8 | Turing, CC 7.5 |
| Approx. price | ~$2,000–2,500 new | ~$500–800 used |

**Prefer the L4.** Both cards have near-identical memory bandwidth, so they
generate tokens at similar speed. The L4 wins on *compute* (~2×) and VRAM.
Compute governs prompt processing — the dominant cost in a prefill-heavy RAG
workload — so the L4 materially improves time-to-first-token, which is the
metric users actually feel.

### Realistic gains vs. CPU-only baseline

| Metric | CPU-only | With L4/T4 |
|---|---|---|
| Generation (8B Q4) | 8–12 tok/s | ~35–45 tok/s (~3–4×, bandwidth-bound) |
| Time to first token @8K | 8–15s | <1s (~10×+, compute-bound) |
| Concurrent generations | 1–2 | 8–10+ |

Note the asymmetry: the headline win is **prefill/TTFT**, not raw generation.
Generation scales with the 300 vs 90 GB/s bandwidth ratio and no further.

### Explicitly unsuitable

- **NVS 300 / GT218 (or any pre-Maxwell card)** — 512MB VRAM (no model fits),
  compute capability 1.2 (below llama.cpp's 5.0 floor and unsupported by
  CUDA 12), ~12.6 GB/s bandwidth (**7× slower than system RAM** — would run
  *worse* than CPU-only), and driver support ended with the EOL 340.xx branch,
  which will not build against a modern kernel.
- **Minimum viable bar:** ≥8GB VRAM and compute capability ≥7.5 (Turing+).

### Pre-purchase verification

- Confirm the R360 riser config exposes a x16 slot with single-wide GPU
  clearance — check the service tag against Dell's configuration guide.
  GPU support on the entry 1U R360 line is limited and riser-dependent.
- Expect elevated chassis fan RPM with a non-Dell-validated card (the BMC has
  no thermal profile for it).
- If the chassis turns out not to support a GPU, the RAM upgrade (§7.1) plus
  the 4B fast-lane model remains a workable CPU-only path.
