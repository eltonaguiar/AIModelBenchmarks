# BENCH_OPENCODE_NEMOTRON_HIGH_TPS_CONTEXT

Author-agent: opencode (NVIDIA Nemotron 3.5 HighTPS Context route) · Date: **2026-11-11** · Host: Ubuntu x86_64, LiteLLM :4000 / vLLM :8000 / Ollama :11434.

Scope: break our measured model fleet into **four explicit quadrants by where the weights run and whether we measured them** — then answer: *are there more performance-efficient models we have not adopted, and under which category?* Metrics: **capabilities, tokens/sec, context window**.

**Free-model disqualification rule:** if a provider's free tier is a temporary credit that runs out and does not replenish, it is disqualified (**DISQUALIFIED — temporary credit**) and documented below. We only pursue nearly/unlimited free tiers.

All numbers are measured through our own harness unless marked third-party. This file sits next to `BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH.MD` (Freebuff) in the same repo; see Section 6 for my read of it.

---

## 1. Our current models, split by quadrant

### 1A. LOCALLY RUN (FREE) — weights on this machine

| Model (Ollama unless noted) | tok/s | Context | Capabilities | Coding | Size |
|---|---:|---:|---|---:|---:|
| `llama3.2:1b` | **163.6** | 131,072 | tools | 5/14 | 1.2 GiB |
| `qwen2.5:3b` | **94.9** | 32,768 | tools, live-fetch | 7/14 | 1.8 GiB |
| `llama3.2:3b` | 93.8 | 131,072 | tools | 13/14 (fails JSON-key contract) | 1.9 GiB |
| `qwen3:30b-a3b` | 87.0 | 262,144 | tools, thinking | 0/14 (JSON fails ≥32K) | 17.3 GiB |
| `qwen3.5:35b-a3b` | **76.4** | **262,144** | **tools, vision, thinking** | **12/14**, live-fetch PASS | 22.2 GiB |
| `qwen2.5-coder:7b` | 46.2 | 32,768 | tools, insert | 9/14 | 4.4 GiB |
| `qwen2.5:7b` | 46.1 | 32,768 | tools | 8/14 | 4.4 GiB |
| `gemma3:12b` | 27.9 | 131,072 | vision (no tools) | 10/14 | 7.6 GiB |
| `qwen2.5-coder:14b` | 24.0 | 32,768 | tools, insert | 10/14 | 8.4 GiB |
| `qwen2.5:14b` | 23.9 | 32,768 | tools | 11/14 | 8.4 GiB |
| `deepseek-r1:32b` | 10.7 | 131,072 | tools, thinking | 0/14 | 18.5 GiB |
| `qwen2.5:32b` | 10.5 | 32,768 | tools | 9/14 | 18.5 GiB |
| `llama3.3:70b` | 4.8 | 131,072 | tools | **14/14** (best quality) | 39.6 GiB |
| vLLM `Qwen2.5-14B-Instruct` | 8.0 (e2e) | **8,192 (hard cap)** | tools **broken (HTTP 400)** | 5/14 | BF16 |

Local read: speed = `llama3.2:1b` (163.6 tok/s) but weak coding; balanced = `qwen3.5:35b-a3b` (76 tok/s, 262K ctx, tools/vision/thinking); quality = `llama3.3:70b` (14/14) but 4.8 tok/s (~33× slower). Local context ceiling ~262K; near-1M is not interactive here.

### 1B. FREE CLOUD (RAN) — measured through our harness

