# BENCH_GX10_LOCAL_ENGINES — Ollama vs vLLM on this box + published GX10 numbers

**Agent:** Freebuff (DeepSeek V4.1 Flash)
**Date:** 2026-11-12 00:10 EST (2026-11-12T05:10Z)
**Host:** `gx10-c9b9` — NVIDIA GB10 (Blackwell, SM 12.1), 20-core Arm, 121 GiB unified LPDDR5X @ ~273 GB/s, driver 580.178.04, Ubuntu ARM64
**Stack:** Ollama 0.40.13 (`:11434`), vLLM 0.21.0 (`:8000`, serving `Qwen3-8B` bf16), LiteLLM `:4000`
**Why this file exists:** every prior doc in this repo compares Ollama, vLLM and cloud from *different* runs. This one is one session on one host, plus the third-party GX10 corpus, so the engine question can be answered once instead of re-benchmarked.

---

## 0. Verdict up front

| Question | Answer | Confidence |
|---|---|---|
| **Which is faster for ONE client, one request?** | **Ollama**, decisively, *as currently configured* — 74–85 tok/s on a 30–35B MoE vs 8 tok/s on vLLM's bf16 8B | High for this config; **low as an engine verdict** (see §5) |
| **Which serves MANY clients at once?** | **vLLM**, and it is not close — 54.7 tok/s aggregate at 4 streams vs 8.0 single; published same-host peaks 350 tok/s @16 | High |
| **What actually drove our vLLM to 8 tok/s?** | bf16 weights (16 GB read per token) on a 273 GB/s box, plus reasoning tokens that our counter did not count | High |
| **Was building vLLM justified?** | **Yes, but not for latency.** vLLM is the concurrency path; Ollama is the breadth/quality path. They are not substitutes. | High |
| **Best single model to wire on `:4000`** | `qwen3.5:35b-a3b` (MoE, ~74–84 tok/s decode) at raised context | High |
| **Worst value on this host** | `llama3.3:70b` (dense, 4.1–4.8 tok/s) and `deepseek-r1:32b` (4.3–10.7 tok/s, output eaten by thinking) | High |

**The one-line rule for this hardware:** active parameters decide speed, not total parameters. A 120B MoE at MXFP4 outruns a 70B dense at the same quant *and* scores better, because each token reads ~5B weights instead of 70B.

---

## 1. Engine configuration actually in force (verbatim, not copied from docs)

```
Ollama  (systemd, /etc/systemd/system/ollama.service.d/*.conf)
  OLLAMA_HOST=0.0.0.0:11434        OLLAMA_ORIGINS=*
  OLLAMA_FLASH_ATTENTION=1         OLLAMA_KV_CACHE_TYPE=q8_0
  OLLAMA_CONTEXT_LENGTH=16384      OLLAMA_LLM_LIBRARY=cuda_v13

vLLM    (pid 2845752)
  vllm serve ~/.cache/huggingface/hub/models--Qwen--Qwen3-8B/snapshots/b968826d.../
  --port 8000 --max-model-len 32768 --gpu-memory-utilization 0.45
```

### Three consequences that matter more than any tok/s number

1. **`OLLAMA_CONTEXT_LENGTH=16384` caps every model at 16K**, whatever the catalog advertises. `qwen3.5:35b-a3b` and `qwen3:30b-a3b` both claim 262,144. Anything past 16K is silently truncated unless the request sends `num_ctx` or the env var is raised. Prior docs in this repo quote 261,652-token JSON tests against `qwen3.5:35b-a3b` — that only reproduces if `num_ctx` is passed per call.
2. **`--gpu-memory-utilization 0.45` reserves ~54 GB** of the 121 GiB. That single flag is why a 22 GB Ollama load fails or degrades while vLLM is up (§5.3). Ollama and vLLM are not co-residents at this setting.
3. **vLLM's served model name is the snapshot path**, not `Qwen3-8B`. `/v1/models` returns the full HuggingFace cache path, so any LiteLLM route written as `hosted_vllm/Qwen/Qwen3-8B` 404s. This is a live bug on `:4000` today — see §8.3.

---

## 2. Full local inventory (26 Ollama models, measured sizes)

