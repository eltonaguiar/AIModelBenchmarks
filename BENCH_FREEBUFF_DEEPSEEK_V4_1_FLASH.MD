# BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH

Author-agent: Freebuff (DeepSeek V4.1 Flash) · Date: **2026-11-11** · Host: Ubuntu ARM64 **NVIDIA GB10** (DGX Spark / ASUS Ascent GX10 class), 121 GiB unified memory, 20 vCPU, ~273 GB/s memory bandwidth, **62 GB free disk**.

Scope: our fleet split by **where the weights run** — locally / free cloud / paid cloud — plus which **free models are worth pulling onto this box**, how to measure them, and where our **rate-limit** ceilings actually are.

Every number is measured on this host through the fleet's own stack unless explicitly marked third-party. Method and limitations: §7.

---

## 1. Our current models, by category

### 1A. LOCALLY RUN (FREE) — weights on this machine

Warm short-prompt decode = Ollama `eval_count / eval_duration`; context = declared; coding = 14-check synthetic mini-suite.

| Model | tok/s | Context | Capabilities | Coding | Weights |
|---|---:|---:|---|---:|---:|
| `llama3.2:1b` | **163.6** | 131,072 | tools | 5/14 | 1.2 GiB |
| `qwen2.5:3b` | 94.9 (39–92 across runs) | 32,768 | tools, file read, live-fetch PASS | 7/14 | 1.8 GiB |
| `llama3.2:3b` | 93.8 | 131,072 | tools | 13/14, JSON-key fails | 1.9 GiB |
| `qwen3:30b-a3b` | 87.0 | 262,144 | tools, thinking | 0/14 (JSON fails ≥32K) | 17.3 GiB |
| **`qwen3.5:35b-a3b`** | **76.4** | **262,144** | **tools, vision, thinking** | **12/14**, live-fetch PASS | 22.2 GiB |
| `qwen2.5-coder:7b` | 46.2 | 32,768 | tools, insert | 9/14 | 4.4 GiB |
| `qwen2.5:7b` | 46.1 | 32,768 | tools | 8/14 | 4.4 GiB |
| `gemma3:12b` | 27.9 | 131,072 | vision, no tools | 10/14 | 7.6 GiB |
| `qwen2.5:14b` | 23.9 | 32,768 | tools | 11/14 | 8.4 GiB |
| `deepseek-r1:32b` | 10.7 | 131,072 | tools, thinking | 0/14 | 18.5 GiB |
| `qwen2.5:32b` | 10.5 | 32,768 | tools | 9/14 | 18.5 GiB |
| `llama3.3:70b` | 4.8 | 131,072 | tools | **14/14** | 39.6 GiB |
| vLLM `Qwen2.5-14B-Instruct` | 8.0 (e2e) | **8,192 hard cap** | tools **broken (HTTP 400)** | 5/14 | BF16 |

- **Speed** winner `llama3.2:1b` (163.6) — 5/14 coding. **Balance** winner `qwen3.5:35b-a3b` (76 tok/s, 262K, tools+vision). **Quality** winner `llama3.3:70b` (14/14 at 4.8 tok/s, 33× slower).
- **Every fast local row is MoE.** Every dense row ≥14B is ≤24 tok/s. That is the GB10 rule.
- **Local context ceiling is latency, not metadata:** one 261,652-token local retrieval took **164.3 s** (~1,593 tok/s prefill). Near-1M locally is not interactive.
- vLLM advertises 8,192 total and has no tool parser — the weakest local member despite being "the GPU engine".

### 1B. FREE (CLOUD) — zero token price

| Route / model | tok/s (e2e) | Context | Capabilities | Strict $0? |
|---|---:|---:|---|---|
| **Groq `openai/gpt-oss-120b`** (free plan) | **268.3 / 301.4** | **131,072** | tools-capable 120B MoE (5.1B active), 65,536 max out | yes, if pinned |
| `nemotron-3.5-lightning:free` (via `free-HighTPS_Context`) | 51.8 mean | **1,000,000** | text, tools | yes |
| `nemotron-3.5-super-vl:free` (same alias) | 20.4 | **1,000,000** | vision at backend, tools | yes |
| Gemini 3.5 Flash (AI Studio) | not valid at 220-token cap | 1M class | vision, tools, grounding | yes (card-free) |
| Cerebras `gpt-oss-120b` | — | — | — | **no — HTTP 402**, free tier gone |
| NVIDIA NIM `gpt-oss-120b`, `deepseek-v4-pro` | — | — | — | **no — 410 Gone**, retired |

