# BENCH_GITHUBCOPILOT_GROK_GHCOPILOT_GROK

**Author:** GitHub Copilot using Grok 4.7
**Date:** 2026-11-11
**Owner marker:** `GHCOPILOT_GROK_README_OWNER`
**Collision:** `BENCH_GITHUBCOPILOT_GROK.MD` was already on `main`. That file is a different Grok author. This file is the suffixed drop from this session. It does not replace the other file.

Compiled index: [`README.md`](README.md).

## What this session measured

Host: NVIDIA GX10 / GB10, `aarch64`, about 121 GiB unified memory. vLLM `:8000` was already serving `Qwen/Qwen2.5-14B-Instruct` with `max_model_len=8192`. It was not restarted. No model was pulled. Disk at the check: 58G free.

Protocol: "Print the integers 1 through 200 separated by commas, without explanation."; temperature 0; 220-token cap; two sequential calls; Ollama `keep_alive: 3m`. Decode = `eval_count / eval_duration`. vLLM = completion tokens / wall.

| Model | This session | Visible text | Prior same-host figure |
|---|---|---|---|
| `llama3.2:1b` | 162.73 / **164.01** decode tok/s | 285 chars | 163.6 decode, coding 5/14 |
| `llama3.2:3b` | 92.47 / **93.5** | 285 chars | 93.8, coding 13/14 |
| `qwen2.5:3b` | 91.97 / **92.72** | 220 chars | 94.9, coding 7/14, tools PASS |
| `qwen2.5:7b` | 45.78 / **46.03** | counting text | ~46 |
| `qwen3.5:35b-a3b` | 73.36 / **73.8** decode | **0 chars both calls** | 76.4, coding 12/14, tools PASS, JSON through 261,652 tokens |
| vLLM Qwen2.5-14B | **8.0 / 8.02 E2E** | 220 completion tokens | 8.0 E2E, coding 5/14, tools broken |

The 35B row is a speed measurement, not a capability confirmation. A thinking model can decode quickly and return no visible answer. The prior coding pass still stands as a prior-session result, not as this prompt's result.

`llama3.3:70b` was not loaded. Prior report: 14/14 coding, 4.8 tok/s, tools PASS/FAIL, 42.52 GB.

## Status probes this session

No keys printed. Bodies not printed.

| Provider | Result | Classification |
|---|---|---|
| Groq `openai/gpt-oss-120b` | HTTP **200** with a User-Agent. Without one, Cloudflare **1010**. | Replenishing free on this probe. Zoo's 403 is not the current key's result. |
| Cerebras `gpt-oss-120b` | HTTP **402** | Disqualified. Spent trial, does not replenish. |
| Gemini `gemini-2.5-flash` | HTTP 200 | Replenishing free tier. Not nearly unlimited. |
| Mistral `open-mistral-nemo` | HTTP 200 | Answered. Freebuff calls the experiment tier recurring. Not proven unlimited. |
| Nous | HTTP 200, cost field not proven zero | Not qualified. Freebuff saw an OpenRouter cost floor. |

Groq tok/s in other files (268.3 / 301.4) is Freebuff's short harness, not this probe. Their 30 RPM / 1k RPD / 8k TPM / 200k TPD was not fetched from Groq's docs here, so Groq is **not ranked on RPM**.

## Third-party GX10 / GB10 web

Fetched this session. Not this host.

- HAOyezi, one ASUS GX10, dense Qwen3.8-27B: 38.6 tok/s best single-stream code gen; 24 tok/s everyday vLLM NVFP4 chat; 227.6 tok/s is 16-way aggregate. https://github.com/HAOyezi/qwen38-27b-dgx-spark-benchmarks
- Tokenstead, citing other Spark benches: Qwen 3.5 27B FP16 11.1 tok/s; GPT-OSS 120B FP8 41.9; Exxact `qwen3.5:35b-a3b` 48.2 with 17/17 tools. The 218 tok/s Qwen 3.6 figure is Atlas + NVFP4, not stock Ollama. https://tokenstead.ai/guides/dgx-spark-benchmarks-2026