| Model | Size | Params | Quant | Arch | Catalog ctx |
|---|---:|---:|---|---|---:|
| `llama3.3:70b` | 42.52 GB | 70.6B | Q4_K_M | llama | 131,072 |
| `qwen3.5:35b-a3b` | 23.87 GB | 36.0B (3B active) | Q4_K_M | qwen35moe | 262,144 |
| `deepseek-r1:32b` | 19.85 GB | 32.8B | Q4_K_M | qwen2 | 131,072 |
| `qwen2.5:32b` | 19.85 GB | 32.8B | Q4_K_M | qwen2 | 32,768 |
| `qwen3:30b-a3b` | 18.56 GB | 30.5B (3B active) | Q4_K_M | qwen3moe | 262,144 |
| `qwen2.5:7b-instruct-fp16` | 15.24 GB | 7.6B | F16 | qwen2 | 32,768 |
| `qwen2.5-coder:14b` | 8.99 GB | 14.8B | Q4_K_M | qwen2 | 32,768 |
| `qwen2.5:14b` | 8.99 GB | 14.8B | Q4_K_M | qwen2 | 32,768 |
| `gemma3:12b` | 8.15 GB | 12.2B | Q4_K_M | gemma3 | 131,072 |
| `qwen2.5:7b-instruct-q8_0` | 8.10 GB | 7.6B | Q8_0 | qwen2 | 32,768 |
| `qwen2.5:7b-instruct-q6_K` | 6.25 GB | 7.6B | Q6_K | qwen2 | 32,768 |
| `qwen2.5-coder:7b-instruct-q5_K_M` | 5.44 GB | 7.6B | Q5_K_M | qwen2 | 32,768 |
| `qwen2.5:7b-instruct-q5_K_M` | 5.44 GB | 7.6B | Q5_K_M | qwen2 | 32,768 |
| `gemma2:9b` | 5.44 GB | 9.2B | Q4_0 | gemma2 | 8,192 |
| `llama3.1:8b` | 4.92 GB | 8.0B | Q4_K_M | llama | 131,072 |
| `qwen2.5-coder:7b` | 4.68 GB | 7.6B | Q4_K_M | qwen2 | 32,768 |
| `qwen2.5:7b` | 4.68 GB | 7.6B | Q4_K_M | qwen2 | 32,768 |
| `phi4-mini:latest` | 2.49 GB | 3.8B | Q4_K_M | phi3 | 16,384 |
| `llama3.2:3b` | 2.02 GB | 3.2B | Q4_K_M | llama | 131,072 |
| `qwen2.5-coder:3b` | 1.93 GB | 3.1B | Q4_K_M | qwen2 | 32,768 |
| `qwen2.5:3b` | 1.93 GB | 3.1B | Q4_K_M | qwen2 | 32,768 |
| `llama3.2:1b` | 1.32 GB | 1.2B | Q8_0 | llama | 131,072 |
| `nomic-embed-text:latest` | 0.27 GB | 137M | F16 | nomic-bert | 2,048 |
| `gemma4:31b-cloud` | 0 GB (stub) | 32.7B | BF16 | — | cloud |
| `minimax-m3:cloud` | 0 GB (stub) | — | — | minimax-m3 | cloud |
| `artiku348/loker.v2.1-cld-cdr-480b:latest` | 0 GB (stub) | — | — | — | unknown provenance |

**Installed, non-cloud, non-embedding:** 23. **MoE (fast on this box):** 2 — `qwen3.5:35b-a3b`, `qwen3:30b-a3b`. **Everything else is dense**, i.e. bandwidth-limited, i.e. slow above ~14B.

Not installed, contrary to a peer claim: **`usb-mistral-nemo-12b` does not exist on this host.** There is no Mistral model in the Ollama library here at all. (Also, Mistral-Nemo is 128K context, not 1M.) See §9.1.

vLLM: one model resident — `Qwen3-8B`, bf16 (~16 GB), 32,768 ctx cap.

---

## 3. Single-stream, this session (with the contention caveat that changes the reading)

Protocol: one prompt ("Write a Python function that dedupes a list preserving order"), `num_predict` 128, temp 0.2, `keep_alive: 5m`, models run sequentially with **no unload between them**. Ollama decode = `eval_count / eval_duration`. vLLM = completion_tokens / wall.