> Our existing free picker entry (`free-HighTPS_Context`) is the **slowest** free option we can reach. Groq free is **5.2–5.8×** faster but caps at **131K**, not 1M.

### 1C. PAID (CLOUD)

| Model | tok/s (e2e) | Context | Capabilities | Caveat |
|---|---:|---:|---|---|
| **Mercury-3** | **547.8** | 1,000,000 | text, tools | fastest paid we run |
| Grok 4.7 | 98.7 | 500,000 | vision, tools, web | half the context |
| DeepSeek V4 Pro | 61.9 | 1,048,576 | text (vision withheld), 393,216 max out | reasoning ate the 220-token budget |
| Kimi K3.1 Pro | 33.6 | 1,048,576 | text, tools | needs `temperature: 1`; 217/220 reasoning |
| Unmeasured: Claude Haiku 4.5, GPT-5.5-instant, GPT-5-chat, MiniMax-M2, Ring 2.6 1T | — | — | — | no same-harness numbers |

---

## 2. Our routes as configured — and which are actually strict $0

`simple-shuffle`, `num_retries 2`, `allowed_fails 1`, `cooldown 65s`. Fallbacks are declared per group, so "free" in a name is not a guarantee.

| Alias (LiteLLM :4000) | Deps | Configured RPM | Behind it | Verdict |
|---|---:|---:|---|---|
| `free-mode` | 37 | **1 / 5 / 10 / 15** per dep | Groq 8b, Gemini 2.5-flash, NIM, Cohere, Fireworks kimi | **not strict** — falls back to `paid-mode` |
| `free-mode-fast` | 5 | 10 | Groq gpt-oss-120b/20b, Gemini flash-lite, Cohere r7b | **not strict** — falls back to paid |
| `ROCKETSPEED-FREE` | 5 | 10 | same as above | **not strict** — falls back to ROCKETSPEED-FAST (paid) |
| `ROCKETSPEED-FAST` | 7 | 5 | + DeepSeek-chat, Grok-4-fast | paid-capable by design |
| `free-mode-large` / `hybrid-model-large` | 14 / 13 | 1 / 5 / 10 / 15 | Gemini 2.5-flash/pro, Fireworks kimi | **not strict** |
| `hybrid-model` | 5 | 5 | Gemini flash, Groq llama, NIM, gpt-4o-mini, gpt-5.5 | mixed |
| `free-mode-tools` | 10 | — | Groq llama-3.1-8b, Mistral small, gpt-4o-mini, gpt-5.5 | mixed |
| **`free-HighTPS_Context`** | 2 | — (0-price enforced per request) | Nemotron Lightning + Super-VL `:free` | **strict $0, no fallback** |
| **`true-free-tools`** | 3 | — | vLLM Qwen2.5-14B, Ollama qwen2.5:14b, qwen3:30b-a3b | strict free (local only) |
| **`true-free-tools-fast`** | 3 | — | Ollama qwen2.5:3b, llama3.2:3b, qwen2.5:7b | strict free (local only) |
| `paid-mode` | 25 | 5,000 / 3,000,000 TPM | Claude Haiku, DeepSeek-chat, MiniMax-M2, gpt-4o-mini | paid |
| `paid-mode-fast` / `paid-mode-large` | 6 / 6 | — | Grok-4-fast, Claude Haiku, gpt-5-chat, gpt-5.5 | paid |
| `XX_*` single-provider probes | 1 each | — | groq qwen3.8-27b, gemini 2.5-flash (rpm 5), deepinfra Qwen3.8-Flash, mistral nemo, nous glm-5.3-flashx, cloudflare llama-3.3-70b, ollamacloud gpt-oss:120b-cloud | measurement aliases |

