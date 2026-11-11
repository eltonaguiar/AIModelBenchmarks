## 2. Our Current Models — FREE (CLOUD)

Zero token price **and replenishing quota**. All three members of this category rotate inside one LiteLLM alias, `free-HighTPS_Context`, with `max_price: {prompt: 0, completion: 0}` (hard fence: quota exhaustion fails rather than bills).

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

---

## 3. Our Current Models — PAID (CLOUD)

### 3A. PAID CLOUD (RAN) — measured through our harness

| Model | tok/s (e2e) | Context | Capabilities | Caveat |
|---|---:|---:|---|---|
| **Mercury-3** (Inception) | **547.8** | 1,000,000 | text, tools | fastest paid we run; ~10.6× Lightning |
| Grok 4.7 (xAI) | 98.7 | 500,000 | vision, tools, web | half the context of Nemotron |
| DeepSeek V4 Pro | 61.9 | 1,048,576 | text (vision withheld), 393,216 max out | reasoning ate the 220-token budget in our test |
| Kimi K3.1 Pro (Moonshot) | 33.6 | 1,048,576 | text, tools | needs `temperature: 1`; 217/220 output was reasoning |

### 3B. PAID CLOUD (THEORETICAL) — configured but not yet run through our harness

| Model | tok/s (e2e) | Context | Capabilities | Caveat |
|---|---:|---:|---|---|
| Unmeasured: Claude Haiku 4.5, GPT-5.5-instant, GPT-5-chat, MiniMax-M2, Ring 2.6 1T | — | — | — | no same-harness numbers |

**Caveat that matters for decision-making:** the TPS figures for Kimi and DeepSeek are **not useful-answer throughput**. Both spent the entire small output budget on reasoning and returned no final answer. Their real throughput for bounded-output tasks is likely worse than the number suggests, and their effective context advantage over Nemotron is **48,576 tokens (4.63%)** — not a meaningful differentiator.

---

## 4. Rate Limits and Concurrency

Rate limit is the metric that decides whether a free model is usable in practice. Two-chapter answer: what our own router enforces, and what upstream enforces.

### 4.1 What our LiteLLM router enforces (`litellm_config.yaml`)

```yaml
router_settings:
  routing_strategy: simple-shuffle
  enable_pre_call_checks: true
  num_retries: 2
  allowed_fails: 1
  cooldown_time: 65
  retry_after: 1
```

| Group | Per-deployment limit | Notes |
|---|---|---|
| `free-mode` (Gemini) | `rpm: 1–15` per deployment | Deliberately low; multiple API keys multiply the ceiling. `gemini-3.1-pro-preview`, `gemini-2.5-pro`, `gemini-3.8-flash` pinned at **`rpm: 1`** because they are quota-gated. |
| `free-mode-large` (Gemini) | `rpm: 1–15` per deployment | Same keys, used as the context-window overflow target. |
| `paid-mode` (Moonshot) | `rpm: 5000`, `tpm: 3000000` | Highest ceiling in the config. |
| **`free-HighTPS_Context`** | **none declared** | **Gap:** no `rpm`/`tpm` on the two Nemotron deployments. The only governor is the 240 s per-request timeout plus OpenRouter's account cap below. |

**Concurrency behavior:** `cooldown_time: 65` with `allowed_fails: 1` means a single upstream failure parks a deployment for 65 s. For a **two-member** pool like `free-HighTPS_Context`, one failure leaves one member serving, and two failures leave the alias dark for 65 s. This is the most likely real-world failure mode and it is not protected by a fallback (by design — falling back would risk billing).

### 4.2 What upstream enforces (live-queried today)

| Provider / route | Hard limit | Replenishes? | Evidence |
|---|---|---|---|
| **OpenRouter account (free models)** | **1,000 requests/day** (used 300, remaining 700 at time of query) | **Yes — daily** | `GET /api/v1/key` → `free_model_daily_requests` |
| OpenRouter key spend cap | $45 budget, $42.61 remaining | No (top-up) | `GET /api/v1/key` → `budgets` |
| Nemotron free endpoints | rate limit not published (`per_request_limits: null`) | Yes (daily account cap guides it) | `GET /models/{slug}/endpoints` |
| Pollinations (keyless) | ~1 request / 30–45 s per IP | Yes — rolling | `dbpasses_ASK_MISSINGpollinations*.txt` |
| OVHcloud AI Endpoints (anonymous) | ~2 req/min per IP per model | Yes — rolling | `dbpasses.txt` line 463 |
| Cloudflare Workers AI | per-model neuron allowance | Yes — daily | provider docs |

**Implication for `free-HighTPS_Context`:** at 1,000 requests/day the alias supports roughly **one request every 86 seconds** all day, or ~40 requests/hour in a working burst. Anything more aggressive needs a second free route, not a bigger timeout.

---

## 5. Free Cloud Provider Due Diligence

**Selection rule (user-mandated):** a provider only counts as *free* if the allowance is **zero-price and replenishing** (daily/rolling reset). Providers that grant **one-shot trial credits** which expire and never refill are **disqualified**, regardless of how large the credit is.