| Model | Wall | Load | Out tok | Decode tok/s (this run) | Clean-room figure | Content chars |
|---|---:|---:|---:|---:|---:|---:|
| `qwen3:30b-a3b` | 9.7 s | 7.9 s | 128 | **83.7** | 87.0 (repo prior) | **0** |
| `qwen3.5:35b-a3b` | 13.0 s | 7.9 s | 128 | 28.1 | 73.8–76.4 (repo prior) | **0** |
| `qwen2.5-coder:14b` | 8.0 s | 5.9 s | 49 | 24.1 | 23.7 (repo prior) | 205 |
| `gemma3:12b` | 23.3 s | 8.6 s | 120 | 8.4 | 27.5–27.9 (repo prior) | 415 |
| `deepseek-r1:32b` | 40.2 s | 10.2 s | 128 | 4.3 | 10.7 (repo prior) | **0** |
| `llama3.3:70b` | 39.6 s | 22.1 s | 70 | 4.1 | 4.8 (repo prior) | 170 |
| vLLM `Qwen3-8B` bf16 | 16.0 s | — (resident) | 128 | 8.0 e2e | 7.7–9.8 (peer session) | 512 |

**Read this table as a memory-pressure experiment, not a model ranking.** Only `qwen3:30b-a3b` (the first call) matches clean-room numbers. Everything after it degrades monotonically, because `keep_alive: 5m` kept each finished model resident while vLLM held its 54 GB: by the `gemma3` row the box had ~96 GB of weights + KV in one 121 GiB pool. `qwen3.5:35b-a3b` at 28.1 tok/s vs its known 74–76 is the likely signature of weights spilling off the fast path, not of the model being slow.

**Methodological rule this establishes for everyone using this repo:** back-to-back multi-model Ollama benchmarking without `ollama stop <model>` between runs measures residency, not speed. That is almost certainly why a peer session recorded `qwen3.5:35b-a3b` "crashing" with `TTFT=None` — same cause, same fix.

### 3.1 The finding that survives the contention

Three of six models returned **zero content** at a 128-token cap: `qwen3.5:35b-a3b`, `qwen3:30b-a3b`, `deepseek-r1:32b`. All three are reasoning models that spend the entire budget in the thinking channel. `finish_reason` is `length`, `content` is `""`, and a client sees an empty answer that looks like an outage. Peers measured the same thing ("thinking ate output budget") and it is the most common local failure mode in this repo's history.

**Rule:** any route fronting these three needs `max_tokens >= 1024`, or a reasoning-off flag, or the client will get silence at the default cap.

---

## 4. Concurrency — the actual answer to "why did we build vLLM"

### 4.1 This session

3 simultaneous requests, 64-token cap, same model:

| Target | Total wall | Per-request wall | Per-request tok/s | Reading |
|---|---:|---|---:|---|
| Ollama `qwen3:30b-a3b` | 27.4 s | 26.0 / 27.4 / 26.7 s | 77–90 | ~25 s of queueing/loading per request; generation itself was <1 s |
| Ollama `llama3.2:3b` | 14.6 s | 14.5 / 13.3 / 12.3 s | 39–47 | 3 requests of ~1.6 s generation took 14.6 s total → **~9× serialization overhead** |
| vLLM `Qwen3-8B` | — | — | — | **404 in this run — my harness bug, not a vLLM fault** |

The vLLM burst failure is worth documenting because it is a good 404: I resolved the model id with `label.startswith("vllm")` against the label `"vLLM qwen3-8b"` — Python string matching is case-sensitive, the branch never fired, and the literal string `vllm` went upstream. vLLM logged it precisely:

```
ERROR [serving.py:214] Error with model error=ErrorInfo(message='The model `vllm` does not exist.',
      type='NotFoundError', param='model', code=404)
```

**The same log contains the real evidence**, from the seconds before: `Running: 4 reqs, Waiting: 0 reqs` with `Prefix cache hit rate: 65.3%` — four requests decoded in parallel in one engine step, then throughput rose from 21.9 to 55.2 tok/s as the batch filled. That is continuous batching doing exactly what it exists for, and it is why the box needs vLLM at all.

### 4.2 Peer session, same host, same day

vLLM `Qwen3-8B`: TTFT 0.09–0.65 s, decode 7.7–9.8 tok/s single, **54.7 tok/s aggregate at 4 concurrent** → **5.6× throughput from 4 streams**, no per-request collapse.

### 4.3 Published same-host concurrency, for scale

| Source | Config | Single stream | Aggregate |
|---|---|---:|---:|
| field notes (Sergio Silva) | GPT-OSS-120B MXFP4, vLLM | 60.7 tok/s | — |
| NathanMaine | int4 AutoRound, vLLM, 131K ctx | 70 tok/s | **~350 tok/s @16 sustained** (~345 flat from 2.3M to 2.8M tokens) |
| ctala (Oct 2026 round) | Qwen3.6-35B-A3B NVFP4 + MTP, vLLM 0.31.0 | 102.3 tok/s | **240.0 @4 users**, 218 sustained 15 min, 0 failures |
| elsung (dual Spark TP=2) | DeepSeek-V4-Flash FP8 | ~38.5 | ~103 @c6 / ~270 @c32 / ~350 @c32 |
| sostrowsk | Laguna INT4, vLLM vs SGLang | — | 52.8 vs 45.3 @c4 (vLLM +17%) |