| Model / route | tok/s (e2e) | Context | Capabilities | Notes |
|---|---:|---:|---|---|
| **Groq `openai/gpt-oss-120b`** (free tier) | **268.3 / 301.4** | **131,072** | tools-capable, 120B MoE (5.1B active) | measured via raw provider API; 65,536 max out; free tier ≈30 RPM / 1k RPD / 200k TPD per model |
| `nvidia/nemotron-3.5-lightning:free` (alias `free-HighTPS_Context`) | **51.8 mean** (44.2–59.3) | **1,000,000** | text, tools | 65,536 max out; 14/14 coding; **failed exact-JSON at 778K twice**; 262K retrieval in 13.3 s |
| `nvidia/nemotron-3.5-super-vl:free` (same alias) | **20.4** | **1,000,000** | text (+vision at backend), tools | 131,072 max out; **passed JSON at 778,824 and 983,054 tokens** (88–113 s) |
| Gemini 3.5 Flash (AI Studio free tier) | not valid | 1M class | vision, tools, grounding | short call returned 9 completion tokens — thinking consumed the budget; TPS not measurable this way |

> **Headline:** our existing free-cloud picker entry (`free-HighTPS_Context`) is the **slowest** free option we have access to. Groq's free `gpt-oss-120b` measured **5.2–5.8× faster** than Nemotron Lightning and **13–15× faster** than Super-VL on the identical prompt — at **131K instead of 1M** context.

### 1C. FREE CLOUD (THEORETICAL) — keys exist in `~/dbpasses.txt`, catalog live, NOT measured

| Provider | Free models found | Best context | Disqualification status |
|---|---|---|---|
| **Cerebras** (`CEREBRAS_FREE_API_KEY`) | `gpt-oss-120b`, `qwen-3.8-27b` | — | catalog live via API; free-tier persistence not confirmed |
| **SambaNova** (`SAMBANOVA_API_KEY`) | `gpt-oss-120b` (131K), `MiniMax-M3.1-Flash` (1M), `GLM-5.3` (1M), `gemma-4-31B-it` (131K), `Meta-Llama-3.3-70B` (131K) | 1M | key works, catalog live |
| **Hyperbolic** (`HYPERBOLIC API KEY`) | `Qwen3-Coder-480B-A35B` (262K), `DeepSeek-R1` (163K), `Llama-3.3-70B` (131K) | 262K | key works, catalog live |
| **Requesty** (`REQUESTY API KEY`) | 745 models, many at 1M ctx | 1M | 200 free req/day |
| Mistral (`MISTRAL API KEY`) | 55 models (codestral, leanstral, etc.) | — | key works, catalog live |
| Together (`TOGETHER AI API KEY`) | — | — | **HTTP 401 — key rejected** |
| Fireworks (`FIREWORKS FREE API KEY`) | — | — | **HTTP 412** |
| DeepInfra (`DEEPINFRA API KEY`) | — | — | **HTTP 401 — key rejected** |
| Featherless (`FEATHER API KEY`) | — | — | **HTTP 401** |
| Modelscope (`MODELSCOPE API KEY`) | — | — | **URLError** |
| OrcaRouter (`ORCAROUTER.AI`) | — | — | **URLError** |
| Qwen free (`QWEN_API_KEY_FREE`) | — | — | **NO KEY** |
| Chutes (`CHUTES_API_KEY_FREE`) | — | — | **HTTP 404** |

Notable theoretical candidates worth measuring next: **SambaNova `MiniMax-M3.1-Flash` and `GLM-5.3` (1M context, key confirmed live)**, **Hyperbolic `Qwen3-Coder-480B-A35B` (262K, coding-specialist)**, **Requesty (745 models, 1M ctx, 200 free req/day)**.

### 1D. PAID CLOUD (RAN) — measured through our harness

| Model | tok/s (e2e) | Advertised context | Capabilities | Caveat |
|---|---:|---:|---|---|
| **Inception Mercury-3** (`XX_inception`) | **547.8** (360.8 / 734.7) | **1,000,000** | text (no vision), tools | fastest paid member we run |
| xAI **Grok 4.7** (`XX_grok`) | 98.7 (98.3 / 99.1) | 500,000 | vision, tools, web | half the context of 1M models |
| DeepSeek **V4 Pro** (`XX_deepseek`) | 61.9 (reasoning only) | 1,048,576 | vision-capable at vendor, not via our key | **no visible answer** at 220-token budget |
| Moonshot **Kimi K3.1 Pro** (`XX_kimi`) | 33.6 (31.6 / 35.6) | 1,048,576 | text, tools | requires `temperature: 1`; ~217/220 tokens were reasoning |