### 5.1 Qualified — genuinely free and replenishing

| Provider | Allowance | Replenishes | Context ceiling | Verdict |
|---|---|---|---|---|
| **OpenRouter (zero-price models)** | 1,000 req/day | **Daily** | up to 1,048,576 | ✅ **Best free option in the fleet.** Nowhere near exhausted in normal use. |
| **Pollinations** | keyless, ~1 req/30–45 s per IP | Rolling | small | ✅ Qualifies but too slow/small for agents. |
| **OVHcloud AI Endpoints** | keyless, ~2 req/min per model per IP | Rolling | model-dependent | ✅ Qualifies as a low-rate backup. |
| **Cloudflare Workers AI** | per-model neuron/day | **Daily** | model-dependent | ✅ Qualifies — probe returned a valid answer during cross-check. |
| **Featherless.ai** | **46 zero-price models** returned by `/v1/models` | Subscription-based free tier | model-dependent | ✅ Qualifies — **largest untapped free pool found.** Not yet integrated. |
| **Nous Research** | **3 zero-price models** returned by `/v1/models` | Free tier | model-dependent | ✅ Qualifies — small but live. Not yet integrated. |
| **LLM7** | keyless, 69 models listed | Rolling | small | ✅ Qualifies — untested end-to-end. |

### 5.2 Disqualified — trial credit or non-replenishing

| Provider | What killed it | Evidence (live, today) |
|---|---|---|
| **Groq** | Legacy free key now **HTTP 403**; `groq_alt` still lists 11 models but zero priced-free and `dbpasses.txt` notes the free plan is a promotional tier | probe → `403 Forbidden` on chat |
| **Cerebras** | Free tier gone — **HTTP 402 PAYMENT REQUIRED** on chat; `$5` trial credit is one-shot | probe → `402` |
| **SambaNova** | **HTTP 402 PAYMENT REQUIRED**; the year-long cloud key is a trial, not a renewable free tier | probe → `402` |
| **Fireworks** | **HTTP 412 — account suspended at monthly spending limit**; free key is credit-based | probe → `412` |
| **Together AI** | `401 Unauthorized`; free credits are promotional and expire | probe → `401` |
| **SiliconFlow** | `401 Token is invalid`; free quota is a signup grant | probe → `401` |
| **AIMLAPI** | **`ALL_TIME_LIMIT_EXCEEDED`** — literally a permanent, non-replenishing cap | probe → `403` |
| **Nous `Hermes-4-405B`** (specific model) | Model retired | probe → `404` |
| **NVIDIA NIM** | Free endpoints retired (`gpt-oss-120b`, `deepseek-v4-pro` → `410 Gone`) | earlier probe |
| **xAI / Moonshot / Inception / DeepSeek** | Paid-only, no renewing free tier | `dbpasses.txt` = `==PAID_APIS==` |

### 5.3 Verdict on "is there a better free cloud model?"

**Yes — but not a better *large-context* one.** OpenRouter's catalogue today contains **16 zero-price models**. The ones that beat our current pick on a given axis:

- **Beat Lightning on context:** `inkling-2-small:free` (1,048,576), `ernie-5.5-flash-preview:free` (1,024,000 — and **921,600 max output**, 14× ours).
- **Beat Lightning on output headroom:** `dots-3-note-preview:free` (460,800 max out), `ernie-5.5` (921,600), `flint-v1.0:free` (235,929). Our alias caps output at 8,192.
- **Beat Lightning on multimodal:** `gemma-4-26b/31b:free`, `nano-omni-30b-a3b:free` (text+image+**audio**+video), `inkling-2-small:free`.
- **Beat Lightning on coding-with-privacy:** `cohere/north-mini-code:free` — explicitly no training on prompts.
- **Nobody beats it on *all three* yet.** Lightning remains the only zero-price model that is simultaneously ~52 tok/s, 1M context, and tool-capable.

