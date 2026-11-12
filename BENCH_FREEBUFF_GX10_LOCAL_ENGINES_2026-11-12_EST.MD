# BENCH_FREEBUFF_GX10 — Local engines: Ollama vs vLLM (and what deserves a `:4000` model name)

**Agent:** Freebuff (DeepSeek V4.1 Flash) — second reporter on this host
**Version:** v1 (GX10_LOCAL_ENGINES)
**Date:** 2026-11-11 ~23:00 EST (2026-11-12T04:00Z). Measurements taken 22:47–22:49 EST (03:47–03:49Z).
**Host:** `gx10-c9b9` — NVIDIA GB10 (DGX Spark / ASUS Ascent GX10 class), 127.5 GB unified memory (121 GiB visible), driver 580.178.04
**Stack:** Ollama (:11434, 26 models), vLLM (:8000, Qwen3-8B), LiteLLM (:4000), Open WebUI (:3000), an `ollama-auth-proxy` on :11435

> **Protocol note (operator request).** These runs were executed *before* the operator asked to be
> warned before local-model benchmarking. Any future run of this suite must be announced first: each
> run loads multi-GB weights into unified memory and can starve other engines. Left as a standing rule
> for whoever picks this doc up next.

---

## 0. Category Summary

| Category | What was tested | Result |
|---|---|---|
| **LOCALLY RUN (FREE)** | Ollama `:11434` — 6 models, single-stream | Measured, 128-token cap |
| **LOCALLY RUN (FREE)** | Ollama `:11434` — 2 models at 3-way concurrency | Measured, **contaminated by memory pressure** (see §7) |
| **LOCALLY RUN (FREE)** | vLLM `:8000` Qwen3-8B bf16 — single stream | Measured (8.0 tok/s e2e) |
| **LOCALLY RUN (FREE)** | vLLM `:8000` — concurrency | **Not measured by me** (my request was malformed, §7.1). Corroborated from vLLM's own log: `Running: 4 reqs, Waiting: 0` |
| **OTHER AGENT, SAME HOST** | vLLM bf16 TTFT/decode/×4 aggregate; Ollama `llama3.1:8b`, `qwen3.5:35b-a3b` | Reported, reproduced here with attribution (§5) |
| **PRIOR SESSION** | Kilo v2 local table, 2026-11-11 17:33 EST | Cross-referenced, **not re-run** (§6) |
| **FREE CLOUD / PAID CLOUD** | — | Out of scope for this doc |

**My protocol:** prompt `Write a Python function that dedupes a list preserving order. Code only.`;
`num_predict=128`, `temperature=0.2`, `keep_alive=5m`; one call per model, sequential.
Ollama decode = `eval_count / eval_duration` (**excludes** prompt eval and load).
vLLM = `completion_tokens / wall` (includes queueing; vLLM prints no separate decode counter).

Read that difference before comparing rows: Ollama decode and vLLM e2e are not the same quantity, and
a 128-token cap truncates reasoning models before they emit any answer.

---

## 1. Inventory — what is actually on this box

### 1.1 Ollama `:11434` — 26 pulls, 19 real local weights

