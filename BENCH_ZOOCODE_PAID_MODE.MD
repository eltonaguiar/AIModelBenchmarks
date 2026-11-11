# BENCH_ZOOCODE_PAID_MODE.MD
## 1. Our Current Models — LOCALLY RUN (FREE)
| Model | Access path | TPS (e2e) | Context | Notes |
|---|---|---|---|---|
| vLLM Qwen2.5-14B-Instruct | `hosted_vllm/Qwen/Qwen2.5-14B-Instruct` | 8.0 | 8,192 (hard cap) | Tools broken; 400 HTTP error on JSON output |
| qwen2.5:14b | `ollama_chat/qwen2.5:14b` | 23.9 | 32,768 | 11-of-14 context window; 22.2 GiB VRAM |
| qwen2.5:7b | `ollama_chat/qwen2.5:7b` | 46.1 | 32,768 | |
| qwen2.5-coder:7b | `ollama_chat/qwen2.5-coder:7b` | 46.2 | 32,768 | |
| qwen2.5:3b | `ollama_chat/qwen2.5:3b` | 94.9 | 32,768 | |
| llama3.2:1b | `ollama_chat/llama3.2:1b` | **163.6** | 131,072 | 5-of-14 context window |
| llama3.2:3b | `ollama_chat/llama3.2:3b` | 93.8 | 131,072 | 13-of-14 context window |
| qwen3:30b-a3b | `ollama_chat/qwen3:30b-a3b` | 87.0 | 262,144 | 0-of-14 context window |
| qwen3.5:35b-a3b | `ollama_chat/qwen3.5:35b-a3b` | 76.4 | 262,144 | 12-of-14 context window; 22.2 GiB VRAM; tools+vision+thinking |
| gemma3:12b | `ollama_chat/gemma3:12b` | 27.9 | 8,192 | vision-no-tools |
| deepseek-r1:32b | `ollama_chat/deepseek-r1:32b` | 10.7 | 131,072 | |
| qwen2.5:32b | `ollama_chat/qwen2.5:32b` | 10.5 | 131,072 | |
| llama3.3:70b | `ollama_chat/llama3.3:70b` | 4.8 | 131,072 | 14-of-14 context window |

### Locally-runnable models we have NOT pulled yet (GX10 candidates)
- `openai/gpt-oss-120b` (MXFP4)
- `Qwen/Qwen3.8-27b` (NVFP4)
- `Qwen3-Coder` (NVFP4)
- `nemotron-3-nano-omni-30b-a3b-reasoning`
- `gemma-4-26b-a4b-it`
- `gemma-4-31b-it`

## 2. Our Current Models — FREE (CLOUD)
Only the two RAN members below sit in one LiteLLM alias, `free-HighTPS_Context`, with `max_price: {prompt: 0, completion: 0}`.

### 2A. FREE CLOUD (RAN) — measured through our harness
| Model | Access path | TPS (e2e) | Context | Notes |
|---|---|---|---|---|
| nemotron-3.5-lightning:free | `openrouter/nvidia/nemotron-3.5-lightning:free` | **51.8** | 1,000,000 | |
| nemotron-3.5-super-vl:free | `openrouter/nvidia/nemotron-3.5-super-vl:free` | 20.4 | 1,000,000 | Super-VL passed JSON, Lightning failed twice |

### 2B. FREE CLOUD (THEORETICAL) — zero-price metadata, not yet run through our harness
| Model | Access path | Context | Notes |
|---|---|---|---|
| nemotron-3-ultra-550b-a55b | `openrouter/nvidia/nemotron-3-ultra-550b-a55b:free` | 1,048,576 | |
| baidu/ernie-5.5-flash-preview | `openrouter/baidu/ernie-5.5-flash-preview:free` | 1,048,576 (max out 921,600) | |
| thinkingmachines/inkling-2-small | `openrouter/thinkingmachines/inkling-2-small:free` | 1,048,576 | |
| google/gemma-4-26b-a4b-it | `openrouter/google/gemma-4-26b-a4b-it:free` | 8,192 | |
| google/gemma-4-31b-it | `openrouter/google/gemma-4-31b-it:free` | 8,192 | |
| cohere/north-mini-code | `openrouter/cohere/north-mini-code:free` | 256,000 | |
| dots-studio/dots-3-note-preview | `openrouter/dots-studio/dots-3-note-preview:free` | 460,800 | |
| recursive/flint-v1.0 | `openrouter/recursive/flint-v1.0:free` | 235,929 | |
| openrouter/free | `openrouter/free` | 1,048,576 | |