**Consequence:** "free-mode-fast" is fast *when it lands on Groq* and 10 dep-RPM limits mean ~1 request every 6 s; it is not a $0 contract. Only `free-HighTPS_Context` and the two `true-free-tools*` groups are.

---

## 3. Rate-limit ceilings — the real bottleneck

| Provider | Free ceiling (dated) | Our observation |
|---|---|---|
| Groq | 30 RPM, 1,000 RPD, **8,000 TPM**, 200k TPD per model (gpt-oss-120b/20b, qwen3.8-27b) | 268–301 tok/s; TPM is the binding cap for real prompts |
| OpenRouter `:free` | 20 RPM, 50 RPD (<$10 lifetime credits), 1,000 RPD after | what `free-HighTPS_Context` runs on |
| Gemini API | per-model RPM/TPM/RPD in AI Studio, unpublished | short call returned 9 tokens (thinking ate budget) |
| Cohere trial | 20 req/min per chat model, **1,000 calls/month**, non-production | in `free-mode*` and `ROCKETSPEED-FREE` |
| Cloudflare Workers AI | 10,000 neurons/day, resets 00:00 UTC | quota exhaustion seen previously |
| NVIDIA NIM | 40 RPM free endpoints | catalog churn: 410/404 today |
| Cerebras | trial credit only, card required | **402** on our key |
| Mistral | free mode, no card | **429** today |
| **Ollama (local)** | `OLLAMA_NUM_PARALLEL=1` → **serialized**, one model at a time; `KEEP_ALIVE=-1`, flash-attn on, KV cache **q4_0** | no RPM cap, but concurrency 1 is a hard queue |
| vLLM (local) | single model, 8K total context | queue depth unmeasured |

Local "unlimited" is misleading: at `NUM_PARALLEL=1` a 76 tok/s model serving 2 requests behaves like ~38 tok/s each. Free cloud RPM matters more than raw TPS once you run agents with tool loops.

---

## 4. Free models worth pulling onto this box (verified tags, live-checked)

`python3 tools/local_candidate_bench.py --plan` — GET-checks each tag, then compares download×1.15 against free disk (62 GB) and free RAM (48.6 GB at run time).

| Tag | Params | Download | Declared ctx | Caps | Fit today | Published GB10 speed (third-party) |
|---|---|---:|---:|---|---|---:|
| `gpt-oss:20b` | 20.9B MoE / 3.6B active | 13 GB | 131K | reasoning, tools | **ok** | 83 tok/s |
| `gemma4:26b` | 26B MoE / ~3.8B active | 18 GB | 262K | **vision**, tools | **ok** | 59.1 tok/s |
| `qwen3.6:27b` | 28B dense | 19 GB | 262K | **vision**, tools, thinking | **ok** | 12–13 tok/s |
| `qwen3-coder:30b` | 30B MoE / 3.3B active | 19 GB | 262K | tools | **ok** | 61 tok/s |
| `glm-4.7-flash:latest` | ~32B class | 20 GB | 131K | tools, thinking | **ok** | 48.7 tok/s |
| `gemma4:31b` | 31B dense | 22 GB | 262K | vision, tools | **ok** | dense — slower than 26B |
| `qwen3-vl:8b` | 8B dense | 6 GB | 262K | **vision/OCR**, tools | **ok** | small, fast |
| `gpt-oss:120b` | 116.8B MoE / 5.1B active | 65 GB | 131K | reasoning, tools | **TIGHT (disk)** | 52.9–58.7 llama.cpp, 60.7 vLLM, 42.1 Ollama |
| `nemotron-3.5-super:120b` | 124.5B MoE / 12B active | 72 GB | **1.05M** | **vision**, tools, thinking | **TIGHT** | ~25 tok/s |
| `qwen3.5:122b-a10b` | 125B MoE / 10B active | 79 GB | 262K | tools, thinking | **TIGHT** | ~29 tok/s |

