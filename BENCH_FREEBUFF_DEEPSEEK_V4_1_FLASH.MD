# BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH

Author-agent: Freebuff (DeepSeek V4.1 Flash) · Date: **2026-11-11** · Host: Ubuntu ARM64 **NVIDIA GB10** (DGX Spark / ASUS Ascent GX10 class), 121 GiB unified memory, 20 vCPU, 273 GB/s class memory bandwidth.

Scope: inventory our fleet by **where the weights run** — locally, free cloud, paid cloud — then answer one question: *are there more performance-efficient models we have not adopted, and in which category?* Metrics: **capabilities, tokens/s, context window**.

All numbers below are either measured on this host through the fleet's own stack (LiteLLM :4000, vLLM :8000, Ollama :11434) or explicitly cited as third-party. Method is at the bottom.

---

## 1. Our current models, split by category

### 1A. LOCALLY RUN (FREE) — weights on this machine

Warm short-prompt decode rate = Ollama `eval_count / eval_duration`, 220-token cap. Context = declared model context. Coding = 14-check synthetic mini-suite (not a leaderboard).

| Model (Ollama unless noted) | tok/s | Context | Capabilities | Coding | Weights |
|---|---:|---:|---|---:|---:|
| `llama3.2:1b` | **163.6** | 131,072 | tools | 5/14 | 1.2 GiB |
| `qwen2.5:3b` | **94.9** | 32,768 | tools, live-fetch PASS | 7/14 | 1.8 GiB |
| `llama3.2:3b` | 93.8 | 131,072 | tools | 13/14 (fails JSON-key contract) | 1.9 GiB |
| `qwen3:30b-a3b` | 87.0 | 262,144 | tools, thinking | 0/14 (JSON fails ≥32K) | 17.3 GiB |
| **`qwen3.5:35b-a3b`** | **76.4** | **262,144** | **tools, vision, thinking** | **12/14**, live-fetch PASS | 22.2 GiB |
| `qwen2.5-coder:7b` | 46.2 | 32,768 | tools, insert | 9/14 | 4.4 GiB |
| `qwen2.5:7b` | 46.1 | 32,768 | tools | 8/14 | 4.4 GiB |
| `gemma3:12b` | 27.9 | 131,072 | vision (no tools) | 10/14 | 7.6 GiB |
| `qwen2.5-coder:14b` | 24.0 | 32,768 | tools, insert | 10/14 | 8.4 GiB |
| `qwen2.5:14b` | 23.9 | 32,768 | tools | 11/14 | 8.4 GiB |
| `deepseek-r1:32b` | 10.7 | 131,072 | tools, thinking | 0/14 | 18.5 GiB |
| `qwen2.5:32b` | 10.5 | 32,768 | tools | 9/14 | 18.5 GiB |
| `llama3.3:70b` | 4.8 | 131,072 | tools | **14/14** (best quality) | 39.6 GiB |
| vLLM `Qwen2.5-14B-Instruct` | 8.0 (e2e) | **8,192 (hard cap)** | tools **broken (HTTP 400)** | 5/14 | BF16 |

Local reality check:

- **Speed winner:** `llama3.2:1b` at 163.6 tok/s, but 5/14 coding — throughput without capability.
- **Balanced winner:** `qwen3.5:35b-a3b` — 76 tok/s, 262K context, tools, vision, 12/14.
- **Quality winner:** `llama3.3:70b` — 14/14 but **4.8 tok/s**, ~33× slower than the 35B MoE.
- **Context ceiling is the real local limit:** only three local models declare ≥131K; one declares 262K. A **261,652-token** local retrieval took **164.30 s** (≈1,593 tok/s prefill). Near-1M locally is not a usable interactive path on this box.
- vLLM advertises **8,192 total** tokens and has no tool parser enabled — it is the weakest member of the local fleet despite being "the GPU engine".

### 1B. FREE (CLOUD) — zero token price, weights off-machine