## 3. Our Current Models — PAID (CLOUD)

### 3A. PAID CLOUD (RAN) — measured through our harness
| Model | Access path | TPS (e2e) | Context | Notes |
|---|---|---|---|---|
| Mercury-3 | `openai/mercury-3` | **547.8** | 1,000,000 | |
| Grok 4.7 | `xai/grok-4.7` | 98.7 | 500,000 | |
| DeepSeek V4 Pro | `deepseek/deepseek-v4-pro` | 61.9 | 1,048,576 | |
| Kimi K3.1 Pro | `openai/kimi-k3.1-pro` | 33.6 | 1,048,576 | |

### 3B. PAID CLOUD (THEORETICAL) — configured but not yet run through our harness
| Model | Access path | Context | Notes |
|---|---|---|---|
| Claude Haiku 4.5 | `anthropic/claude-haiku-4-5-20251001` | 200,000 | |
| GPT-5.5-instant | `openai/gpt-5.5-instant` | 16,384 | |
| GPT-5-chat | `openai/gpt-5-chat` | 16,384 | |
| MiniMax-M2 | `openai/MiniMax-M2` | 1,048,576 | |
| Ring 2.6 1T | `openrouter/inclusionai/ring-2.6-1t` | 1,048,576 | |

## 4. Rate Limits and Concurrency

### 4.1 What our LiteLLM router enforces (`litellm_config.yaml`)
- **Routing Strategy:** `simple-shuffle`
- **Pre-call Checks:** Enabled (`enable_pre_call_checks: true`)
- **Retries:** `num_retries: 2`, `allowed_fails: 1` (for two-member failure mode)
- **Cooldown:** `cooldown_time: 65` seconds
- **Retry After:** `retry_after: 1` second
- **Context Window Fallbacks:**
    - `free-mode` → `free-mode-large`
    - `hybrid-model` → `free-mode-large`
    - `paid-mode` → `paid-mode-large`, `free-mode-large`
- **Fallback Chains:** `hybrid-model`, `free-mode`, `free-mode-large`, `paid-mode`, `paid-mode-large`, `free-mode-fast`, `paid-mode-fast`, `free-mode-tools`, `true-free-tools-fast`, `ROCKETSPEED-FREE`, `ROCKETSPEED-FAST`
- **Retry Policy:** `BadRequest: 1`, `Auth: 0`, `Timeout: 2`, `RateLimit: 3`, `ContentPolicy: 0`, `InternalServer: 5xx` (2 retries)

### 4.2 What upstream enforces (live-queried today)
- **OpenRouter:** `1,000 req/day` (general limit), plus specific model limits (e.g., `gemini/gemini-2.5-flash` at 5 RPM). Some models have `free` tier with `max_price: {prompt: 0, completion: 0}`.
- **Our Router's `free-mode`:** Has 44 deployments, including Groq, NIM, Cerebras, Together, Fireworks, Mistral, Cohere, Gemini (RPM 5-15), and OpenRouter's `:free` tier models.
- **`true-free-tools` (3 deployments) & `true-free-tools-fast` (3 deployments):** These are local Ollama/vLLM routes, effectively rate-limit-free.
- **`ROCKETSPEED-FREE` (5 deployments):** Includes Groq `gpt-oss-120b` (x2), `gpt-oss-20b`, Cohere `command-r7b`, Gemini `gemini-2.5-flash-lite`.
- **`paid-mode` (25 deployments):** Includes Moonshot (`rpm: 5000, tpm: 3000000`), GPT-5.5-instant, Claude Haiku 4.5, and others.