**Order I would pull them in:**
1. **`gpt-oss:20b` (13 GB)** — fits today, tools + reasoning, ~83 tok/s class; the cheapest capability jump from a 3B.
2. **`gemma4:26b` (18 GB)** — the only local path to *vision + tools* at usable speed; 262K context.
3. **`qwen3-coder:30b` (19 GB)** — code-shaped tool work at MoE speed.
4. **`gpt-oss:120b` (65 GB)** — the real prize, but **blocked by disk (62 GB free)**, not RAM. Clear ~15 GB first (HF cache is 158 GB), then pull.
5. `nemotron-3.5-super:120b` last — it is the only *local* near-1M option, but ~25 tok/s and 72 GB make it a specialist.

Do **not** bother with `qwen3.6:27b` for throughput: 28B dense on 273 GB/s ≈ 12 tok/s, i.e. worse than our current 35B MoE by 6×, whatever its coding score.

### Measured with the new harness (validation run, this host)

`tools/local_candidate_bench.py --model qwen2.5:3b --model llama3.2:3b --tiers 8192`

| Model | declared ctx | decode tok/s | e2e tok/s | tools | vision | file read | coding | JSON contract | 8K fill |
|---|---:|---:|---:|---|---|---|---:|---|---|
| `qwen2.5:3b` | 32,768 | 39.4 (92.1 in an earlier run) | 39.1 | PASS | n/a | PASS | 3/3 | **PASS** (alpha/charlie/echo) | **7,944 tok, 1.51 s** |
| `llama3.2:3b` | 131,072 | 92.7 | 91.9 | PASS | n/a | PASS | 3/3 | **FAIL** (returned `delta` for `end`) | 7,952 tok, 1.61 s |

What this already shows: both 3B models pass tools, file reading and all three coding tasks; only qwen2.5:3b survives the strict JSON contract. And the 39-vs-92 swing on the same model is why single-call TPS rankings are noise on a shared host.

### How to benchmark a candidate yourself

```bash
python3 tools/local_candidate_bench.py --plan                      # fit + tag check, no inference, no pulls
ollama pull gpt-oss:20b
python3 tools/local_candidate_bench.py --model gpt-oss:20b --tiers 8192,32768
python3 tools/local_candidate_bench.py --model gpt-oss:20b --model gemma4:26b --tiers 8192,32768,131072 \
    --out reports/candidates.json --md reports/candidates.md
```

It reports: warm decode TPS (3 calls, median), e2e TPS, prefill TPS, real context fill (actual `prompt_eval_count`, re-scaled until ≥85% of the requested tier), tool-call emission (`web_search` schema → this is the proxy for *web browsing*, since a local model can only browse via a tool), vision (real 1px PNG upload), file reading (5-function synthetic module), coding (3 tasks), and strict-JSON output. `--plan` never pulls or infers.

---

## 5. Cross-check by other AIs

Full three-category inventory + gap questions sent to providers via `consult_multi` (keys read from the private credential file; no values stored or printed).

| Voice | Result | Useful signal |
|---|---|---|
| **Groq · gpt-oss-120b** | answered | agreed MoE/35B-A3B is the local upgrade path; invented "Llama 3.2 8B at 125 tok/s" and "Mistral-7B free 1M ctx at 68 tok/s" — **rejected, unsupported** |
| **DeepInfra · Llama-3.1-8B** | answered | proposed Gemma 4 26B and correctly criticised comparing categories across different hardware; its "missing paid model = Mercury-3" is wrong (deployed) |
| **Google · Gemini 3.5 Flash** | partial | called GB10/DGX Spark "fictional" — **rejected**; `nvidia-smi` reports NVIDIA GB10 |
| Cerebras / NVIDIA NIM / Mistral / Nous / Together / GitHub Models | failed | 402 / 410+404 / 429 / 404 / 401 / DNS — six cross-check attempts blocked by account state, not by the question |

**Consensus:** MoE in the 26–120B class is the efficiency answer on GB10; `qwen3.5:35b-a3b` is a sound default but not the ceiling. **Open disagreement:** next dollar to Cerebras (speed) vs Groq paid (cheaper, 480 tok/s). My call — Groq paid first.

Two of three responding models produced confident, checkable errors. Single-model AI audits are hypotheses; the harness is the evidence.

---

## 6. Notes on `BENCH_GITHUBCOPILOT_GROK.MD` (the other file in this repo)