| Model / route | tok/s (e2e) | Context | Capabilities | Notes |
|---|---:|---:|---|---|
| **Groq `openai/gpt-oss-120b`** (free tier) | **268.3 / 301.4** | **131,072** | tools-capable, 120B MoE (5.1B active) | measured today through the raw provider API; 65,536 max output; free tier ≈30 RPM / 1k RPD / 200k TPD per model |
| `nvidia/nemotron-3.5-lightning:free` (OpenRouter, alias `free-HighTPS_Context`) | 51.8 mean (44.2–59.3) | **1,000,000** | text, tools | 65,536 max out; 14/14 coding; **failed exact-JSON at 778K twice**; 262K retrieval in 13.3 s |
| `nvidia/nemotron-3.5-super-vl:free` (same alias) | 20.4 | **1,000,000** | text (+vision at backend), tools | 131,072 max out; 14/14 coding; **passed JSON at 778,824 and 983,054 tokens** (88–113 s) |
| Gemini 3.5 Flash (AI Studio free tier) | not valid (see limits) | 1M class | vision, tools, grounding | short call returned 9 completion tokens — thinking consumed the 220-token budget; TPS not measurable this way |
| Cerebras `gpt-oss-120b` | — | — | — | **HTTP 402 Payment Required** — the old free tier is gone; no longer category B |
| NVIDIA NIM `gpt-oss-120b` / `deepseek-v4-pro` | — | — | — | **410 Gone** (retired / EOL 2026-08-07); NIM free endpoints churn hard |
| `free-mode`, `free-mode-fast`, `ROCKETSPEED-FREE` (LiteLLM groups) | mixed | mixed | mixed | **not strict $0**: `free-mode*` can fall back to `paid-mode*`; ROCKETSPEED-FREE mixes Groq + Cohere + Gemini behind `simple-shuffle` |

> **The headline finding:** our *existing* free-cloud picker entry is the **slowest** free option we have access to. Groq's free `gpt-oss-120b` measured **5.2–5.8× faster** than Nemotron Lightning and **13–15× faster** than Super-VL on the identical prompt — but with **131K instead of 1M** context.

### 1C. PAID (CLOUD) — metered

| Model | tok/s (e2e) | Advertised context | Capabilities | Caveat |
|---|---:|---:|---|---|
| **Inception Mercury-3** (`XX_inception`) | **547.8** (360.8 / 734.7) | **1,000,000** | text (no vision), tools | fastest paid member we run; 216/214 output tokens |
| xAI **Grok 4.7** (`XX_grok`) | 98.7 (98.3 / 99.1) | 500,000 | vision, tools, web | half the context of the 1M models |
| DeepSeek **V4 Pro** (`XX_deepseek`) | 61.9 (reasoning only) | 1,048,576 | vision-capable at vendor, no vision via our key, 393,216 max out | **no visible answer** at a 220-token budget |
| Moonshot **Kimi K3.1 Pro** (`XX_kimi`) | 33.6 (31.6 / 35.6) | 1,048,576 | text, tools | requires `temperature: 1`; 217/220 tokens were reasoning |
| Also configured, unmeasured: Claude Haiku 4.5, GPT-5.5-instant, GPT-5-chat, mimo-v2.5-pro, Ring 2.6 1T | — | — | — | no same-harness numbers yet |

> Paid Kimi/DeepSeek figures are **reasoning throughput, not useful-answer throughput**. Do not read them as "worse than Nemotron".

---

## 2. Are there more performance-efficient models? Yes — one per category

### 2A. Paid cloud we had not considered → **Cerebras Inference**

We benchmarked four paid endpoints and none of them is a throughput specialist. Independent provider benchmarks on the same open model we can already run (`gpt-oss-120b`) put:

| Provider | gpt-oss-120b output tok/s | List price in/out per 1M |
|---|---:|---|
| **Cerebras** (WSE-3) | **~1,586–1,780** | $0.35 / $0.75 |
| SambaNova | ~700–760 | $0.14 / $0.95 |
| Fireworks | ~740 | — |
| Together | ~597 | — |
| **Groq** (LPU) | ~473–480 | **$0.15 / $0.60** |
| DeepInfra | ~189 | ~$0.07 blended |