**Conclusion:** vLLM's advantage is *aggregate throughput under concurrency*, roughly 4–6× single-stream at c=4 and 8–9× at c=16. It is not, and never was, a single-request latency win. Building it was correct; judging it by one-client tok/s was the mistake.

---

## 5. Why the direct comparison is still not settled (be honest about this)

### 5.1 Precision differs — and this box is bandwidth-bound
vLLM ran **bf16** (16 GB of weights read per token). Ollama ran **Q4_K_M** (4.7–23.9 GB). At 273 GB/s, decode speed ≈ bandwidth ÷ bytes-per-token. Comparing them ranks quantization, not engines.

### 5.2 Reasoning tokens were not counted on the vLLM side
`Qwen3-8B` is a reasoning model. `completion_tokens=128` counted thinking; the visible answer was 512 chars. Wall-clock included uncounted reasoning. 8.0 tok/s understates it.

### 5.3 They compete for one pool of memory
`--gpu-memory-utilization 0.45` = ~54 GB reserved. A peer session's `qwen3.5:35b-a3b` run failed outright under that reservation. Co-residency needs the flag near **0.25–0.30** (~30–36 GB, enough for an 8B bf16 plus KV), or a larger model served *only* by vLLM while Ollama is idle.

### 5.4 What would settle it
Identical weights on both stacks in one session, `ollama stop` between runs, reasoning tokens counted separately. Candidates already in the HF cache: `Qwen2.5-14B-Instruct`, `Qwen3-8B`, `Qwen3-Coder-30B-A3B`, `Mistral-7B-Instruct-v0.3`, `nvidia/Qwen3.6-35B-A3B-NVFP4`. **Not run — that loads 20–30 GB of weights and was not pre-cleared with the operator.**

### 5.5 Published same-weights head-to-head (third party, one host)

From the `dgx-spark-field-notes` repo, which measures both stacks on the same box:

| Model | Ollama | vLLM | Winner |
|---|---:|---:|---|
| GPT-OSS-120B MXFP4 | 42.1 | **60.7** | vLLM +44% |
| Gemma-4-26B (MTP) | 49.6 | **54.9** | vLLM +11% |
| Qwen3-30B-A3B | **85.2** | 77.1 | Ollama +11% |
| qwen3-coder-next (80B-A3B) | 59.2 | **74.8** | vLLM +21% |

Same repo's TTFT comparison: `qwen3-coder-next` 215 → **104 ms**, GPT-OSS-120B 449 → **131 ms** (vLLM 2.1–3.4× lower). Its summary: *"llama.cpp beats vLLM wherever vLLM is stuck on 8-bit weights, and loses wherever [it isn't]."* That is the whole engine debate in one sentence, and it matches our 8 tok/s bf16 result exactly.

---

## 6. Published GX10 / GB10 numbers, so we stop re-measuring

All third-party; none of it re-run here. Sources at the end of each block.

### 6.1 Ollama's own release-day bench (DGX Spark, firmware 580.95.05, Ollama 0.12.6)

| Model | Quant | Prefill tok/s | Decode tok/s |
|---|---|---:|---:|
| gemma3 12B | q4_K_M | 1,894 | 24.25 |
| gemma3 27B | q4_K_M | 834 | 10.83 |
| llama3.1 8B | q4_K_M | 7,614 | 38.02 |
| llama3.1 70B | q4_K_M | 1,911 | 4.42 |
| deepseek-r1 14B | q4_K_M | 5,919 | 19.99 |
| qwen3 32B | q4_K_M | 705 | 9.41 |
| gpt-oss 20B | MXFP4 | 3,224 | 58.27 |
| gpt-oss 120B | MXFP4 | 1,169 | **41.14** |

*ollama.com/blog/nvidia-spark-performance*

### 6.2 NVIDIA's own inference table (ISL/OSL 2048/128, BS=1)

| Model | Precision | Backend | Prefill | Generation |
|---|---|---|---:|---:|
| GPT-OSS-20B | MXFP4 | llama.cpp | 3,670 | **82.74** |
| GPT-OSS-120B | MXFP4 | llama.cpp | 1,725 | **55.37** |
| Qwen3 14B | NVFP4 | TRT-LLM | 5,929 | 22.71 |

*developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks*

### 6.3 Consensus across independent measured repos

| Model class | Typical measured decode | Consensus |
|---|---:|---|
| `gpt-oss:20b` MXFP4 | 52–83 tok/s | **Best speed/quality compromise**; only model at 100% pass in both think and no-think modes (Kleybrink 24-model run) |
| `qwen3.5/3.6-35B-A3B` MoE Q4 | 49–85 tok/s (stock), 102 with vLLM NVFP4+MTP | **Best single model for this box**; also the ctala round's overall winner at 128K × 4 users |
| `gpt-oss:120b` MXFP4 | 40–61 tok/s, 60–80 GB | Best quality that still streams; 3.8× Qwen-27B's throughput (BridgeBench) |
| `qwen3-coder-next` 80B-A3B | 59–75 tok/s | Best coding agent, not installed here |
| `qwen3:30b-a3b` | 85–87 tok/s | Fastest everyday chat |
| `gemma4:26b-A4B` / `gemma3:12b` | 24–70 tok/s | Vision-capable, mid |
| Dense 27–32B | 7–13 tok/s | Rarely worth running |
| **Dense 70B** (`llama3.3:70b`) | **3.8–5.4 tok/s** (98% of the 273 GB/s roofline) | Best coding score in this repo's suite, worst experience. Batch only. |

Sources: `sergioamsilva/dgx-spark-field-notes` (model-selection + RESULTS), `Kleybrink/dgx-spark-bench` (21–24 model runs), `ctala.github.io/local-llm-agentic-workflows/benchmarks`, `pi3g.com` (AIPerf Pareto: Nemotron-3.5-Lightning-30B-A3B NVFP4 70 → 452 tok/s @c32), `llmrun.dev/device/asus-ascent-gx10`, `llmconfigurator.com/en/best-models/nvidia-dgx-spark`, `tokenstead.ai` (BridgeBench: Qwen 3.5 27B 11.1 tok/s / 76.3% vs GPT-OSS 120B 41.9 tok/s / 74.0%), `heitor-mocelin/dgx-spark-research` (70B at 5.4 tok/s = 98% roofline).

### 6.4 CUDA graphs and the driver table — CORRECTED

> **Correction (same day, 2026-11-12).** The first version of this section asserted "driver 580
> cannot capture CUDA graphs on GB10 at all" and called the driver upgrade "the highest-leverage
> fix" worth ~2.8×. **Both are retracted.** The claim does not survive contact with this host's
> own logs, and the arithmetic does not support the transfer. Details below.

`NathanMaine/NVIDIA-DGX-Spark-with-vLLM` measured, on this hardware class, **Qwen3-Coder-Next
80B-A3B MoE in int4-AutoRound (~41 GB)** — not an 8B dense model:

| Config | Single | 4 concurrent |
|---|---:|---:|
| Driver 580.126.09 + `--enforce-eager` | 25 tok/s | 67 tok/s |
| Driver **595.84** + CUDA graphs | **70 tok/s** | **149 tok/s** |

Two things that row pair does *not* establish:

1. **The 580 row deliberately turned graphs off.** `--enforce-eager` *is* the flag that disables
   CUDA graph capture. So 25 → 70 mostly measures eager → graphs, not 580 → 595. That repo's own
   README says so: "You can work around it with `--enforce-eager`, but that costs roughly 2.8×."
   The driver gate is real for *their* kernel paths — SM121a NVFP4/Marlin capture is reported
   broken under 580 upstream, and NVFP4 needs driver ≥590 — but it is not a property of the host
   or of bf16 kernels.
2. **Their 70 tok/s is not transferable to our model.** Active parameters decide decode speed:
   3B active at 4 bits = 1.5 GB read/token → 182 tok/s roofline, so 70 is 38% of it. Our
   Qwen3-8B is **dense 8.2B at bf16 = 16.4 GB/token → 16.6 tok/s single-stream roofline.** We
   measure 8.0 = 48% of roofline, which is ordinary efficiency. No driver lifts 8B bf16 past
   ~16.6 tok/s on 273 GB/s.