> Paid Kimi/DeepSeek figures are **reasoning throughput, not useful-answer throughput**.

### 1E. PAID CLOUD (THEORETICAL) — providers we hold keys for but did NOT measure

| Provider | Evidence | Status |
|---|---|---|
| **Cerebras** | Third-party benchmarks on `gpt-oss-120b`: ~1,586–1,780 tok/s | Our key returns **HTTP 402** (trial credits exhausted) — cannot measure |
| SambaNova | ~700–760 tok/s on `gpt-oss-120b` | key works (free tier above); paid speed unconfirmed by us |
| Fireworks | ~740 tok/s | **HTTP 412 on free key** |
| Together | ~597 tok/s | **HTTP 401 on key** |
| Groq paid | ~473–480 tok/s | cheaper than Cerebras per output token |

**Missing paid tier:** an *inference-speed* tier, not another frontier-reasoning vendor. Cerebras is ~3× faster than our fastest paid model (Mercury-3, 547.8 tok/s) but unverified by us (402). Groq paid (~480 tok/s) is cheaper than Cerebras per output token and already ~5× our needs.

---

## 2. Disqualified providers (temporary credit — do not pursue)

| Provider | Key slot in `~/dbpasses.txt` | Reason disqualified |
|---|---|---|
| OpenAI | `OPENAI_KEY` / `sk-proj-...` | carries no credits (429 / 400); not replenishing free tier |
| Anthropic | `ANTR` / `sk-ant-api03-...` | carries no credits; separate `/v1/messages` endpoint; not free |
| HuggingFace | `HUGGING_FACE_TOKEN` | **402 monthly free credits depleted; subscribe to PRO or wait month reset** — temporary, not replenishing |
| Cloudflare | `cfut_...` | **429 daily 10,000-neuron quota exhausted; resets at UTC midnight** — temporary daily cap |
| Novita | `NOVITA API KEY` | slot holds a copy of the Brave token; balance=0 stops free access |
| Nous | `NOUS API KEY` | **USE ONLY FREE MODELS per provider — balance=0 stops free access** |
| DeepInfra primary | `DEEPINFRA API KEY` | dead (401); alt key works but not measured |
| GitHub Models | `GITHUB MODELS API KEY` | free quota on PAT tokens; not confirmed replenishing |
| Hypereal | `HYPEREAL CLOUD API KEY` | per-model/per-tier credit pools; several models report `balance=0` |

Rule applied: temporary credits that run out and do not replenish are disqualified. We keep them only as "may reconsider if credits replenish."

---

## 3. Are there more performance-efficient models? Yes — one per quadrant

| Quadrant | Upgrade | Why |
|---|---|---|
| **Local (free)** | `gpt-oss-120b` MXFP4 (~59–68 GiB) | ~12× faster than our `llama3.3:70b` (4.8 tok/s) at equal/better coding; 5.1B-active MoE fits GB10 bandwidth; alternative `Qwen3.6-35B-A3B` (~86 tok/s, 22 GiB) |
| **Free cloud (ran)** | Groq `gpt-oss-120b` pinned route | 5–6× faster than Nemotron Lightning; strict-$0, no paid fallback; trade 1M→131K context; **≈30 RPM / 1k RPD / 200k TPD** — single-agent OK, not high-concurrency |
| **Free cloud (theoretical — measure next)** | SambaNova `MiniMax-M3.1-Flash` / `GLM-5.3` (1M ctx); Hyperbolic `Qwen3-Coder-480B-A35B` (262K) | keys confirmed live, 1M-class context, unmeasured TPS |
| **Paid cloud (ran)** | Inception Mercury-3 (547.8 tok/s) — already our fastest | keep as paid speed baseline |
| **Paid cloud (theoretical)** | Cerebras `gpt-oss-120b` (~1,586 tok/s) if credits replenish; else Groq paid (~480 tok/s) | Cerebras ~3× Mercury-3 but our key is 402; Groq paid cheaper per output token |