Read in full. Where we agree:

- Same three-way split; same "free in the alias name ≠ strict $0" warning.
- Groq free is the speed answer (their 450–1,000 tok/s is Groq's *advertised* LPU figure; **our measured value on this box is 268–301 tok/s e2e**, which is the number to plan capacity against).
- Cerebras is the notable paid speed tier; Nemotron Super-VL is the long-context pick; Qwen3.6-class MoE is the local upgrade direction.

Where it needs correcting:

1. **Hardware.** It says vLLM runs on a *desktop RTX 5070 12 GB* at "typically 30–60 tok/s". vLLM actually runs on this **GB10** box, BF16, hard-capped at **8,192** tokens, tools returning HTTP 400, measured **8.0 tok/s e2e**. The RTX-5070 description is stale text inherited from the old benchmark scripts.
2. **Storage.** It calls GX10 storage "not a limiter (1–4 TB, 5–10 large models)". This box has **62 GB free of 916 GB**, so `gpt-oss:120b` (65 GB) does not fit today. Storage, not RAM, is the blocker.
3. **Qwen3.6-35B-A3B is not an Ollama pull.** The 35B-A3B NVFP4 build it recommends is a vLLM/Hugging Face artifact; the Ollama tag that exists is `qwen3.6:27b` (dense, ~12 tok/s here). Pull path matters more than the model name.
4. **Its 90–110 tok/s figure for 35B-A3B assumes vLLM + MTP/speculative decoding.** Our Ollama `qwen3.5:35b-a3b` measures 76 tok/s with no speculation — same ballpark, different engine. Comparing the two as "the model's speed" hides an engine decision.
5. **No rate-limit matrix and no measurement of Groq/Cerebras/Gemini/NIM in its file** ("keys not accessed"). §3 above fills that gap with dated ceilings, and we measured the free providers directly.

Net: its recommendations are directionally right and its speed numbers are advertise-sourced; treat them as ceilings, not expectations.

---

## 7. Method and limitations

- **Speed:** one prompt ("Print the integers 1 through 200 separated by commas…"), temperature 0, 220-token cap, 2–3 calls/model. TPS = completion tokens / wall seconds, **including** queue, prefill and network — not decoder-only streaming.
- **Axes are not identical:** Ollama local rows are native decode (`eval_count/eval_duration`); cloud rows are end-to-end. Do not compare 76.4 local decode with 268 cloud e2e as one axis — the local e2e figure on the same prompt was 72.8.
- **Shared host:** a 3B model swung 39→92 tok/s between runs. Medians over 3 calls, not single readings.
- **Context figures are advertised metadata** except the retrieval tests: 261,652 local; 778,824 and 983,054 on the free route — all actual `prompt_eval_count`.
- **Coding** = 14-check synthetic mini-suite (and 3 tasks in the new harness). Ranks these models against each other, not a coding leaderboard.
- **GB10/provider third-party numbers** are cited, labelled, and not our measurements.
- **Free-tier ceilings change weekly** (Cerebras free tier is already gone; NIM retired two models we had configured). Re-check before building on any of them.

## 8. Reproduction and evidence

```bash
python3 tools/local_candidate_bench.py --plan
python3 tools/local_candidate_bench.py --model qwen2.5:3b --tiers 8192
python3 /tmp/bench_free.py                # free-cloud vs local, same prompt (writes the JSON below)
.venv/bin/python -m unittest discover -s tests -p test_free_high_tps_context.py -v
python3 tools/consult_multi.py --provider groq --model openai/gpt-oss-120b --prompt-file /tmp/bench_consult.md
```

Evidence on the host: `reports/local_model_benchmark_2026-11-11.md`, `reports/free_high_tps_context_2026-11-11.json`, `reports/model_bench_three_categories_2026-11-11.json`, `reports/local_candidate_bench_validation_2026-11-11.json`. No credential values appear in this file.

---

## 9. UI and engine layer — what sits in front of the models

Complements §2/§3: the router is not the only path a request can take on this box.

| Layer | What it is | Limits found (measured 2026-11-11) | Consequence |
|---|---|---|---|
| **Open WebUI 0.9.5** (:3000, docker) | chat UI + RAG/tool orchestration | `OPENAI_API_BASE_URL` is **empty**; `OLLAMA_BASE_URL=http://192.168.2.52:11434` (that IP **is** this host, `gx10-c9b9`) | **Open WebUI chats bypass LiteLLM entirely.** No fallback, cooldown, RPM accounting or strict-$0 guard applies there, and only Ollama models are selectable |
| RAG embedding | in-container `sentence-transformers/all-MiniLM-L6-v2` | not our local `nomic-embed-text` | document retrieval is a separate stack from the chat models — not covered by any figure above |
| **vLLM 0.21.0** (:8000) | GPU inference | `--max-model-len 8192 --gpu-memory-utilization 0.35 --port 8000`; tools broken (HTTP 400) | only ~35% of the 121 GiB unified pool is claimed → the 8K cap and 8 tok/s e2e are a **config ceiling, not a model ceiling** |
| **Ollama 0.40.13** (:11434) | local runtime | `NUM_PARALLEL=1`, `KEEP_ALIVE=-1`, `FLASH_ATTENTION=1`, `KV_CACHE=q4_0` | inference is **serialized** (one request at a time per model) and loaded models hold RAM indefinitely — 48.6 GB free while models were resident |
| **LiteLLM** (:4000) | router | repeated non-fatal `This model isn't mapped yet. model=openai/gpt-oss-120b` during model-info lookup; requests still return 200 | cost and context metadata cannot be resolved for that id, so **cost accounting and pre-call context checks are silently unavailable** for it |

Two practical consequences:

1. **Interactive work in Open WebUI is ungoverned.** If $0, fallback policy or rate accounting matter, call :4000 directly rather than the UI.
2. **vLLM is under-allocated.** At 0.35 utilisation it cannot serve a larger window or a tool parser; raising utilisation and adding `--enable-auto-tool-choice --tool-call-parser` would materially change Category A. We changed nothing here — this is a measured observation, not a recommendation to edit a shared service.

---

## 10. Notes on `BENCH_MODEL_PERFORMANCE_SUMMARY.md` (third file in this repo)

This one is a **digest**, not a measurement run: it restates figures from the earlier session summaries and adds no new tests. Its framing is sound and its reasoning caveat for Kimi/DeepSeek is correct. Specific corrections:

1. **The local category is effectively empty** (one row, `Qwen2.5-14B`, TPS "N/A"). This host runs **22 local generation models**; the measured spread is 163.6 tok/s (`llama3.2:1b`) down to 4.8 tok/s (`llama3.3:70b`), with `qwen3.5:35b-a3b` at 76.4 tok/s / 262K context as the balanced pick. See §1A — "local = Qwen2.5-14B only" understates Category A by roughly an order of magnitude in capability, and hides that the best local model is 9.5× faster than the vLLM one it lists.
2. **"Paid Cloud (Moonshot) 5,000 RPM / 3M TPM" is our router config, not Moonshot's limit.** 5,000 RPM and 3,000,000 TPM are the `paid-mode` group's LiteLLM deployment parameters in `litellm_config.yaml`. They are an internal throttle we set, not a ceiling the provider grants — presenting them as provider capacity is the kind of number that quietly becomes a planning assumption.
3. **"Free Mode (Gemini) 1–15 RPM, 65s cooldown"** mixes three different things: per-deployment `rpm` values from our config (1/5/10/15), the router's 65 s cooldown after failures, and Gemini's own unpublished per-model limits. Only the first is ours; the cooldown is not a rate limit.
4. **Groq free is missing entirely** — the single largest free-cloud fact available: **268–301 tok/s e2e measured**, 30 RPM / 8k TPM / 1k RPD / 200k TPD per model, 131K context. Its table therefore ranks Nemotron Lightning as the free-cloud speed leader when it is in fact ~5.5× slower than Groq.
5. **`free-mode*` is presented as "free mode (Gemini)"**, but that group has 37 deployments and **falls back to `paid-mode`**. Calling it free is exactly the misreading §2 warns about.
6. Minor: Grok 4.7 is labelled "text → text" — it is our only paid entry with vendor vision + web-search tools configured. Mercury-3 genuinely is text-only.

What is worth keeping from it: the three-category structure, the 2.54× Lightning-vs-Super-VL ratio (matches our 51.8 / 20.4), and the explicit warning that Kimi/DeepSeek TPS is reasoning throughput rather than answer throughput.

---

## 11. Free-tier key audit: every LLM provider in the credential file

Method: mapped each LLM label in the private key file to its provider endpoint (labels only — no values are printed, stored or committed), read the live model catalog with the key, then sent **one** `max_tokens: 8` probe to a free-class model to prove access. A catalog listing alone is not proof: Cerebras lists models and still bills/refuses.

**Rule applied (per operator):** a *recurring* allowance that replenishes qualifies. A one-time signup credit, a $5 trial, or a "valid 30–90 days" grant is **disqualified**, even when it is generous, because it does not replenish.

### 11A. Qualified — recurring free that actually answered

| Provider | Free mechanism | Probe result | Free models usable now |
|---|---|---|---|
| **Groq** | Free plan, recurring: 30 RPM / 1k RPD / 8k TPM / 200k TPD per model | **OK** — `openai/gpt-oss-120b`, 0.16 s | `gpt-oss-120b`, `gpt-oss-20b`, `qwen3.8-27b`, `qwen3.6-27b`, `gpt-oss-safeguard-20b`, `allam-2-7b` |
| **OpenRouter `:free`** | Recurring daily quota: 20 RPM, 50 RPD, **1,000 RPD** because the key has lifetime credit | **OK** — `nemotron-3.5-lightning:free` and `cohere/north-mini-code:free`, both **cost 0** | 13 `:free` models incl. `nemotron-3.5-lightning`, `nemotron-3.5-super-vl`, `nemotron-3-ultra-550b-a55b`, `cohere/north-mini-code`, `baidu/ernie-5.5-flash-preview`, `recursive/flint-v1.0`, `liquid/lfm-3-1.7b` |
| **Google Gemini API** | Free tier on Flash/Gemma models, no card, recurring | **OK** — `gemma-4-31b-it` answered (33 s, thinking-heavy) | 63 catalog entries; free class includes `gemma-4-31b-it`, `gemma-4-26b-a4b-it`, `gemini-2.5-flash`, `gemini-flash-lite-latest` |
| **Mistral** | Free "experiment" tier, no card, recurring | **OK** — `ministral-3b-latest`, 0.42 s | `ministral-3b/8b/14b`, `mistral-small`, `open-mistral-nemo` |
| **Cohere (trial)** | Trial key: **1,000 calls/month**, 20 RPM per chat model, recurring but non-commercial | **OK** — `command-a-03-2025` | Command A family, Command R/R7B |

### 11B. Disqualified — one-time credit, dead credential, or metered

| Provider | Why disqualified |
|---|---|
| **Cerebras** | `HTTP 402 Payment required`. Free tier is now a **$5/30-day trial** with card; our credit is spent and does not replenish |
| **NVIDIA NIM** | Free endpoints are "prototyping" credits, and both probes **timed out at 70 s** while the catalog read fine. Even if revived, treat as evaluation-only, not standing capacity |
| **SambaNova** | Catalog read OK (5 models) but chat returned **HTTP 401 "Incorrect API key"** — the key is inconsistent between endpoints. Its documented free tier (20 RPM/20 RPD/200k TPD) would qualify **if** a working key existed |
| **Nous Research** | Label says "USE ONLY FREE MODELS", but the probe returned `cost: 3.24e-05` with `cost_details.source: openrouter_cost_floor` — **it bills at OpenRouter floor rates**. Not free, and the label is misleading |
| **Hugging Face** | Probe worked but `estimated_cost` was charged; free Inference Provider credits were removed in Oct 2026. Metered |
| **DeepInfra** | Probe worked, `estimated_cost 2.8e-06`. Metered, no recurring free allowance |
| **Featherless** | Subscription only (chat $25/mo, developer $50 credits/mo). The 22,554-model catalog is readable but **nothing is callable without a paid plan** — not a free tier, and no probe was possible |
| **SiliconFlow** | `HTTP 401 Token is invalid`. Its free models (xing4.0-29b-a4b, qwen3-8b, glm-4-9b…) need mainland-China real-name verification plus a ¥14 signup credit — one-time **and** geo-restricted |
| **ModelScope** | `HTTP 401 invalid token` on DeepSeek V4.1 Flash, despite a 6M-token/day free allowance being reported. Unusable credential |
| **Together** | `HTTP 401 Unauthorized`; no free tier (trial credit discontinued) |
| **Fireworks** | `HTTP 412 account suspended` (monthly spend limit). Its "free" label refers to a signup credit |
| **Kimi / Moonshot** | `HTTP 401 Invalid Authentication`; no standing free tier |
| **Scaleway** | `HTTP 403 insufficient permissions`; offers are **1M free tokens per model** — a time-boxed grant, disqualified by your rule |
| **Alibaba DashScope / Qwen** | 1M tokens per model valid **90 days** — one-time grant, disqualified. (The `QWEN_API_URL` label is a URL, not a key) |
| **Nebius Token Factory** | $1 trial, 30 days, **card required** — disqualified |
| **xAI, Inception, DeepSeek, Hypereal, Novita** | Paid only; no recurring free allowance |
| **Hyperbolic** | Endpoints **decommissioned** (404 on Llama-3.3-70B) |
| **GitHub Models** | Retired (endpoint unresolvable) |
| **Cloudflare Workers AI** | *Qualifies in principle* (10,000 neurons/day, recurring) but the stored value is an account-level key with no account ID wired up, so no probe was possible — **blocked, not disqualified** |

### 11C. Free models we already hold but are not routing

1. **`gemma-4-31b-it` and `gemma-4-26b-a4b-it` are free on the Gemini API.** That is a 31B-class, 262K-context, vision-capable model at $0 recurring — currently unused. The strongest single addition found in this audit.
2. **`nemotron-3-ultra-550b-a55b:free`** on OpenRouter (1M context, frontier-scale reasoning) — we only route the two smaller Nemotrons.
3. **`cohere/north-mini-code:free`** — a code-specialised free model, verified cost 0, distinct from our trial-key Cohere route.
4. **`baidu/ernie-5.5-flash-preview:free`, `recursive/flint-v1.0:free`, `liquid/lfm-3-1.7b:free`** — 1M-context / small-and-fast free options, all cost 0.
5. **Groq `qwen3.8-27b`** — our alias `XX_groq` points at `groq/qwen/qwen3.8-27b`, which is a **free-plan model**. Same for `XX_gemini` (`gemini-2.5-flash` free tier). Two aliases we treat as probes are already $0.

### 11D. Not in the credential file — new recurring-free options worth a signup

- **LLM7.io** — free token, **100k tokens/day**, 1 r/s / 60 rpm / 250 rph, no card. `turbo` models only.
- **OpenCode Zen** — ~10 free models including DeepSeek V4.1 Flash Free, GLM-5.3-Flash Free, Nemotron 3.5 Lightning/Super-VL; Zen key required; free prompts may be used for training.
- **Kilo** — `qwen/qwen3.8-27b:free` (262K) with no key required.
- **Cloudflare Workers AI** — fix the account-ID wiring on the key we already hold and 10k neurons/day becomes usable.

### 11E. Credential-health anomalies (not free-tier questions, but found while auditing)

- Same-label key inconsistency at **SambaNova**: catalog accepts it, chat rejects it.
- **Nous** label instructs "free models only" but the gateway prices every call — a policy label, not a $0 guarantee.
- Two labels in the file are not keys (`QWEN_API_URL`, and the Ollama-cloud slot), and **Brave/Tavily/Exa/Firecrawl** search keys are outside this audit's scope but separately known to be broken or capped.

**Bottom line:** of **28** LLM providers with a key in the file, **5 offer recurring free capacity that answered today** (Groq, OpenRouter `:free`, Gemini free tier, Mistral, Cohere trial), **1 is blocked on configuration** (Cloudflare), and **22 are disqualified** — mostly because "free" meant a one-time credit that will not come back.