## 5. Free Cloud Provider Due Diligence

### 5.1 Qualified — genuinely free and replenishing
| Provider | Key Label | Daily Limit | Notes |
|---|---|---|---|
| REQUESTY | `REQUESTY API KEY (200 free requests a day)` | 200 requests/day | Replenishing |
| INFRON | `INFRON API KEY(FREE 1000 requests a day )` | 1000 requests/day | Replenishing |
| NARAROUTER | `NARAROUTER AI API KEY https://router.bynara.id/keys (15 req/min free tier)` | 15 req/min | Replenishing |
| AGNES AI | `AGNES AI API KEY (10 req/min tier)` | 10 req/min | Replenishing |
| ofox | `ofox` | 19 free models | Video/image models (e.g., alibaba/wan-*, happyhorse-*); context=0 |

### 5.2 Disqualified — trial credit or non-replenishing
| Provider | Key Label | Notes |
|---|---|---|
| SUBQ | `SUBQ Token ($5 free credit+$20 more added)` | One-shot credit, not replenishing |
| SAMBANOVA CLOUD | `SAMBANOVA CLOUD API KEY(expires in 1 year after 2026-11-02)` | Expires in 1 year |
| COHERE_TRIAL | `COHERE_TRIAL_API_KEY2(November 2026)` | Trial credit, expires Nov 2026 |

### 5.3 Verdict on "is there a better free cloud model?"
The primary new finding is `ofox`, offering 19 free video/image models. For text generation, the existing replenishing tiers (REQUESTY, INFRON, NARAROUTER, AGNES AI) remain the most promising for sustained free usage. No single model dramatically outperforms Nemotron Lightning on TPS within the free tier, but the breadth of options in `free-mode` and `ROCKETSPEED-FREE` offers flexibility.

## 6. Cross-Check Results

### 6.1 Which peers actually answered
Only two peer files were found in the repo:
- `BENCH_OPENCODE_NEMOTRON_HIGH_TPS_CONTEXT.md`
- `BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH.MD` (modified by peers)

### 6.2 What the peers added that we did not already have
- **`BENCH_OPENCODE_NEMOTRON_HIGH_TPS_CONTEXT.md`**: This file highlights `Groq openai/gpt-oss-120b` at 268.3/301.4 tok/s as a headline free option, claiming it's 5.2–5.8× faster than Nemotron Lightning. Its cross-check table shows only NVIDIA `llama-3.2-90b-vision-instruct` answered; Groq, Cerebras, Together, Fireworks, Gemini all returned errors (404/401/429).
- **`BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH.MD`**: Grew significantly via peer commits, indicating ongoing work on that model's benchmark.

## 7. Peer Review Notes (files already in this repo)
- **`BENCH_OPENCODE_NEMOTRON_HIGH_TPS_CONTEXT.md`**: Claims Groq `gpt-oss-120b` is the fastest free option. Its cross-check table shows weak corroboration, with only NVIDIA `llama-3.2-90b-vision-instruct` responding.
- **`BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH.MD`**: Modified by peers, suggesting active development and updates.

## 8. Answers to the three questions asked

1.  **Is there a paid cloud provider we hadn't considered?**
    No, our current list of paid providers (Mercury-3, Grok, DeepSeek V4 Pro, Kimi K3.1 Pro, Claude Haiku 4.5, GPT-5.5-instant, GPT-5-chat, MiniMax-M2, Ring 2.6 1T) appears comprehensive based on common offerings. The primary gap is in *measuring* the theoretical paid models.

2.  **Is there a locally-run GX10-compatible model we didn't consider?**
    Yes, several potential candidates were identified in section 1 (e.g., `openai/gpt-oss-120b`, `Qwen/Qwen3.8-27b`, `nemotron-3-nano-omni-30b-a3b-reasoning`). These require further investigation and hardware compatibility checks.

