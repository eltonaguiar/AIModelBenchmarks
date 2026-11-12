# AI Model Benchmarks

**Owner marker:** `GHCOPILOT_GROK_README_OWNER`
**Compiled by:** GitHub Copilot using Grok 4.7
**Compiled:** 2026-11-11
**Host this compiler measured:** NVIDIA GX10 / GB10 Grace Blackwell, `aarch64`, Ubuntu, about 121 GiB unified memory. vLLM on this host is **not** an RTX 5070.
**Replaces:** the 12-byte `readme test` stub. Peer bench files are not replaced.

This is the normalized index. Every number is labeled as one of: **this session**, **same-host prior report** (`reports/local_model_benchmark_2026-11-11.md` and the 2026-11-11 JSON harnesses in the trading repo), **another agent**, or **third-party web**. Local Ollama decode tok/s and cloud/vLLM end-to-end tok/s are different metrics and are not ranked against each other.

**Free rule used here:** zero token price **and** a replenishing or nearly unlimited allowance. A one-time signup credit, a spent trial, or a grant that does not refill is **disqualified**, even if a catalog still lists the model.

**RPM rule:** a high tok/s model with an unknown or low request cap is not operationally better for fan-out. Unknown RPM stays "unknown — not ranked on RPM". A LiteLLM `rpm:` value is a client cap, not a provider grant.

## Expected drops on `main`

| Expected | File on `main` at this compile | Status |
|---|---|---|
| FREEBUFF | `BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH.MD` | Present through `dca3449`, including its live gateway coverage audit; user marks FREEBUFF done. |
| OPENCODE | `BENCH_OPENCODE_NEMOTRON_HIGH_TPS_CONTEXT.md` | Present; user marks OPENCODE done. |
| ZOOCODE | `BENCH_ZOOCODE_PAID_MODE.MD` | Present with live coverage audit at `092b79c`; user marks ZOOCODE done. Some conclusions remain disputed below. |
| KILO_V1 | `BENCH_KILO_v1_NEMOTRON_AUDIT.MD` | Present as of `6fda6f1`; user marks KILO_V1 done. |
| GHCOPILOT_GROK (this session) | `BENCH_GITHUBCOPILOT_GROK_GHCOPILOT_GROK.MD` | This follow-up adds the final runtime/config coverage cross-check. The unsuffixed file is a different author. |
| NEMO_HERMES | `BENCH_NEMOTRON_HERMES.MD` | Present from `d84f6a1`; user says Hermes is still pending/in progress. |
| KILO_V2 | — | **In progress; not on `main` at last fetch.** |
| LOCAL_ENGINES (Freebuff, this session) | `BENCH_GX10_LOCAL_ENGINES_2026-11-12_EST.MD` | New: Ollama-vs-vLLM head-to-head from one session on this host, full 26-model inventory with sizes, published GX10 corpus, and a dedicated `:4000` model-name plan. Corrects 4 peer claims. |

---

## LOCALLY RUN(TESTED)

Weights with nonzero size on this GX10, or the live vLLM process, that were actually generated. Size-0 Ollama `:cloud` stubs and `nomic-embed-text` are excluded.

Practical local cap is not an RPM. It is one resident runner, cold start, and GPU contention. Freebuff reports Ollama `NUM_PARALLEL=1`; that is their observation, not re-measured here. Disk at this session's check: 58G free. Do not pull a 65 GB weight on that margin.

### Speed — this session

Protocol: prompt "Print the integers 1 through 200 separated by commas, without explanation."; temperature 0; 220-token cap; two sequential calls; `keep_alive: 3m`. Ollama decode = `eval_count / eval_duration`. vLLM = completion tokens / wall. Source: `/tmp/ghcopilot_grok_local_bench.json`, measured by this GitHub Copilot / Grok 4.7 session on 2026-11-11. vLLM was not restarted. `llama3.3:70b` was not loaded.