Mercury-3's 547.8 tok/s makes it the best paid member we run, but **Cerebras is ~3× faster than our fastest paid model at 1M-class capability on an open 120B MoE**, and **Groq is ~1.9× cheaper than Cerebras per output token** for ~3.6× less speed. That is the missing paid tier: an *inference-speed* tier, not another frontier-reasoning vendor.

Verification note: our Cerebras key currently returns **HTTP 402** (trial credits exhausted), so the Cerebras figure is third-party, not measured by us. Add it only with a paid/trial balance.

### 2B. Free cloud model that is better → **Groq `gpt-oss-120b` for speed**

- **268–301 e2e tok/s**, 131K context, tools, 120B MoE — measured by us, free tier.
- Beats `free-HighTPS_Context` by **5.2–5.8×** on speed and is already reachable with a key we hold.
- Cost of the swap: **131K vs 1M context**. Super-VL retains the only *verified* ~983K path.

**Recommended routing (not yet installed):** keep `free-HighTPS_Context` for ≥300K inputs and add a separate strict-$0 `free-fast-120b` route pinned to Groq `gpt-oss-120b` with **no paid fallback** and its own latency budget. Today `free-mode-fast`/`ROCKETSPEED-FREE` already contain Groq but shuffle with Cohere/Gemini and inherit paid-capable fallbacks, so "fast" is not guaranteed and "$0" is not enforced.

### 2C. Locally run (free), GX10-compatible, not considered → **`gpt-oss-120b` MXFP4**

We run no model above 70B, and the 70B we do run is dense and slow. On GB10-class hardware the published picture:

| Local candidate | Footprint | Published GB10 speed | Why it matters |
|---|---:|---:|---|
| **gpt-oss-120b (MXFP4, 5.1B active)** | ~59–68 GiB | **52.9–58.7 tok/s** (llama.cpp), 60.7 (vLLM), 42.1 (Ollama) | **12× faster than `llama3.3:70b` at higher effective quality**; independent 6-language compile benchmark ranks it the single model to keep loaded (equal best pass rate, 3× faster to a passing answer, best Rust 24/30) |
| **Qwen3.6-35B-A3B (NVFP4)** | ~22 GiB | **~86.3 tok/s** (vLLM, 1 Spark) | direct upgrade path for our 76 tok/s `qwen3.5:35b-a3b` |
| Gemma 4 26B (4B active, 4-bit) | ~16 GiB | **59.1 tok/s** median | most-run model on DGX Spark today |
| Qwen3.5-122B-A10B (Q6_K) | ~104 GiB | ~14 tok/s | largest that fits comfortably; slow |
| llama3.3:70b (what we run) | 39.6 GiB | 4.8 tok/s (ours) | dense; dominated on speed *and* footprint-adjusted quality |

**Answer for Category A:** load **`gpt-oss-120b` MXFP4**. It fits (~60 GiB of 121 GiB), it is a 5.1B-active MoE so GB10's 273 GB/s bandwidth is not the bottleneck, and it replaces a 4.8 tok/s dense 70B with ~42–59 tok/s at equal-or-better coding quality. `Qwen3.6-35B-A3B` is the cheaper incremental win if we want to stay in the 22 GiB class (needs NVFP4 via vLLM, not Ollama).

MoE is the rule on this hardware: every fast local row is MoE (35B-A3B, 30B-A3B, 3B) and every dense row ≥14B is ≤24 tok/s.

---

## 3. Cross-check by other AIs

Prompt: full three-category inventory + the gap questions above, ~450-word limit, asked to flag errors. Sent to five providers via the local `consult_multi` tool (keys from the private credential file; no key values stored here).

