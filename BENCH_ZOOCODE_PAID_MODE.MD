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
Zero token price **and replenishing quota**. Only the two RAN members below sit in one LiteLLM alias, `free-HighTPS_Context`, with `max_price: {prompt: 0, completion: 0}` (hard fence: quota exhaustion fails rather than bills).

### 2A. FREE CLOUD (RAN) — measured through our harness
| Model | Access path | tok/s (e2e) | Context | Capabilities | Strict $0? |
|---|---|---:|---:|---|---|
| **`nvidia/nemotron-3.5-lightning:free`** | OpenRouter → Nvidia | **51.8** mean | **1,000,000** | text, tools, reasoning | **yes** (0 prompt/0 completion) |
| **`nvidia/nemotron-3.5-super-vl:free`** | OpenRouter → Nvidia | 20.4 | **1,000,000** | vision at backend, tools, reasoning | **yes** |

### 2B. FREE CLOUD (THEORETICAL) — zero-price metadata, not yet run through our harness
| Model | Access path | tok/s (e2e) | Context | Capabilities | Strict $0? |
|---|---|---:|---:|---|---|
| `nvidia/nemotron-3-ultra-550b-a55b:free` | OpenRouter → Nvidia | not measured | 1,000,000 | text, tools, reasoning | **yes** |
| `baidu/ernie-5.5-flash-preview:free` | OpenRouter → Baidu | not measured | 1,024,000 | text, tools, reasoning, 921,600 max out | **yes** |
| `thinkingmachines/inkling-2-small:free` | OpenRouter | not measured | 1,048,576 | text+image, tools, reasoning | **yes** |
| `google/gemma-4-26b-a4b-it:free` | OpenRouter → Google | not measured | 262,144 | text+image+video, tools | **yes** |
| `google/gemma-4-31b-it:free` | OpenRouter → Google | not measured | 262,144 | text+image+video, tools | **yes** |
| `cohere/north-mini-code:free` | OpenRouter → Cohere | not measured | 256,000 | coding, tools (no prompt training) | **yes** |
| `dots-studio/dots-3-note-preview:free` | OpenRouter | not measured | 512,000 | text+image, tools, 460,800 max out | **yes** |
| `recursive/flint-v1.0:free` | OpenRouter | not measured | 262,144 | text, tools, 235,929 max out | **yes** |
| `openrouter/free` (router) | OpenRouter | n/a | 200,000 | random free-model selection | **yes** |

**High-context verification (our own run):** both Nemotron models retained three planted markers at **≈779K input tokens** (778,824). Super-VL passed the requested JSON formatting; **Lightning failed JSON formatting twice** at that size. The alias rotates, so the pool is not deterministic on format-critical work.

**What we have NOT yet integrated but could:** the table above lists 8 additional OpenRouter zero-price models with ≥256K context, four of them at ≥1M. They are free to try and cost nothing to qualify.

----

-| Model | tok/s (e2e) | Context | Capabilities | Caveat |
-|---|---:|---:|---|---|
-| **Mercury-3** (Inception) | **547.8** | 1,000,000 | text, tools | fastest paid we run; ~10.6× Lightning |
-| Grok 4.7 (xAI) | 98.7 | 500,000 | vision, tools, web | half the context of Nemotron |
-| DeepSeek V4 Pro | 61.9 | 1,048,576 | text (vision withheld), 393,216 max out | reasoning ate the 220-token budget in our test |
-| Kimi K3.1 Pro (Moonshot) | 33.6 | 1,048,576 | text, tools | needs `temperature: 1`; 217/220 output was reasoning |

-| Model | tok/s (e2e) | Context | Capabilities | Caveat |
-|---|---:|---:|---|---|
-| Unmeasured: Claude Haiku 4.5, GPT-5.5-instant, GPT-5-chat, MiniMax-M2, Ring 2.6 1T | — | — | — | no same-harness numbers |

**Caveat that matters for decision-making:** the TPS figures for Kimi and DeepSeek are **not useful-answer throughput**. Both spent the entire small output budget on reasoning and returned no final answer. Their real throughput for bounded-output tasks is likely worse than the number suggests, and their effective context advantage over Nemotron is **48,576 tokens (4.63%)** — not a meaningful differentiator.

----

Rate limit is the metric that decides whether a free model is usable in practice. Two-chapter answer: what our own router enforces, and what upstream enforces.

```yaml
-router_settings:
-  routing_strategy: simple-shuffle
-  enable_pre_call_checks: true
-  num_retries: 2
-  allowed_fails: 1
-  cooldown_time: 65
-  retry_after: 1
```

-| Group | Per-deployment limit | Notes |
-|---|---|---|
-| `free-mode` (Gemini) | `rpm: 1–15` per deployment | Deliberately low; multiple API keys multiply the ceiling. `gemini-3.1-pro-preview`, `gemini-2.5-pro`, `gemini-3.8-flash` pinned at **`rpm: 1`** because they are quota-gated. |
-| `free-mode-large` (Gemini) | `rpm: 1–15` per deployment | Same keys, used as the context-window overflow target. |
-| `paid-mode` (Moonshot) | `rpm: 5000`, `tpm: 3000000` | Highest ceiling in the config. |
-| **`free-HighTPS_Context`** | **none declared** | **Gap:** no `rpm`/`tpm` on the two Nemotron deployments. The only governor is the 240 s per-request timeout plus OpenRouter's account cap below.