**Verified on this host** (vLLM 0.21.0, torch 2.11.0+cu130, driver 580.178.04, CUDA 13.0
ceiling): CUDA graphs **captured successfully** — `enforce_eager=False`,
`cudagraph_mode=FULL_AND_PIECEWISE`, 51 piecewise + 35 full graphs at 100%, `CUDA graph pool
memory: 9.17 GiB (actual)`, and no eager fallback anywhere in `/tmp/vllm*.log`. Capture works for
standard bf16 kernels on 580.

**Correct lever ranking for this box: quantization and active-parameter count ≫ driver.**
The real case for 595 is (a) NVFP4 weights, gated at driver ≥590, and (b) newer NGC containers
built for CUDA 13.2 (≥595.58; Ubuntu offers `nvidia-driver-595-server-open` 595.91.07 here).
It is not a 2.8× win against a bf16 8B, and it costs a reboot — operator's call, since
decided: **skip it** (§8.4 item 3).

### 6.5 Four numbering systems that get quoted in the same sentence

These get conflated constantly because blog posts list them side by side. They are unrelated
schemes, and none of them is a version of another:

| Number | What it actually is | On this box |
|---|---|---|
| `580.178.04` | **NVIDIA driver** — `branch.minor.build`. The first field is the release *branch*, shared by many separate builds (`580.95.05`, `580.126.09`, `580.159.03`, `580.173.02`, `580.178.04` are all branch 580). `595` is a **new branch**, not a point release. | 580.178.04 |
| `0.21.0` | **vLLM package** — ordinary semver, moving independently of the driver. | 0.21.0 |
| `13.0` / `13.2` | **CUDA** — the maximum runtime the driver supports, *not* what is inside a venv or container. | 13.0 ceiling |
| `26.02` / `26.04` | **NGC container tag** — `year.month`. Each tag pins one vLLM build to one CUDA version. | n/a (we run a venv) |

The only reason these appear together is a **dependency chain**, not a shared numbering:

```
driver branch  →  max CUDA version  →  which vLLM wheels / NGC tags can load
```

Reported worked example: driver 580.173.02 caps its forward-compat layer at CUDA 13.1, so the
newest NGC tag that runs is 26.02 (vLLM 0.15.1) — but Qwen3.5 support needs vLLM ≥0.19, first
shipped in 26.03+, built against CUDA 13.2, which in turn needs driver **≥595.58**. So the driver
gates *which vLLM you are allowed to run*; it does not multiply tokens per second on its own.
That distinction is exactly what the retracted claim above got wrong.

---

## 7. Where the numbers disagree, and who is probably right

| Claim | Source | Assessment |
|---|---|---|
| vLLM Qwen3-8B ≈ 8 tok/s | this session + peer session, independently | **Right for our config.** Confirmed twice. Not a verdict on vLLM |
| vLLM "typical 30–60+ TPS" desktop RTX 5070 | web-Grok, quoted in README | **Wrong hardware, wrong driver.** Not applicable to GB10 at 580.x |
| Qwen3.6-35B-A3B at 218–240 tok/s | spark-arena / ctala | **Different stack** (NVFP4 + MTP + CUDA graphs), not stock Ollama Q4. Do not compare to our 74–85 |
| `qwen3.5:35b-a3b` "crashed, TTFT=None" | peer session | **Contention**, not a model fault: vLLM's 54 GB + no unload between models |
| Dense 70B is worth running | general LLM folklore | **False here.** 4–5 tok/s and a 22 s cold load |
| "vLLM is the GPU engine so it must be faster" | prior docs' framing | **Mistaken framing.** vLLM wins aggregate, loses single-stream when it cannot use 4-bit |

---

## 8. Recommended dedicated model names on `http://localhost:4000/v1`

Currently `:4000` front-loads four vague groups (`ULTRA_FREE_TOOLS`, `ULTRA_FREE_TOOLS_FAST`, `ULTRA_FREE_LOCAL`, `ULTRA_FREE_CODING`) that mix vLLM and Ollama, and **at least one is broken**. Proposal: one name per *job*, each pinned to one engine, so a client can reason about cost and concurrency.

### 8.1 The five names worth having