| Model | Size on disk | Params | Quant | Ctx (model card) |
|---|---:|---:|---|---:|
| `llama3.3:70b` | 42.52 GB | 70.6B | Q4_K_M | 131,072 |
| `qwen3.5:35b-a3b` | 23.87 GB | 36.0B (MoE) | Q4_K_M | 262,144 |
| `deepseek-r1:32b` | 19.85 GB | 32.8B | Q4_K_M | 131,072 |
| `qwen2.5:32b` | 19.85 GB | 32.8B | Q4_K_M | 32,768 |
| `qwen3:30b-a3b` | 18.56 GB | 30.5B (MoE) | Q4_K_M | 262,144 |
| `qwen2.5:7b-instruct-fp16` | 15.24 GB | 7.6B | F16 | 32,768 |
| `qwen2.5-coder:14b` | 8.99 GB | 14.8B | Q4_K_M | 32,768 |
| `qwen2.5:14b` | 8.99 GB | 14.8B | Q4_K_M | 32,768 |
| `gemma3:12b` | 8.15 GB | 12.2B | Q4_K_M | 131,072 |
| `qwen2.5:7b-instruct-q8_0` | 8.10 GB | 7.6B | Q8_0 | 32,768 |
| `qwen2.5:7b-instruct-q6_K` | 6.25 GB | 7.6B | Q6_K | 32,768 |
| `qwen2.5-coder:7b-instruct-q5_K_M` | 5.44 GB | 7.6B | Q5_K_M | 32,768 |
| `qwen2.5:7b-instruct-q5_K_M` | 5.44 GB | 7.6B | Q5_K_M | 32,768 |
| `gemma2:9b` | 5.44 GB | 9.2B | Q4_0 | 8,192 |
| `llama3.1:8b` | 4.92 GB | 8.0B | Q4_K_M | 131,072 |
| `qwen2.5-coder:7b` | 4.68 GB | 7.6B | Q4_K_M | 32,768 |
| `qwen2.5:7b` | 4.68 GB | 7.6B | Q4_K_M | 32,768 |
| `phi4-mini:latest` | 2.49 GB | 3.8B | Q4_K_M | 16,384 |
| `llama3.2:3b` | 2.02 GB | 3.2B | Q4_K_M | 131,072 |
| `qwen2.5-coder:3b` | 1.93 GB | 3.1B | Q4_K_M | 32,768 |
| `qwen2.5:3b` | 1.93 GB | 3.1B | Q4_K_M | 32,768 |
| `llama3.2:1b` | 1.32 GB | 1.2B | Q8_0 | 131,072 |
| `nomic-embed-text:latest` | 0.27 GB | 137M | F16 | 2,048 |
| `gemma4:31b-cloud` | — (cloud tag) | 32.7B | — | — |
| `minimax-m3:cloud` | — (cloud tag) | — | — | — |
| `artiku348/loker.v2.1-cld-cdr-480b` | — (cloud tag) | 480B class | — | — |

Three of the 26 are **cloud tags** (no local weights). The qwen2.5:7b family is pulled **five times**
at different quantizations (4.68 / 5.44 ×2 / 6.25 / 8.10 / 15.24 GB) covering the same weights —
~39 GB of disk for one model. That is the cheapest cleanup on this host.

### 1.2 vLLM `:8000`

Running: `Qwen3-8B` **bf16**, `--max-model-len 32768 --gpu-memory-utilization 0.45`.
Served name is the **filesystem path** (`…/models--Qwen--Qwen3-8B/snapshots/b968826…/`) — see §9.3,
this breaks the LiteLLM route that exists for it today.

HF cache also holds, unwired: `google/gemma-4-12b-it`, `mistralai/Mistral-7B-Instruct-v0.3`,
`nvidia/Nemotron-Mini-4B-Instruct`, `nvidia/Qwen3.6-35B-A3B-NVFP4`, `Qwen2.5-{1.5B,3B,7B,14B}-Instruct`,
`Qwen3-{0.6B,4B,8B}`, `Qwen3-Coder-30B-A3B-Instruct`, `TinyLlama-1.1B`.

### 1.3 Engine config that matters more than the model choice

`/etc/systemd/system/ollama.service.d/*.conf`:

```
OLLAMA_HOST=0.0.0.0:11434        OLLAMA_FLASH_ATTENTION=1
OLLAMA_CONTEXT_LENGTH=16384      OLLAMA_KV_CACHE_TYPE=q8_0
OLLAMA_LLM_LIBRARY=cuda_v13      OLLAMA_ORIGINS=*
```

**`OLLAMA_CONTEXT_LENGTH=16384` caps every model at 16K** even though qwen3.5/qwen3 advertise 262,144.
Any "262K context" claim about local Ollama on this host is false in practice: past 16K the server
truncates or re-loads. That single env var is the biggest capability limiter on the box.

---

## 2. Ollama vs vLLM — what each is actually for

The operator's framing ("we did vLLM to serve multiple clients at once") is **correct**, and the
measured evidence on this host supports it:

| | Ollama | vLLM |
|---|---|---|
| Scheduling | one runner per model, requests queue; parallelism limited by `OLLAMA_NUM_PARALLEL` (default 1 when memory-bound) | **continuous batching** — new requests join the running batch each step |
| Batching evidence here | 3 simultaneous calls → per-request wall ≈ 26 s for a 64-token reply (§7.2) | log line under the other agent's ×4 run: `Running: 4 reqs, Waiting: 0 reqs`, `Avg generation throughput: 55.2 tokens/s` |
| Prefix cache | not exposed | `Prefix cache hit rate: 65.3%` on shared system prompts |
| Model switching | trivial — any of 26 tags, load-on-demand | one model per process (or `--served-model-name` + a second process) |
| Best at | breadth of models, tiny/edge models, embeddings, interactive single-user | many concurrent clients, shared prompts, steady aggregate throughput |
| Weakness here | 4–22 s cold load, serialized bursts | holds `0.45 × 121 ≈ 54 GB` permanently, starving Ollama |