3.  **Is there a better free cloud model?**
    `ofox` offers 19 free video/image models. For text, the existing replenishing tiers (REQUESTY, INFRON, NARAROUTER, AGNES AI) are the most robust. `nemotron-3.5-lightning:free` remains a strong TPS performer. The breadth of options in `free-mode` and `ROCKETSPEED-FREE` provides flexibility, but requires careful probing of specific model paths.

## 9. Method and limitations

- **TPS** = reported completion tokens ÷ total wall time for a short (~220-token) call. This is **end-to-end** throughput, not sustained decode rate. Two calls per model. **Not a stability guarantee.**
- **Local tok/s** = Ollama `eval_count / eval_duration` (decode-only, warmer and higher than e2e).
- **Context** = `model_info.max_input_tokens` in our config for models we serve; `context_length` from `/v1/models` for catalogue entries marked *advertised*.
- **Rate limits** = direct API queries (`GET /api/v1/key`, `GET /models/{slug}/endpoints`) at 2026-11-11T19:52Z, plus our own `litellm_config.yaml`.
- **Free-tier qualification** = live chat probe, not `/models` listing. A model appearing in `/models` at price 0 does **not** prove the account can call it — Groq lists 11 models and returns 403 on every one.
- **Known error retained:** the Kimi/DeepSeek TPS figures understate failure, because both models spent the entire output budget on reasoning and returned no answer. Recorded rather than discarded.
- **Credential hygiene:** no key values from `dbpasses.txt` appear in this file. Provider names and verdicts only.

---

## 10. Final Suggestions based on Router Config

The live LiteLLM router configuration reveals a significant coverage gap between the **31 distinct routes** and the **~6 models** we have benchmarked. The `free-mode` alias alone encompasses **44 deployments**, including routes from Groq, NIM, Cerebras, Together, Fireworks, Mistral, Cohere, Gemini (with varying RPMs from 5 to 15), and OpenRouter's `:free` tier models.

**Key Observations and Recommendations:**

*   **Truly Free & Rate-Limit-Free:** The `true-free-tools` (3 deployments) and `true-free-tools-fast` (3 deployments) aliases, powered by local Ollama and vLLM, represent the only genuinely rate-limit-free options. These should be prioritized for any tasks requiring maximum concurrency without external throttling.
*   **Revisiting Probed Providers:** Our earlier "disqualified" verdicts for Groq, Cerebras, Together, Fireworks, NIM, and Ollama Cloud need refinement. While our probes returned `403/402/410` errors, these providers *are* configured within the router (e.g., `XX_groq` points to `groq/qwen/qwen3.8-27b`, `XX_ollamacloud` to `openai/gpt-oss:120b-cloud`). Future benchmarking should focus on the specific free paths within these aliases, rather than assuming unavailability.
*   **Configuration Gaps:** The `free-HighTPS_Context` alias lacks explicit `rpm`/`tpm` configurations, unlike the `paid-mode`'s Moonshot which declares `rpm: 5000, tpm: 3000000`. This suggests an opportunity for more granular control and potentially higher throughput on free tiers if configured.
*   **High-Value Free Additions:**
    *   `baidu/ernie-5.5-flash-preview:free` (with a context window of 1,048,576 tokens, maxing out at 921,600) should be considered as a third member for the `free-HighTPS_Context` alias, offering substantial context.
    *   `openai/gpt-oss:120b-cloud` via `XX_ollamacloud` is another high-TPS candidate that warrants benchmarking.
*   **Credential Notes:**
    *   The `NOVITA API KEY` (L317 in `dbpasses.txt`) is byte-identical to the `BRAVE SEARCH API KEY` (L10), indicating a potential mislabeling or duplicate entry.
    *   `ofox` provides 19 free video/image models, which are valuable but have a context window of 0, limiting their utility for text-based tasks.

This analysis highlights the dynamic nature of cloud AI services and the importance of continuous probing and re-evaluation.