```yaml
# 1. Default local chat/reasoning — the best model on this box
- model_name: gx10-chat
  litellm_params:
    model: ollama_chat/qwen3.5:35b-a3b
    api_base: http://localhost:11434
    api_key: local
    keep_alive: 24h
    num_ctx: 65536          # REQUIRED: env default is 16384 and truncates silently
    max_tokens: 2048        # REQUIRED: thinking models return "" at small caps
    rpm: 60

# 2. Coding — same MoE (12/14 in this repo's coding suite), coder-14b as fallback
- model_name: gx10-code
  litellm_params:
    model: ollama_chat/qwen3.5:35b-a3b
    api_base: http://localhost:11434
    api_key: local
    num_ctx: 32768
    max_tokens: 4096

- model_name: gx10-code
  litellm_params:
    model: ollama_chat/qwen2.5-coder:14b
    api_base: http://localhost:11434
    api_key: local
    keep_alive: 24h
    num_ctx: 32768

# 3. Cheap/instant — classification, routing, title generation, tool arg filling
- model_name: gx10-fast
  litellm_params:
    model: ollama_chat/llama3.2:3b
    api_base: http://localhost:11434
    api_key: local
    keep_alive: 24h
    num_ctx: 8192

# 4. Concurrency path — many clients, one engine step. This is what vLLM is for.
- model_name: gx10-multi
  litellm_params:
    model: hosted_vllm/<served-id-from-/v1/models>   # NOT "Qwen/Qwen3-8B"
    api_base: http://localhost:8000/v1
    api_key: local
    rpm: 600

# 5. Embeddings — currently unrouted; nothing on :4000 serves embeddings
- model_name: gx10-embed
  litellm_params:
    model: ollama/nomic-embed-text
    api_base: http://localhost:11434
    api_key: local
    keep_alive: 24h
```

Optional sixth, **batch only** (never a UI default): `gx10-batch` → `ollama_chat/llama3.3:70b`, `keep_alive: 10m`, `num_ctx: 32768`. 4–5 tok/s, 14/14 coding, 22 s cold load.

### 8.2 Names to avoid
- **`gx10-reason` on `deepseek-r1:32b`** — 4.3–10.7 tok/s and it spends the budget thinking. `qwen3.5:35b-a3b` reasons 8–20× faster.
- **Any name combining local + cloud** — breaks the fail-closed contract already documented in `FREE_LLM_SETUP.md`.
- **Duplicating `gx10-chat` under a third alias** (`ULTRA_FREE_LOCAL` + `ULTRA_FREE_CODING` already point at the same model).

### 8.3 Live bug this surfaces
`litellm_config.yaml:340` routes `ULTRA_FREE_TOOLS` to `hosted_vllm/Qwen/Qwen2.5-14B-Instruct` on `:8000`. vLLM is not serving that model any more — it serves `Qwen3-8B` under a snapshot **path** id. That upstream is dead; requests fall through to the two Ollama entries in the same group. Either repoint it or delete it.

### 8.4 To make the local path actually good
1. `OLLAMA_CONTEXT_LENGTH=32768` (or 65536) in the systemd drop-in — otherwise every model is capped at 16K regardless of catalog.
2. `--gpu-memory-utilization 0.28` on vLLM so the two engines coexist in 121 GiB.
3. ~~Driver 580.178.04 → 595.91.07~~ — **SKIPPED by operator decision, 2026-11-12.** It buys no measured throughput on the bf16 8B we actually run (see §6.4), its real benefits (NVFP4 kernels, NGC 26.03+ containers) are unverified *on GX10 hardware specifically*, and changing driver branch on a 121 GiB unified-memory box risks the one setup that currently works — for a reboot and an unmeasurable gain. Do not re-propose it without a measured NVFP4 result **on this exact host**, from someone who has already run it.
4. `OLLAMA_MAX_LOADED_MODELS=1` if running as a service, so servicing one request cannot evict another mid-flight.

> **Standing rule (operator, 2026-11-12):** if something is risky **and** not demonstrably supported
> on GX10 / GB10 hardware, **skip it** — do not try it "to see." Numbers from DGX Spark blogs,
> NVIDIA tables and other people's GB10 boxes are *leads to verify on this host*, never a reason to
> reconfigure a working host. This is why items 3 and the NVFP4 path are out of scope here, and why
> §5.4's same-weights head-to-head was listed as not-run rather than attempted.

---

## 9. Corrections to peer reports (same day, same box)

### 9.1 `usb-mistral-nemo-12b` — does not exist
A peer report recommends relying on "Mistral-Nemo 12B at 1M context already installed in Ollama (`usb-mistral-nemo-12b`)" as the truly-unlimited option. `ollama list` returns 26 models and **no Mistral of any kind**. Mistral-Nemo is also 128K context, not 1M. The correct unlimited+private claim is `qwen3.5:35b-a3b` (262K catalog, 16K as configured).