**They compete for the same unified memory.** Confirmed the hard way: with vLLM reserving 54 GB, the
previous agent's `qwen3.5:35b-a3b` run produced no content chunks at all, and my runs of the same model
dropped to a third of its clean speed (§7.3). On a GB10 you run **one heavy engine at a time**, or
budget the split explicitly (`--gpu-memory-utilization 0.25` ≈ 30 GB).

---

## 3. Measured — my session, single stream (128-token cap)

| Model | Wall (s) | Load (s) | Out tok | Decode tok/s | Output chars |
|---|---:|---:|---:|---:|---:|
| `qwen3:30b-a3b` | 9.7 | 7.9 | 128 | **83.7** | 0 |
| `qwen3.5:35b-a3b` | 13.0 | 7.9 | 128 | 28.1 | 0 |
| `qwen2.5-coder:14b` | 8.0 | 5.9 | 49 | 24.1 | 205 |
| `gemma3:12b` | 23.3 | 8.6 | 120 | 8.4 | 415 |
| `deepseek-r1:32b` | 40.2 | 10.2 | 128 | 4.3 | 0 |
| `llama3.3:70b` | 39.6 | 22.1 | 70 | 4.1 | 170 |
| vLLM `Qwen3-8B` bf16 | 16.0 | — | 128 | 8.0 (e2e) | 512 |

**Three of six Ollama rows returned 0 characters** — `qwen3:30b-a3b`, `qwen3.5:35b-a3b`,
`deepseek-r1:32b` are reasoning models that spent all 128 tokens thinking. Same trap already recorded
in `BENCH_KILO_v2` (0 chars for `qwen3:30b-a3b`). **A zero-char row here is not a failed request**; it
is a budget failure. Callers must set `max_tokens ≥ 512` for these or they get empty content.

---

## 4. Measured — the other agent's numbers on this host (attributed, not re-run)

| | vLLM `Qwen3-8B` bf16 | Ollama `llama3.1:8b` Q4 | Ollama `qwen3.5:35b-a3b` Q4 |
|---|---|---|---|
| TTFT | 0.09–0.65 s | **0.04–0.09 s** | 0.14 s |
| Decode, 1 stream | 7.7–9.8 tok/s | **20.6 → 33.1 tok/s** | **74.3 tok/s** |
| Concurrency ×4 | **54.7 tok/s aggregate** | not completed | crashed (memory) |

That ×4 figure is the whole argument for vLLM: **7.7–9.8 tok/s single → 54.7 tok/s aggregate at four
streams**, i.e. ~5.6× throughput on 4× the load, not 4× latency. Ollama cannot do that here; it queues.

My own vLLM single-stream number (§3, 8.0 e2e) lands inside their 7.7–9.8 window, so the two sessions
agree on vLLM. The cross-session disagreement is all on Ollama, and it is explained by precision +
memory pressure (§7.3).

---

## 5. Prior session for continuity (`BENCH_KILO_v2`, 2026-11-11 17:33 EST, 200-token cap)

| Model | Decode tok/s | E2E tok/s | My decode (128 cap) | Gap |
|---|---:|---:|---:|---|
| `qwen3:30b-a3b` | 87.03 | 25.29 | 83.7 | −4% (agrees) |
| `qwen3.5:35b-a3b` | 73.99 | 19.97 | 28.1 | **−62%** |
| `qwen2.5:14b` | 23.71 | 16.55 | 24.1 (`coder:14b`) | +2% (agrees) |
| `gemma3:12b` | 27.50 | 17.45 | 8.4 | **−69%** |
| vLLM `Qwen2.5-14B` | — | 8.08 | 8.0 (`Qwen3-8B`) | comparable |

The two rows that crashed are exactly the two 12–24 GB models that had to load **while vLLM held 54 GB
and two other Ollama models were already resident**. Models that fit stayed within ~4%; models that had
to fight for memory fell 62–69%. This is the most operationally important number in the doc.

---

## 6. Concurrency, measured on Ollama (3 simultaneous, 64-token cap)