| Model | Warm decode or E2E | Wall, call 1 / 2 | Visible answer | Capability on this prompt |
|---|---:|---:|---|---|
| `llama3.2:1b` | **164.01** decode (162.73 cold-ish) | 2.948 / 1.354 s | 285 chars | Fastest raw local. Not a quality pick. |
| `llama3.2:3b` | **93.5** decode (92.47 first) | 4.225 / 2.373 s | 285 chars | Speed only; coding not re-run. |
| `qwen2.5:3b` | **92.72** decode (91.97 first) | 4.01 / 2.391 s | 220 chars | Speed only. |
| `qwen3.5:35b-a3b` | **73.8** decode (73.36 first) | 10.065 / 3.051 s; load 6.757 s then 0 | **0 chars both calls** | Speed-only. Do not treat as a successful answer. |
| `qwen2.5:7b` | **46.03** decode (45.78 first) | 4.835 / 4.808 s | counting text | Already warm. |
| vLLM `Qwen/Qwen2.5-14B-Instruct` | **8.02 E2E** (8.0 first) | 27.488 / 27.435 s | 220 completion tokens | Different metric. `max_model_len=8192`. |

### Speed, coding, tools, context — same host, prior report, not re-run this session

Source: `reports/local_model_benchmark_2026-11-11.md`. These agree in order of magnitude with this session where both exist. They are still prior-session figures.

| Model | Warm decode tok/s | Coding | Tools | Context evidence |
|---|---:|---|---|---|
| `llama3.2:1b` | 163.6 | 5/14 | FAIL | Fast, weak. |
| `qwen2.5-coder:3b` | 96.0 | 7/14 | — | Not re-run. |
| `qwen2.5:3b` | 94.9 | 7/14 | PASS/PASS | JSON pass at 31,798 input tokens. |
| `llama3.2:3b` | 93.8 | 13/14 | PASS/FAIL | Giant-context markers retained; strict JSON key contract failed. |
| `qwen3:30b-a3b` | 87.0 | 0/14 | — | Long-context JSON FAIL. Catalog 262,144. |
| `phi4-mini` | 79.7 | 7/14 | — | Not re-run. |
| **`qwen3.5:35b-a3b`** | **76.4** | **12/14** | **PASS/PASS** | Verified complete JSON through **261,652** input tokens at allocated 262,144. Resident about 25.34 GiB. Best balanced installed model on prior evidence. |
| `qwen2.5-coder:7b` | 46.2 | 9/14 | — | Cited by Freebuff from the same report family. Not re-run here. |
| `qwen2.5:7b` | ~46 | 8/14 in Zoo's table | tools in Zoo's table | This session measured 46.03 decode. |
| `llama3.1:8b` | 43.8 | 4/14 | — | Not re-run. |
| `gemma3:12b` | 27.9 | 10/14 | no tools | Not re-run. |
| `qwen2.5:14b` | 23.9 | 11/14 | tools | Not re-run. |
| `deepseek-r1:32b` | 10.7 | 0/14 | tools, thinking | Reasoning ate the budget. |
| `qwen2.5:32b` | 10.5 | 9/14 | PASS/PASS | Not re-run. |
| `llama3.3:70b` | 4.8 | **14/14** | PASS/FAIL | Best coding, worst speed. 42.52 GB. Not loaded this session. |
| vLLM Qwen2.5-14B | 8.0 E2E | 5/14 | FAIL, parser options absent | Hard cap 8,192. This session reproduced 8.0–8.02 E2E. |

**Local ranking, capability-split:**

- Raw speed: `llama3.2:1b`.
- Balanced installed: `qwen3.5:35b-a3b`, on the prior coding/tools/context suite. This session confirms ~74 decode tok/s and does **not** confirm a visible answer on the counting prompt.
- Coding quality: `llama3.3:70b`, only if 4.8 tok/s is acceptable.
- Do not use vLLM as the local default until the 8,192 cap and tool parser are fixed. The web-Grok claim of "typical 30–60+ TPS" on a desktop RTX 5070 does not describe this process.

---

## LOCALLY RUN(THEORETICAL based on web research)

Not this host's ranking. No model was pulled. If a page was not fetched, it is not cited.

Fetched this session with `defuddle parse`:

- [HAOyezi Qwen3.8-27B on one ASUS GX10 / GB10](https://github.com/HAOyezi/qwen38-27b-dgx-spark-benchmarks). Author-measured, dense 27B, not this machine. Fastest single-stream they publish: **38.6 tok/s** code generation (SGLang + DFlash2, NVFP4). Everyday chat: **24 tok/s** vLLM NVFP4, SGLang chat about 15.9. FP8 was slower than Q4_K_M (7.8 vs 23 tok/s). 227.6 tok/s is a **16-concurrency aggregate**, not single-stream decode.
- [Tokenstead DGX Spark guide](https://tokenstead.ai/guides/dgx-spark-benchmarks-2026), which cites other benches rather than measuring this GX10. BridgeBench numbers they quote: Qwen 3.5 27B FP16 **11.1 tok/s**; GPT-OSS 120B FP8 **41.9 tok/s**; Gemma 4 31B FP16 **16.5 tok/s**. Exxact agent-bench numbers they quote: `qwen3.5:35b-a3b` **48.2 tok/s** with 17/17 tool score; Gemma 4 26B **52.7 tok/s**; Nemotron-3 Nano 30B **64.7 tok/s**. A spark-arena Qwen 3.6 35B figure of **218.85 tok/s** is a different Atlas + NVFP4 stack, not stock Ollama.

Not fetched this session, so not adopted: NVIDIA forum Qwen3.8 multi-Spark thread, `sparkbench.dev`, `gpuwerk.com`, llmconfigurator estimates, and the Qwen3.5-122B forum thread. Freebuff's published GB10 speeds for unpulled `gpt-oss:20b`, `qwen3.6:27b`, and `gpt-oss:120b` stay **their citations**, not this host's table.

**Do not pull** `gpt-oss:120b` or Qwen3.6 from this report. Freebuff's disk-blocker note (about 65 GB weight, HF cache larger than free disk) is consistent with the 58G free observed here. That cache size is Freebuff's measurement.

---

## FREE CLOUD (RAN)

Only replenishing or nearly unlimited zero-price routes with an actual generation in a cited run. None of these are unlimited. Daily or monthly caps decide fan-out, not the tok/s column.

| Route | What was actually run | Speed | Context | RPM / quota | Verdict |
|---|---|---|---|---|---|
| Groq `openai/gpt-oss-120b` | Freebuff short harness: prompt 86, output 220, e2e **301.4** and **268.3**. This session: HTTP **200** with a User-Agent; an earlier no-UA call got Cloudflare **1010**, which is not a provider billing 403. Wave-2 reply was truncated to 40 bytes. | Freebuff's number, not this session's tok/s. Nov 6 isolation bench was a different protocol (120b median 215 tok/s). | Catalog 131,072. Not a 1M route. | Freebuff states 30 RPM / 1k RPD / 8k TPM / 200k TPD. **Not fetched from Groq docs by this session.** Unknown for ranking purposes. | Qualifies as replenishing free on chat success. Not a high-fan-out winner until RPM is fetched. |
| OpenRouter `nvidia/nemotron-3.5-lightning:free` | Prior short JSON rows about 44.2–59.3 e2e, mean ~51.8. A separate Markdown table cites 102.0. Do not collapse them. | Short-protocol ~52 e2e. | Catalog 1,000,000. Verified retrieval through 261,568 in the sourced report. OpenCode: exact JSON **failed twice at ~778K**. | Zoo reports `GET /api/v1/key` = 1,000 requests/day, 700 remaining at their query. Freebuff says 20 RPM and 1,000 RPD because the key has lifetime credit. This session did not re-query the key endpoint. | Best measured free **long-context** route, not the fastest free route. Format at huge context is unreliable. |
| OpenRouter `nvidia/nemotron-3.5-super-vl:free` | Prior short rows ~19.7–20.9 e2e. Markdown table also cites 35.9. | ~20 e2e on the short protocol. | OpenCode: JSON passed at 778,824 and **983,054** tokens. | Same OpenRouter daily cap. | Slower, better behaved at huge context in the sourced report. |
| Gemini free tier | This session: `gemini-2.5-flash` HTTP 200. Freebuff: `gemma-4-31b-it` answered in 33 s, thinking-heavy. Their Gemini 3.5 Flash short call returned 9 and 6 tokens because thinking ate the budget. | No valid useful-answer tok/s. | Metadata 1,048,576 / 65,536 output in the prior report. Not a measured generation at that size. | LiteLLM client caps: flash-lite 10, several 3.x models 15, some pro models 1. Three independent flash keys can be 15 rpm only if the keys are independent. Provider RPM not re-fetched. | Qualifies as replenishing free. Not nearly unlimited. Do not score thinking-budget failures as speed. |
| Mistral experiment tier | This session: `open-mistral-nemo` HTTP 200. Freebuff: `ministral-3b-latest` in 0.42 s. | Their 0.42 s is an 8-token probe, not the 220-token harness. | Not measured here. | Not fetched. | Qualifies as recurring experiment-tier free on Freebuff's classification plus this session's 200. Not proven nearly unlimited. |
| Cohere trial | Freebuff only: `command-a-03-2025` OK. 1,000 calls/month, 20 RPM, recurring, **non-commercial**. | Not measured by this session. | Not measured. | 1,000/month is replenishing but small. | Limited replenishing, not nearly unlimited. Do not put it in the champion row. If the trial itself expires, reclassify as disqualified. |
| Cloudflare Workers AI | This session's earlier fan-out got substantive replies from Llama 3.1 8B and Llama 3.3 70B FP8. Freebuff could not probe their stored key because no account ID was wired. | No tok/s measured. | Not measured. | Zoo says per-model neurons/day. Not fetched from Cloudflare docs here. | Qualifies in principle and answered on the fan-out path. Not ranked on speed. |

`ROCKETSPEED-FREE` is documented as $0 upstreams, but Freebuff says the alias can fall back to paid. A free-looking LiteLLM group is not proof of zero cost. `true-free-tools` is local, not cloud.

---

## FREE CLOUD(THEORETICAL)

Catalog claims, other agents' unread or ungenerated routes, and disputed $0 listings. Not a speed ranking.

| Candidate | Why it is only theoretical | How to treat it |
|---|---|---|
| OpenRouter `:free` models not generated for speed: `nemotron-3-ultra-550b-a55b:free`, `ernie-5.5-flash-preview:free`, `flint-v1.0:free`, `cohere/north-mini-code:free`, Gemma 4 26B/31B `:free` | Price 0 in other agents' catalogs. ERNIE previously returned 400 because reasoning is mandatory. Inkling-2-small previously returned 403 (approved harness required). | Do not add them to `free-HighTPS_Context` on this report alone. |
| Groq `qwen3.8-27b`, `gpt-oss-20b`, `qwen3.6-27b` | Freebuff lists them as free-plan models. This session did not time them. | Same unknown RPM caveat as Groq 120B. |
| LLM7, Pollinations, OVH anonymous | Other agents call them keyless and replenishing, with very low RPM. This session did not re-probe. | Backup only. Not a quality or context upgrade. |
| Cloudflare account-ID path | Qualifies in principle at 10,000 neurons/day per Freebuff, but their stored key could not be probed. | Fix wiring before counting it as standing capacity. |
| Featherless | **Disputed.** Zoo qualifies it from a `/models` listing of 46 zero-price models. Freebuff says nothing is callable without a paid plan. Zoo's own method says a catalog listing is not proof. | Not qualified. |
| Nous | **Not qualified.** Freebuff's probe returned an OpenRouter cost floor. This session got HTTP 200 and did not prove `cost = 0`. The "USE ONLY FREE MODELS" label is not a $0 guarantee. | Do not route as free. |
| SambaNova | Freebuff: catalog OK, chat 401. OpenCode's theoretical section is more optimistic. | Conflict. Not qualified until a chat probe succeeds. |
| Kilo `qwen3.8-27b:free`, OpenCode Zen free prompts | Named by Freebuff as signup options. No KILOCODE file is on `main`. Zen prompts may be used for training. | Not measured. |

### Disqualified — do not call these free

Confirmed or attributed, not copied blindly:

- **Cerebras:** this session HTTP **402** Payment required. Freebuff: $5/30-day trial, credit spent, does not replenish. OpenCode's 1,500+ tok/s figure is an advertisement, not a run.
- **One-time grants, per Freebuff's audit:** Scaleway 1M tokens/model, Alibaba/DashScope 1M tokens valid 90 days, Nebius $1/30-day card trial, SiliconFlow signup credit plus geo restriction.
- **Dead or metered credentials, attributed to the agent who probed them:** Together 401, Fireworks 412, GitHub Models retired, Hyperbolic 404, Hugging Face and DeepInfra billed on Freebuff's probe, NVIDIA NIM prototyping credits and timeouts, AIMLAPI `ALL_TIME_LIMIT_EXCEEDED` in Zoo's file.
- **Zoo's Groq disqualification is rejected.** Their 403 does not survive a User-Agent retry on the current Groq free key. A Cloudflare 1010 is not evidence that the free plan ended.

---

## PAID CLOUD (ran)

Source: `reports/free_high_tps_context_2026-11-11.json` and Freebuff's same-day short harness. This session did not re-run paid calls. None of these are ranked above local on RPM, because provider RPM was not fetched.

| Model | Measured e2e | Protocol caveat | Catalog context | Useful? |
|---|---:|---|---:|---|
| Inception Mercury-3 | 360.83 and 734.68, mean ~547.8 | **14-token prompt.** Tiny-prompt burst. | 1,000,000 | Fastest short-call paid row. Not a fair 10.6× win over Lightning. |
| xAI Grok 4.7 | 99.13 and 98.30 | Prompt about 1,257 tokens, output 256/275. | 500,000 | Fairest long-prompt paid row in that JSON. |
| DeepSeek V4 Pro | 61.46 and 62.28 | Empty visible text. Reasoning consumed the 220-token budget. | 1,048,576 | Not useful-answer tok/s. |
| Kimi K3.1 Pro | 35.64 and 31.59 after temperature 1 | First calls HTTP 400 on temperature. Mostly reasoning tokens. | 1,048,576 | Not useful-answer tok/s. |

Nov 6 `ROCKETSPEED-FAST` also ran paid `deepseek-chat` at 6/6, 0.88 s, 4 tok/s. That is a different protocol. Gemini 3 Flash preview in that note was labeled free tier, not paid.

---

## PAID CLOUD(THEORETICAL)

Not measured on the 220-token harness.

| Item | What is known | What is not |
|---|---|---|
| Moonshot `rpm: 5000`, `tpm: 3000000` | LiteLLM **client** cap on `moonshot-v1-8k` in `litellm_config.yaml`. | Not Moonshot's published limit. Not transferable to `kimi-k3.1-pro`. |
| Gemini client `rpm: 1` to `rpm: 15` | Operator-set in the same yaml. | Not a fresh Google doc fetch. |
| Claude Haiku 4.5, GPT-5.5-instant, GPT-5-chat, MiniMax-M2, Ring 2.6 1T | Present as unmeasured paid names in Zoo's table. | No same-harness number. |
| Cerebras / Groq paid / SambaNova / Fireworks / Together advertised tok/s | OpenCode quotes third-party ranges from ~480 to ~1,780 tok/s. | Our Cerebras key is 402. Do not adopt. |
| `ROCKETSPEED-FAST` paid members | xAI Grok-4-fast, DeepSeek chat, Gemini 3 Flash are mixed into a "fast" alias. | Alias membership is not a benchmark. |

---

## Normalized picks

1. **Local default:** `qwen3.5:35b-a3b` for balance, with the empty-answer caveat from this session's counting prompt. Use `llama3.2:3b` when 13/14 coding and ~93 tok/s matter more than 262K context. Use `llama3.2:1b` only for raw speed.
2. **Free cloud, single agent, speed over context:** Groq `gpt-oss-120b`, if the free plan is still answering. Cap fan-out until Groq's RPM page is fetched. Do not trust a no-User-Agent 403/1010.
3. **Free cloud, long context:** OpenRouter Nemotron pair inside `free-HighTPS_Context`, remembering the daily cap and Lightning's JSON failures near 778K. Super-VL is the one that passed the large JSON in the sourced report.
4. **Paid short burst:** Mercury. Do not quote 548 tok/s as a long-context result.
5. **Paid long prompt already measured:** Grok 4.7 at ~99 e2e on a 1,257-token prompt.
6. **Not a pick:** Cerebras, Featherless-as-free, Nous-as-free, vLLM-as-configured, and any unpulled 65 GB local weight.

---

## Service health spot-check

After the operator reported recovery, vLLM `http://127.0.0.1:8000/v1/models` and LiteLLM `http://127.0.0.1:4000/health/liveliness`, `/health/readiness`, and `/v1/models` all returned HTTP 200; both ports were listening. vLLM listed `Qwen/Qwen2.5-14B-Instruct`, and LiteLLM listed 31 aliases including `paid-mode`. This verifies API reachability only; no `paid-mode` completion was sent, so routing to its upstream was not re-tested. This check also says nothing about GitHub Copilot's Auto model selection or account billing. No services were restarted or changed.

## Live gateway coverage

Read-only snapshot on 2026-11-11. `/v1/models` exposed **31 aliases**; `/model/info` exposed **170 deployments / 73 distinct upstream model IDs**. The local YAML also contained 31 aliases and 170 deployment rows. Alias-set comparison and exact `(alias, upstream model)` pair comparison both had **zero differences**. This verifies mapping coverage at that moment, not upstream availability, billing, or the completeness of provider catalogs.

| Route class | Aliases | Coverage reading |
|---|---|---|
| Local-only | `true-free-tools`, `true-free-tools-fast` | Ollama/vLLM deployments; no cloud quota. Live `/model/info` shows three deployments per group. |
| Dedicated zero-price long context | `free-HighTPS_Context` | Two OpenRouter Nemotron deployments, 900K input / 8,192 output, 240-second route timeout, explicit zero-price guard and no paid fallback. |
| Free-named or hybrid pools | `free-mode`, `free-mode-fast`, `free-mode-large`, `free-mode-tools`, `ROCKETSPEED-FREE`, `ROCKETSPEED-FAST`, `hybrid-model`, `hybrid-model-large` | Mixed providers, duplicated keys/models, per-deployment client caps and some paid-capable/fallback routes. Alias labels are not a zero-cost guarantee. Freebuff's latest audit reports 44 `free-mode` deployments and identifies dead slots from individual provider probes. |
| Paid pools | `paid-mode`, `paid-mode-fast`, `paid-mode-large` | Paid-capable routes; a healthy proxy/model-list response does not test any specific paid upstream or estimate charges. |
| Direct/model-specific aliases | `cloudflare-llama`, `nvidia-deepseek-v4-pro`, `openrouter-ring-1t`, `claude-haiku-direct`, `deepseek-chat-direct`, `novita-deepseek-v4-pro-direct`, `XX_groq`, `XX_gemini`, `XX_deepinfra`, `XX_mistral`, `XX_nous`, `XX_cloudflare`, `XX_ollamacloud`, `XX_deepseek`, `XX_grok`, `XX_kimi`, `XX_inception` | Seventeen direct aliases span free, paid, local-adjacent, and unverified routes; classify by deployment/provider, not alias alone. |

### Final suggestions (documentation only)

1. Measure the existing multi-deployment free pools with one fixed low-token prompt per deployment before adding providers. Keep per-model success, latency, client RPM/TPM, provider quota, and cost evidence separate; aggregate group metadata is not an upstream guarantee.
2. Triage Freebuff's 17 reported dead deployments. Reconfirm each error against its exact route/key once before proposing removal; 401/404/402/412 results are evidence for that probe, not proof every provider account is globally dead.
3. Do not promote ERNIE into `free-HighTPS_Context` yet: earlier calls failed because reasoning was required, and Freebuff's latest tool-schema probe got a 429. Require a successful long-context JSON/tool test and explicit zero-price verification first.
4. Treat `command-a-03-2025` as a limited, non-commercial Cohere trial, not a general-purpose free champion; verify the terms before routing shared workloads to it.
5. Keep health checks observational. A failed readiness probe should alert an operator; do not automatically restart the proxy while agents may be using it without a restart policy and coordination.
6. This is a point-in-time mapping audit, not a config change. The worktree's `litellm_config.yaml` was already modified; this task did not edit or stage it.

## Thoughts on files already on `main`

### `BENCH_GITHUBCOPILOT_GROK.MD` — not this session

Different Grok author. Hardware claim is wrong for this machine: vLLM is on the GB10 at **8.0 E2E**, context 8,192, coding 5/14, tools broken. It is not a 12 GB RTX 5070 at 30–60 tok/s. Its Lightning ~52 and Super-VL ~20 match the short JSON rows, not the 102.0 / 35.9 Markdown-table pair. Mercury ~548 is the 14-token mean. DeepSeek ~62 and Kimi ~34 are reasoning-token e2e, not useful answers. Groq "30 RPM" is unsourced by this session. Qwen3.6 at 90–110 tok/s and local gpt-oss-120b at ~45 tok/s were not measured here.

### `BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH.MD`

Best provider/key audit on `main`, now with a live route coverage section. Its 31 aliases, 170 deployments, and 73 upstream IDs match this session's read-only runtime/config comparison. The dead-slot findings are attributed to Freebuff; no config changes were made here. Groq 268.3/301.4 is their harness and matches `reports/model_bench_three_categories_2026-11-11.json`. Rate-limit values remain attributed to their probes, not provider documentation fetched by this session. Reconfirm per route before removing a deployment.

### `BENCH_OPENCODE_NEMOTRON_HIGH_TPS_CONTEXT.md`

Agrees on the local order and on Groq beating Nemotron Lightning on short e2e. Also agrees Lightning failed exact JSON at ~778K while Super-VL passed 983,054. That protocol split is better than Zoo's collapsed "both retained facts at 779K". Do not adopt the recommendation to pull local `gpt-oss-120b`: disk margin is too small. Their Groq RPM figure remains attributed, not independently fetched here. SambaNova catalog HTTP 200 is not generation proof and conflicts with Freebuff's chat 401.

### `BENCH_ZOOCODE_PAID_MODE.MD`

Its newest coverage audit correctly recognizes the 31 aliases and two-member Nemotron route. The remaining corrections are:

- The host line still adds a desktop RTX 5070. That split is the web-Grok error. vLLM's 8.0 E2E row in Zoo's own local table contradicts it.
- Section 2 says all free-cloud members rotate inside `free-HighTPS_Context`. Groq is a separate free route and the measured speed leader.
- The Groq 403 still contradicts this session's User-Agent retry (HTTP 200) and Freebuff's successful gpt-oss probe. Treat it as a key/probe-path-specific failure, not a provider-wide death verdict.
- The ERNIE addition suggestion is premature: earlier calls returned 400 because reasoning was mandatory, and Freebuff's later tool-schema probe got a 429. Keep it out of the long-context alias until it passes the required route tests.

Zoo's OpenRouter 1,000 requests/day observation is the right operational ceiling **if** their key query is accurate. This session did not repeat that HTTP call.


### `BENCH_KILO_v1_NEMOTRON_AUDIT.MD`

Agrees with this session on the hardware correction: vLLM is on the GB10 at 8.0 E2E, not an RTX 5070 at 30–60 tok/s. Their 90-second proxy timeout is the global LiteLLM default, but `free-HighTPS_Context` overrides it to 240 seconds; the latter's 113-second Super-VL retrieval was within that route timeout. Their Groq rate ceilings remain their claims, not provider documentation fetched here. Kilo's Mistral 429 and this session's separate `open-mistral-nemo` HTTP 200 are endpoint/key-specific results. Its GitHub Models claim conflicts with Freebuff's retired-endpoint finding. Requesty's 200/day is limited, not nearly unlimited; Feather's pricing/quota remains unknown. Treat `gpt-oss:20b` and ModelScope DeepSeek-V4.1-Flash as candidates, not measurements from this session.

### `BENCH_NEMOTRON_HERMES.MD`

Hermes lists `usb-mistral-nemo-12b` and `library-mistral-nemo-latest` as resident models, but neither appears in this session's live Ollama `/api/tags` response (26 tags). Treat these as unverified/nonresident here, not tested local candidates. Its Groq 403 is contradicted by this session's HTTP 200 retry and Freebuff's successful probe. NVIDIA/SambaNova catalog responses are not generation benchmarks; SambaNova free usability remains disputed by Freebuff's chat 401. The `gpt-oss:20b` pull and other theoretical options remain recommendations, not measurements from this session.

### Still pending

KILO_V2 is in progress and was not on `main` at last fetch. Hermes's report is present, but the user says it may still be in progress. OPENCODE, KILO_V1, FREEBUFF, and ZOOCODE are marked done by the user.

### `BENCH_GX10_LOCAL_ENGINES_2026-11-12_EST.MD` — Freebuff, this session

First file here that answers "Ollama or vLLM on *this* box" from one session instead of cross-report comparison.
Answers: Ollama wins single-stream **as configured** (74–85 tok/s MoE vs 8 tok/s bf16 8B); vLLM wins concurrency
(54.7 tok/s @4 vs 8.0 single; published 350 @16); vLLM's 8 tok/s is our precision + driver, not the engine.
Live bug found: `litellm_config.yaml:340` routes `ULTRA_FREE_TOOLS` to a vLLM model that is no longer served.
Methodological rule established: sequential multi-model Ollama runs without unload measure memory residency, not speed.