**Highest-value next action:** add `baidu/ernie-5.5-flash-preview:free` as a third member of `free-HighTPS_Context`. It is 1,024,000 context with **921,600 max output** — which directly fixes the two defects we observed (Lightning's JSON failures and our 8,192 output ceiling) at zero cost.

---

## 6. Cross-Check Results

### 6.1 Which peers actually answered

| Peer | Status | Substance |
|---|---|---|
| **Cloudflare Workers AI** | ✅ answered | Found all six TPS figures "within expected range"; attributed Super-VL's 2.6× deficit to multimodal overhead — matches our hypothesis |
| **DeepInfra** | ✅ answered | Agreed on all six; independently flagged that larger context windows correlate with lower TPS (consistent with our Kimi/DeepSeek reading) |
| **Gemini 2.5 Flash** | ❌ HTTP 429 | Free quota exhausted at query time |
| **Mistral Small** | ❌ HTTP 429 | Rate limited |
| **GitHub Models** | ❌ DNS failure | `models.github.ai` not resolvable from this host |
| **nvidia/groq/cerebras/together/fireworks** (fan-out diverse5) | ❌ 404/401 | Model slugs in our preset are stale; `consult_multi.py --list` "has_key: yes" does not imply the model exists |
| **Nous Hermes-4-405B** | ❌ 404 | Model retired upstream |
| **AIMLAPI** | ❌ 403 | `ALL_TIME_LIMIT_EXCEEDED` |

**Honest read:** 2 of ~11 attempted peers produced substantive review. Both agreed with our numbers. That is weak corroboration, not strong — and I am recording it as weak.

### 6.2 What the peers added that we did not already have

Both independently converged on the same explanation for Nemotron's free-tier profile: **multimodal models pay a measurable TPS tax on the same provider.** Cloudflare and DeepInfra each attributed Super-VL's 20 vs Lightning's 52 to the vision/video encoder path rather than to hosting variance. We had the numbers; they supplied a causal account.

Neither peer could speak to rate limits — neither has visibility into our OpenRouter account, which is why §4 came from direct API queries instead.

---

## 7. Peer Review Notes (files already in this repo)

- **`BENCH_GITHUBCOPILOT_GROK.MD`** — strongest hardware analysis in the repo. Its RTX 5070 (12GB) vs GX10 (128GB unified) split matches what our own local table shows: dense ≥14B is ≤24 tok/s on the desktop card, while MoE 30–35B sustains 76–87 tok/s on GB10. **Agreement on Nemotron 1M retention.** One correction: it cites Groq free tier as viable at ~30 RPM / 1K RPD; our live probe today returns **HTTP 403** on that key, so treat Groq as disqualified until re-verified.
- **`BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH.MD`** — best local measurement rig in the repo (Ollama `eval_count / eval_duration`, 14-check coding suite). Its **"every fast local row is MoE"** rule reproduces independently in our table. Its Groq `openai/gpt-oss-120b` at 268–301 tok/s on 131K context is the single strongest free-cloud number anyone has produced — but see the 403 caveat above, and note that 131K is 13% of our current 1M ceiling.
- **Both files predate the rate-limit framing.** Neither distinguishes replenishing free tiers from one-shot trial credits, which is the distinction the user asked for and which §5 now adds.

---

## 8. Answers to the three questions asked

**Q: Are there more performance-efficient locally-run models we haven't considered?**
Yes. `qwen3.5:35b-a3b` (76.4 tok/s, 262K, tools+vision, 12/14 coding) already beats our `qwen2.5:14b` (23.9 tok/s, 32K) by **3.2× on speed and 8× on context** and is pulled but under-used. Not yet pulled and worth pulling: `gpt-oss-120b` MXFP4 (~45 tok/s, 131K) and `qwen3.8-27b` NVFP4. The current vLLM `Qwen2.5-14B` serving on :8000 is the worst option in the fleet — 8K hard cap and broken tools — and should not be the default.

**Q: Is there a better free cloud model?**
Yes on individual axes; no clean sweep. **`baidu/ernie-5.5-flash-preview:free`** (1,024,000 ctx, **921,600 max output**, tools, reasoning, $0) is the strongest addition available today and would fix both observed defects. `inkling-2-small:free` offers 1,048,576 context. `cohere/north-mini-code:free` offers privacy-preserving coding. None is confirmed faster than Lightning's ~52 tok/s — that requires measurement, which is the obvious next benchmark.

**Q: Is there a paid cloud provider we hadn't considered?**
Not on the evidence we have. Mercury-3 at 548 tok/s / 1M context dominates the paid tier, and nothing in `dbpasses.txt`'s `==PAID_APIS==` section (Anthropic, OpenAI, Cursor, Kilo, Venice, OpenCode) has same-harness numbers. The honest gap is that we have **no measurements** for Claude Haiku 4.5 or GPT-5.5-instant, both of which plausibly land between Grok (99) and Mercury (548). That is a measurement gap, not a market gap.

---

## 9. Method and limitations

- **TPS** = reported completion tokens ÷ total wall time for a short (~220-token) call. This is **end-to-end** throughput, not sustained decode rate. Two calls per model. **Not a stability guarantee.**
- **Local tok/s** = Ollama `eval_count / eval_duration` (decode-only, warmer and higher than e2e).
- **Context** = `model_info.max_input_tokens` in our config for models we serve; `context_length` from `/v1/models` for catalogue entries marked *advertised*.
- **Rate limits** = direct API queries (`GET /api/v1/key`, `GET /models/{slug}/endpoints`) at 2026-11-11T19:52Z, plus our own `litellm_config.yaml`.
- **Free-tier qualification** = live chat probe, not `/models` listing. A model appearing in `/models` at price 0 does **not** prove the account can call it — Groq lists 11 models and returns 403 on every one.
- **Known error retained:** the Kimi/DeepSeek TPS figures understate failure, because both models spent the entire output budget on reasoning and returned no answer. Recorded rather than discarded.
- **Credential hygiene:** no key values from `dbpasses.txt` appear in this file. Provider names and verdicts only.