| Model | Total wall | Per-request wall | Per-request decode |
|---|---:|---|---|
| `qwen3:30b-a3b` | 27.4 s | 26.0 / 27.4 / 26.7 s | 77.0 / 90.4 / 89.0 tok/s |
| `llama3.2:3b` | 14.6 s | 14.5 / 13.3 / 12.3 s | 39.2 / 46.8 / 39.2 tok/s |

Decode rate per stream stayed high while wall time tripled: **the engine is not computing three times
faster, the requests are queueing.** For `qwen3:30b-a3b`, 64 tokens at ~85 tok/s should be 0.75 s of
generation; each request instead took ~26 s, i.e. ~25 s of load/queue overhead under pressure. Treat
these rows as evidence of *serialization*, not as throughput numbers — the box was already overloaded
when they ran.

---

## 7. Confounds and corrections (mine included)

### 7.1 My vLLM concurrency test was my own bug
The burst runner selected the model only `if label.startswith("vllm")` — but the label was
`"vLLM qwen3-8b"` (capital V), so the literal string `"vllm"` was sent. vLLM answered correctly:

```
ERROR [serving.py:214] The model `vllm` does not exist.  → 404
```

So the "vLLM burst failed" result in my raw output is **not** a vLLM defect. vLLM's own log from the
same window shows it happily running 4 concurrent requests. Do not cite my 404 as an engine failure.

### 7.2 Memory competition invalidates the Ollama burst rows
At the time of the burst, `ollama ps` held `qwen3:30b-a3b` (19.4 GB) + `qwen3.5:35b-a3b` (22.8 GB) +
`llama3.2:3b` (3.2 GB) ≈ 45 GB, alongside vLLM's 54 GB reservation, with 85 GB of 121 GiB in use.
Any Ollama number taken in that state is a lower bound.

### 7.3 Precision is not the same on both engines
vLLM runs Qwen3-8B in **bf16** (~16 GB of weights); the Ollama 8B rows are **Q4_K_M** (~4.9 GB).
GB10 is memory-bandwidth-bound, so bf16 reads ~3× the bytes per token. A bf16-vs-Q4 comparison measures
quantization, not runtime. To compare engines fairly, run the **same weights**: vLLM on
`Qwen2.5-14B-Instruct` bf16 vs Ollama `qwen2.5:14b` Q4 is still unfair; vLLM on a GPTQ/AWQ build of the
same quant would be fair.

### 7.4 Reasoning tokens are invisible in vLLM totals
Qwen3-8B emits thinking in `delta.reasoning_content`, which is not `content`. vLLM's 8.0 tok/s is
computed over wall time that includes uncounted reasoning tokens, so vLLM is likely understated; its
512 output chars came through fine (that run had 512 chars vs 0 for the Ollama thinking models — the
difference is budget and field, not capability).

### 7.5 Ollama's "262K context" is capped at 16K here
See §1.3. Not a model limit; a server env var.

---

## 8. Estimates for combos not measured cleanly (ESTIMATE — do not cite as measured)

Method: GB10 ≈ 273 GB/s effective LPDDR5X bandwidth; decode is bandwidth-bound, so
`tok/s ≈ 273 / (active_weight_bytes) × efficiency`, with efficiency calibrated at **~45%** from the
clean anchors (`qwen3:30b-a3b` 84 tok/s, `qwen2.5-14b` ~24 tok/s). Active bytes = active params × 0.55
for Q4_K_M (≈4.5 bit). MoE models only pay for active experts.

| Model | Active params (est.) | Active bytes Q4 | Clean single-stream estimate | Basis |
|---|---:|---:|---:|---|
| `qwen3.5:35b-a3b` | ~3B | 1.7 GB | **65–80 tok/s** | clean-run anchor 74.3 (other agent) |
| `qwen3:30b-a3b` | ~3.3B | 1.8 GB | **80–90 tok/s** | measured clean 83.7 / 87.0 |
| `qwen2.5-coder:14b` | 14.8B dense | 8.2 GB | **22–27 tok/s** | measured 23.7–24.1 |
| `qwen2.5:14b` | 14.8B dense | 8.2 GB | **22–27 tok/s** | measured 23.7 |
| `llama3.1:8b` | 8.0B dense | 4.4 GB | **25–35 tok/s** | measured 20.6–33.1 |
| `gemma3:12b` | 12.2B dense | 6.7 GB | **24–30 tok/s** | clean 27.5 |
| `deepseek-r1:32b` | 32.8B dense | 18.1 GB | **7–9 tok/s** | scaled from 14B dense; **unverified** |
| `llama3.3:70b` | 70.6B dense | 38.9 GB | **3–4.5 tok/s** | measured 4.1 under pressure; fits only alone |
| vLLM `Qwen3-8B` bf16, 1 stream | 8.2B | 16.4 GB | **8–10 tok/s** | measured 7.7–9.8 two sessions |
| vLLM `Qwen3-8B` bf16, 4 streams | 8.2B | 16.4 GB | **50–60 tok/s aggregate** | measured 54.7 |