## Category calls

- **LOCALLY RUN(TESTED):** the six rows above, plus the prior-report models listed in the README. Best balanced installed model remains `qwen3.5:35b-a3b` on the prior suite. Fastest raw local is `llama3.2:1b`.
- **LOCALLY RUN(THEORETICAL):** the web numbers above. Do not pull `gpt-oss:120b` on 58G free. KILO's `gpt-oss:20b` (~13 GB, their ~83 tok/s) is a candidate, not a measurement here.
- **FREE CLOUD (RAN):** Groq answered; OpenRouter Nemotron pair was measured in the prior report; Gemini and Mistral answered this session; Cloudflare answered in the earlier fan-out. None are unlimited.
- **FREE CLOUD(THEORETICAL):** other `:free` catalog rows, LLM7/Pollinations/OVH as cited by peers, Featherless and Nous disputed or billed. Cerebras disqualified.
- **PAID CLOUD (ran):** Mercury mean ~547.8 on a 14-token prompt; Grok ~99 on a ~1,257-token prompt; DeepSeek ~62 and Kimi ~34 are reasoning-token e2e, not useful answers. Those are prior JSON rows, not re-run here.
- **PAID CLOUD(THEORETICAL):** Moonshot 5,000 rpm / 3M tpm is a LiteLLM client cap. Gemini 1–15 rpm in the yaml is a client cap. Advertised Cerebras 1,500+ tok/s is not runnable on a 402 key.

## Thoughts on files already on main

### `BENCH_GITHUBCOPILOT_GROK.MD`

Not this session. Wrong hardware. vLLM here is 8.0 E2E on the GB10, context 8,192, tools broken. Lightning ~52 and Super-VL ~20 match the short JSON, not the 102.0 / 35.9 table pair. Mercury 548 is not a long-context ranking. Groq RPM in that file is unsourced by this session.

### `BENCH_FREEBUFF_DEEPSEEK_V4_1_FLASH.MD`

Best audit on `main` at this compile. Correct host, correct metric split, correct `free-mode*` warning, correct Cerebras 402, correct router-cap reading of Moonshot. Section 11 applies the replenishing-credit rule properly. Groq 268.3/301.4 is their harness. Their RPM numbers stay labeled as theirs. User says FREEBUFF is still in progress, so a later edit may supersede rows this file cites.

### `BENCH_OPENCODE_NEMOTRON_HIGH_TPS_CONTEXT.md`

Agrees Groq beats Nemotron Lightning on short e2e, and that Lightning failed exact JSON near 778K while Super-VL passed 983,054. That split is better than a collapsed "both retained facts" sentence. Do not adopt their local 120B pull.

### `BENCH_ZOOCODE_PAID_MODE.MD`

`27d0b61` fixed the alias intro: two measured members, not three. Still wrong on two points. Section 7 still calls the RTX 5070 split the strongest hardware analysis. Section 5.2 still disqualifies Groq on HTTP 403. This session's User-Agent retry was HTTP 200; the no-UA failure was Cloudflare 1010. Their cooldown note is right: `cooldown_time: 65` and `allowed_fails: 1` on a two-member zero-price pool can darken the alias. Adding ERNIE as a third member is their proposal, not a measured upgrade. ERNIE previously required reasoning.

### `BENCH_KILO_v1_NEMOTRON_AUDIT.MD`

Agrees the RTX 5070 label is stale and that vLLM is the weak local member. Agrees on the installed local ranking. Their Groq RPM/TPM ceiling and `gpt-oss:20b` pull are labeled as theirs. KILO_V2 was not on `main`.

### Absent

HERMES and KILO_V2 had not landed when this file was written. Nothing from them is adopted.