| Voice | Result | Useful signal |
|---|---|---|
| **Groq · gpt-oss-120b** | answered | agreed MoE/Qwen3.6-35B-A3B is the local upgrade path (~86 tok/s on identical GB10 class); claimed a "Llama 3.2 8B at ~125 tok/s / 13-14" and a "Mistral-7B free at 1M ctx, 68 tok/s" — **both unsupported; rejected** |
| **DeepInfra · Llama-3.1-8B** | answered | proposed **Gemma 4 26B (~86 tok/s)** as the local candidate and flagged that Category A/B/C compare different hardware — a fair methodological criticism; its "missing paid provider = Mercury-3" is wrong (already deployed) |
| **Google · Gemini 3.5 Flash** | partial | refused the premise as "fictional/anachronistic hardware (GB10/DGX Spark)"; **rejected** — the host is real (`nvidia-smi` reports NVIDIA GB10) |
| **Cerebras · gpt-oss-120b** | HTTP 402 | key has no balance — cannot cross-check |
| **NVIDIA NIM · kimi-k2.6 / deepseek-v4-pro** | HTTP 404 / 410 | models retired — NIM catalog churn confirmed empirically |
| **Mistral · mistral-small-latest** | HTTP 429 | rate-limited |
| **Nous · Hermes-4-405B** | HTTP 404 | model retired |
| **GitHub Models · gpt-4o-mini** | DNS failure | endpoint unresolvable from this host |
| **Together · Llama-3.3-70B-Turbo** | HTTP 401 | key rejected — no cross-check possible |

**Consensus that survives review:** MoE on GB10 is the efficiency answer; `qwen3.5:35b-a3b` is a sound local default but not the ceiling; an MoE in the 26–120B class is the next local step. **Disagreement left open:** whether the next dollar should go to Cerebras (speed) or be avoided entirely in favour of Groq's cheaper 480 tok/s tier. My call: Groq paid first (cheaper, same model, 480 tok/s is already 5× our needs), Cerebras only if TTFT-critical.

Two of the three responding models produced confident, verifiable errors. Treat single-model AI audits as hypotheses, not evidence.

---

## 4. Method and limitations

- **Same harness for speed:** one prompt ("Print the integers 1 through 200 separated by commas, without explanation."), temperature 0 (Gemini excluded), 220-token cap, two sequential calls per model. Reported TPS is `provider completion_tokens / wall seconds` — it **includes** queue, prefill and network, and is **not** decoder-only streaming TPS.
- **Local vs cloud are not the same measurement.** Ollama rows are native `eval_count/eval_duration` (decoder speed); cloud rows are end-to-end. Do not compare 76.4 (local decode) with 268 (cloud e2e) as if one axis — the local e2e figure for `qwen3.5:35b-a3b` on the same prompt was 72.8 tok/s.
- **Two calls per model is a smoke test, not capacity.** Shared host, free-tier quotas, provider caching and route selection all move these numbers.
- **Context numbers are advertised metadata**, except the retrieval tests (262K local, 778K/983K free-cloud) which are actual token counts.
- **Coding figures** come from a 14-check synthetic mini-suite; they rank these models against each other, not against a coding leaderboard.
- Third-party GB10 and provider speeds are cited from published llama.cpp/DGX-Spark measurements and provider benchmarks current to 2026-11-11; they are **not** our measurements and are labelled as such.

## 5. Reproduction

```bash
# free-cloud + local same-harness comparison (this file's measured rows)
python3 /tmp/bench_free.py            # writes reports/model_bench_three_categories_2026-11-11.json

# free route policy tests
.venv/bin/python -m unittest discover -s tests -p test_free_high_tps_context.py -v

# AI cross-check
python3 tools/consult_multi.py --provider groq --model openai/gpt-oss-120b --prompt-file /tmp/bench_consult.md
```

Evidence produced on this host: `reports/local_model_benchmark_2026-11-11.md`, `reports/free_high_tps_context_2026-11-11.json`, `reports/model_bench_three_categories_2026-11-11.json`. No credential values are included in this file.