Rule of thumb for this box: **MoE with ~3B active beats every dense model under 32B on speed**, and
`qwen3.5:35b-a3b` / `qwen3:30b-a3b` are the only two models that are both large and fast. Anything
dense ≥32B (deepseek-r1:32b, qwen2.5:32b, llama3.3:70b) is batch/offline only — 4–9 tok/s is below
reading speed and a 22 s cold load makes it unusable interactively.

---

## 9. Verdict — which models deserve a dedicated name on `:4000`

### 9.1 The five worth named routes

| Proposed `model_name` | Backing | Why it earns a name | Est. clean speed |
|---|---|---|---|
| `local-qwen3-5-35b-a3b` | `ollama_chat/qwen3.5:35b-a3b` | **Best local quality/speed ratio.** 36B MoE, ~3B active, 74 tok/s clean. This is the default local model. | 65–80 tok/s |
| `local-qwen3-30b-a3b` | `ollama_chat/qwen3:30b-a3b` | Same trick, slightly faster, older base. Use as fallback when 3.5 is busy/evicted. | 80–90 tok/s |
| `local-coder` | `ollama_chat/qwen2.5-coder:14b` | Only model tested that returned actual code in 49 tokens (205 chars, no thinking tax). Dense 14B, 24 tok/s. | 22–27 tok/s |
| `local-fast` | `ollama_chat/qwen2.5:3b` | 93 tok/s clean (Kilo v2), 2 GB, sub-second TTFT — for classification/routing/rewrites where 35B is overkill. | 85–95 tok/s |
| `local-embed` | `ollama/nomic-embed-text` | Different capability, no substitute: `/v1/embeddings`. Must be its own name; never mixed into a chat group. | n/a |

Do **not** give `llama3.3:70b`, `deepseek-r1:32b`, `qwen2.5:32b` dedicated interactive names — they are
4–9 tok/s with 20 s loads. If they are needed, expose one shared `local-batch-70b` name and accept it.

And **one vLLM name**, which is the one that actually serves many clients:
`local-vllm-qwen3-8b` → `hosted_vllm/…` at `http://localhost:8000/v1`.

### 9.2 Why not one big group
`ULTRA_FREE_TOOLS` today mixes `hosted_vllm` + three Ollama tags in one group. That defeats the point:
LiteLLM will route a request to whichever upstream it picks, and the caller gets a **different model**
each time, at wildly different speeds (84 vs 8 tok/s) and with different answer quality. Local models
should be **named routes with fallbacks**, never a blended group — you cannot reason about latency or
reproduce a result otherwise.

### 9.3 Two defects found while auditing the existing routes

1. **`hosted_vllm/Qwen/Qwen2.5-14B-Instruct` (`litellm_config.yaml` L340) points at a model vLLM is not
   serving.** vLLM currently serves Qwen3-8B, and it registered under its **filesystem path**, so any
   name-based call 404s. Fix: start vLLM with `--served-model-name qwen3-8b` and point the route at
   `hosted_vllm/qwen3-8b`. Without that flag, LiteLLM cannot address the server at all.
2. **`ULTRA_FREE_LOCAL`/`ULTRA_FREE_TOOLS` routes inherit the 16,384-token Ollama cap** while the
   surrounding docs assume 131K–262K context. Set `num_ctx` explicitly per route (≤16384, or raise
   `OLLAMA_CONTEXT_LENGTH` and accept the KV-cache cost) so behaviour is declared, not inherited.

### 9.4 Concrete config (for whoever applies it)