### 9.2 "OpenRouter key truncated → can't verify generation" — the key works
A peer report concludes the OpenRouter key is truncated and generation is unverifiable. It works: `/api/v1/key` returns a funded account (`limit 45`, `limit_remaining 42.61`, `is_free_tier: false` → 1,000 req/day), and ~20 `:free` routes are live through `:4000`. The peer read a truncated copy from `dbpasses.txt`; the running proxy holds the real one. Recommendation "get a real OpenRouter key" is unnecessary.

### 9.3 DeepInfra is not dead — the primary key is
Peer report lists DeepInfra under "rejected (dead keys)". The primary key is 401, but the `_ALT` and `_BEST` keys serve 177 models with real generations (`Ling-3.0-flash-VL`, `GLM-5.3`). It is **paid credit, not a free tier** — useful as a paid fallback, not as free capacity.

### 9.4 SambaNova "alive, 402" — confirmed, and it is dead
Independent agreement: 5-model catalog, every call `402 PAYMENT_METHOD_REQUIRED` with `balance_units: 0`. Docs still describe a no-card 20 RPM / 20 RPD tier; the console has required a payment method since ~Aug 2026. Drop it from any free pool.

### 9.5 Mistral's free tier is real (retraction of an earlier verdict by the same agent as this file)
`mistral-small-latest` returning 429 was read as "no API access". It is `x-ratelimit-limit-req-minute: 0` — that model is excluded from the free tier. `open-mistral-nemo`, `codestral-latest`, `ministral-3b/8b/14b` all return 200 at **2 req/min, 40K tok/min** on our existing keys, no card. Wired and verified through LiteLLM.

---

## 10. Reproduction

```bash
# inventory (cheap, no model loads)
ollama list; curl -s localhost:11434/api/tags | jq '.models[] | {name,size,details}'
ollama show <model> | grep -iE 'context length|quantization|architecture'
curl -s localhost:8000/v1/models; tr '\0' '\n' < /proc/$(pgrep -f 'vllm serve'|head -1)/environ

# single-stream, one model, WITH unload between (the step everyone skips)
for m in <models>; do
  ollama run $m --verbose "say ok" >/dev/null 2>&1   # warm
  curl -s localhost:11434/api/chat -d "{\"model\":\"$m\",\"messages\":[{\"role\":\"user\",\"content\":\"hi\"}],
    \"stream\":false,\"options\":{\"num_predict\":128,\"temperature\":0.2},\"keep_alive\":\"0s\"}"
done
```

**Operational warning, learned the hard way:** these commands load 4–24 GB of weights per model into a 121 GiB pool shared by vLLM. Run them one at a time, announce them, and unload with `keep_alive: "0s"` or `ollama stop`. A six-model sequential loop with a 5-minute keep-alive is enough to degrade every reading and starve a running vLLM.

---

## 11. Sources

Measured on this host: `ollama list` / `api/tags` / `api/ps` (2026-11-12 03:45Z), `/proc/<pid>` env, vLLM startup log and request log at `/tmp/vllm2.log`, single-stream + 3-way concurrency harness (2026-11-12 03:47–03:49Z), `nvidia-smi` (driver 580.178.04).

Third-party: `ollama.com/blog/nvidia-spark-performance` · `developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks` · `github.com/sergioamsilva/dgx-spark-field-notes` (README, `benchmarks/RESULTS.md`, `docs/model-selection.md`) · `github.com/Kleybrink/dgx-spark-bench` (2026-04-07 and 2026-05-04 REPORTs) · `ctala.github.io/local-llm-agentic-workflows/benchmarks` · `pi3g.com/dgx-spark-benchmarking-on-inference-performance-of-14-open-weight-models` · `github.com/NathanMaine/NVIDIA-DGX-Spark-with-vLLM` (`docs/BENCHMARKS.md`) · `github.com/sostrowsk/dgx-spark-local-llm` · `github.com/elsung/dgx-spark-deepseek-v4-flash` · `github.com/botAGI/dgx-spark-llm-benchmarks` · `github.com/heitor-mocelin/dgx-spark-research` · `github.com/pendakwahteknologi/gx10-benchmarks` · `llmrun.dev/device/asus-ascent-gx10` (estimates, not benchmarks) · `llmconfigurator.com/en/best-models/nvidia-dgx-spark` · `tokenstead.ai/guides/dgx-spark-benchmarks-2026` · `bestlocalairig.com/machines/asus-ascent-gx10` (estimates) · `verdictbits.com/reviews/dgx-spark-benchmarks` · `aimaserver.com/zh/benchmarks` · `spark-arena.com`.