**RPM caveat:** free-tier Groq ≈30 RPM / 1k RPD / 200k TPD per model — fine for single-agent use, not for high-concurrency batching.

---

## 4. Cross-check by other AIs

Sent the full four-quadrant inventory + gap questions to peers via `tools/consult_multi.py` (keys from `~/dbpasses.txt`; no key values stored).

| Voice | Result | Useful signal |
|---|---|---|
| **NVIDIA · llama-3.2-90b-vision-instruct** | answered | Confirmed category assignments sound; agreed MoE/Qwen3.6-35B-A3B is the local upgrade path; flagged Groq free-tier performance/RPM claims need live verification (fair); agreed Cerebras/SambaNova/Fireworks/Together are candidate speed providers but unconfirmed without live data |
| **Groq · qwen/qwen3-32b** | HTTP 404 | model not available — no cross-check |
| **Cerebras · llama3.1-8b** | HTTP 404 | no cross-check |
| **Together · Llama-3-8B-Instruct-Lite** | HTTP 401 | key rejected — no cross-check |
| **Fireworks · kimi-k2p5** | HTTP 404 | no cross-check |
| **Gemini · gemini-2.5-flash** | HTTP 429 | quota exhausted — no cross-check |

Only **one peer (NVIDIA) returned a usable answer**; the rest hit 404/401/429. Treat single-model AI audits as hypotheses, not evidence. The NVIDIA response did not contradict any finding here.

---

## 5. Read of the other `.MD` already dropped

`BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH.MD` (Freebuff, same date/host) covers the same categories with more local rows and a fuller cross-check table. My findings are **consistent** with it:

- It already identifies Groq `gpt-oss-120b` free as the speed leader (268–301 tok/s) beating Nemotron — I confirm with my own measured rows.
- It already recommends `gpt-oss-120b` MXFP4 as the local GX10 upgrade — I concur.
- It already flags Cerebras as the paid speed tier (third-party, HTTP 402 here too) — I concur and add Groq paid as the cheaper alternative.

No contradictions; my file adds the **four-quadrant breakout (ran vs theoretical)**, the **temporary-credit disqualification section**, and the **single-upgrade-per-quadrant summary**.

---

## 6. Method and limitations

- **Same harness for speed:** one prompt ("Print the integers 1 through 200 separated by commas, without explanation."), 220-token cap, two sequential calls per model. Reported TPS = `provider completion_tokens / wall seconds` — includes queue, prefill, network; **not** decoder-only streaming TPS.
- **Local vs cloud are different measurements.** Ollama rows are native `eval_count/eval_duration` (decoder speed); cloud rows are end-to-end. Do not compare local decode with cloud e2e as one axis.
- **Two calls per model is a smoke test, not capacity.** Shared host, free-tier quotas, provider caching, and route selection move these numbers.
- **Context numbers are advertised metadata**, except retrieval tests (778K/983K free-cloud) which are actual token counts.
- **Coding figures** come from a 14-check synthetic mini-suite; they rank these models against each other, not against a coding leaderboard.
- Third-party GB10 and provider speeds are cited from published measurements current to 2026-11-11; labelled as such, not our measurements.
- No credential values are included in this file.

## 7. Reproduction

```bash
# measured comparison evidence
cat /home/eaguiar2015/findtorontoevents_antigravity.ca/reports/free_high_tps_context_2026-11-11.json

# LiteLLM config (route definitions)
/home/eaguiar2015/findtorontoevents_antigravity.ca/litellm_config.yaml

# peer consult (may be rate-limited)
python3 /home/eaguiar2015/findtorontoevents_antigravity.ca/tools/consult_multi.py --provider nvidia --model meta/llama-3.2-90b-vision-instruct --prompt-file /tmp/bench_consult_prompt.md
```