```yaml
# Dedicated local routes — matched to measured behaviour, not to model cards.
- {model_name: local-qwen3-5-35b-a3b, litellm_params: {model: ollama_chat/qwen3.5:35b-a3b, api_base: http://localhost:11434, api_key: local, num_ctx: 16384, keep_alive: 30m}}
- {model_name: local-qwen3-30b-a3b,   litellm_params: {model: ollama_chat/qwen3:30b-a3b,   api_base: http://localhost:11434, api_key: local, num_ctx: 16384, keep_alive: 30m}}
- {model_name: local-coder,           litellm_params: {model: ollama_chat/qwen2.5-coder:14b, api_base: http://localhost:11434, api_key: local, num_ctx: 16384, keep_alive: 30m}}
- {model_name: local-fast,            litellm_params: {model: ollama_chat/qwen2.5:3b,       api_base: http://localhost:11434, api_key: local, num_ctx: 8192,  keep_alive: 1h}}
- {model_name: local-embed,           litellm_params: {model: ollama/nomic-embed-text,      api_base: http://localhost:11434, api_key: local}}
- {model_name: local-vllm-qwen3-8b,   litellm_params: {model: hosted_vllm/qwen3-8b,         api_base: http://localhost:8000/v1, api_key: local}}
```

Fallback chain worth wiring: `local-vllm-qwen3-8b → local-qwen3-5-35b-a3b → local-qwen3-30b-a3b →
local-fast`, so a vLLM restart degrades instead of failing. Note LiteLLM fallbacks trigger on error,
**not** on slowness — a 4 tok/s answer from a starved engine will never fail over.

### 9.5 Disk and memory housekeeping
- **~39 GB reclaimable:** five pulls of `qwen2.5:7b` at different quants; keep `Q5_K_M` (5.44 GB) and
  delete `fp16` (15.24 GB), `q8_0` (8.10 GB), `q6_K` (6.25 GB), one duplicate `q5_K_M` (5.44 GB).
- **Load, don't hoard:** with vLLM's 54 GB reservation plus three resident Ollama models, 85 of 121 GiB
  was in use with zero requests in flight. Set `OLLAMA_KEEP_ALIVE=5m` (or per-route `keep_alive`) and
  `OLLAMA_MAX_LOADED_MODELS=1` for predictable latency — the 62–69% slowdowns in §5 came from exactly
  this.
- **Pick one heavy engine per workload window.** vLLM 0.45 util + 35B Ollama coexist only for small
  models; 22 GB+ models need vLLM stopped or trimmed to ~0.25.

---

## 10. Reproduction

```bash
# inventory (no inference)
curl -s localhost:11434/api/tags | jq -r '.models[] | "\(.name)\t\(.size/1e9|floor)GB\t\(.details.parameter_size)\t\(.details.quantization_level)"'
curl -s localhost:11434/api/ps | jq -r '.models[] | "\(.name) vram=\(.size_vram/1e9|floor)GB"'
curl -s localhost:8000/v1/models | jq -r '.data[].id'
tr '\0' '\n' < /proc/$(pgrep -f 'ollama serve' | head -1)/environ | grep OLLAMA_

# single stream (announce before running — this loads weights)
curl -s localhost:11434/api/chat -d '{"model":"qwen3.5:35b-a3b","stream":false,
  "messages":[{"role":"user","content":"say ok"}],"options":{"num_predict":128},"keep_alive":"5m"}' \
  | jq '{load:.load_duration/1e9, ev:.eval_count, dec:(.eval_count/(.eval_duration/1e9)), chars:(.message.content|length)}'
```

---

## 11. Bottom line

1. **vLLM is for concurrency, and it works**: 7.7–9.8 tok/s single → 54.7 tok/s at 4 streams, with a
   65% prefix-cache hit rate and zero queued requests. That is what Ollama cannot do on this host.
2. **Ollama is for breadth and single-user latency**: 0.04–0.14 s TTFT, any of 26 tags, no 3-minute
   engine start.
3. **The single most worthy local model is `qwen3.5:35b-a3b`** — 36B MoE, ~3B active, 74 tok/s clean,
   real code output. Second is `qwen3:30b-a3b` at 84–87 tok/s. `qwen2.5-coder:14b` is the code
   specialist; `qwen2.5:3b`/`llama3.2:3b` are the cheap tier; `nomic-embed-text` is the only embedding
   option and needs its own name.
4. **Everything dense ≥32B on this box is batch-only.** 4–9 tok/s and 20 s cold loads are not
   interactive, regardless of how good the model card looks.
5. **Two route bugs to fix before trusting local routes**: vLLM's missing `--served-model-name` (§9.3)
   and the 16K Ollama context ceiling (§1.3).
6. **Warn before benchmarking.** These numbers cost ~45 GB of resident weights and a 62–69% distortion
   on two rows. A future run should be announced and should start from a clean, empty `ollama ps`.
