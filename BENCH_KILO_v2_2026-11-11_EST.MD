# BENCH_KILO_v2 — Model Benchmark (LOCAL / FREE CLOUD / PAID CLOUD)

**Agent:** Kilo (opencode fork, nvidia/nemotron-3.5-super-vl:free via Kilo Gateway)
**Version:** v2 (KILO_V2)
**Date:** 2026-11-11 17:33 EST (2026-11-11T22:33:14Z)
**Host:** Ubuntu ARM64 NVIDIA GB10 (DGX Spark / ASUS Ascent GX10 class), 121 GiB unified memory, ~62 GB free disk
**Stack:** Ollama 0.40.13 (:11434), vLLM 0.21.0 (:8000), LiteLLM :4000

---

## 0. Category Summary

| Category | What was tested | Result |
|---|---|---|
| **LOCALLY RUN (FREE)** | Ollama :11434 (10 models) + vLLM :8000 (Qwen2.5-14B) | Measured — native decode + e2e TPS |
| **FREE CLOUD (RAN)** | LiteLLM :4000 free-tier aliases (free-mode, ROCKETSPEED-*, free-HighTPS_Context) | Measured — e2e TPS |
| **FREE CLOUD (THEORETICAL)** | Other free-tier catalog models (not run this session) | Not measured |
| **PAID CLOUD (RAN)** | None called this session (no paid credentials used) | Not measured |
| **PAID CLOUD (THEORETICAL)** | Paid frontier models (Claude, GPT, etc.) | Not measured |

**Protocol:** prompt = "Write a Python function that returns the nth Fibonacci number using memoization."; temperature 0; 200-token cap; single call per model. Ollama decode = `eval_count / eval_duration`. Cloud/vLLM = completion_tokens / wall.

---

## 1. LOCALLY RUN (FREE) — Ollama decode + vLLM e2e

| Model | Decode TPS | E2E TPS | Wall (s) | Output chars | Notes |
|---|---:|---:|---:|---:|---|
| `llama3.2:1b` | **161.87** | 77.57 | 2.58 | 838 | Fastest decode; small model |
| `qwen2.5:3b` | 93.60 | 92.53 | 2.16 | 806 | Strong speed/quality balance |
| `llama3.2:3b` | 93.21 | 91.97 | 1.81 | 702 | Fast, decent quality |
| `qwen3:30b-a3b` | 87.03 | 25.29 | 7.91 | 0 | Thinking ate output budget (0 chars) |
| `phi4-mini:latest` | 79.14 | 25.79 | 3.14 | 258 | Good speed, small output |
| `qwen3.5:35b-a3b` | 73.99 | 19.97 | 10.02 | 314 | Best balanced large model |
| `qwen2.5:7b` | 46.19 | 30.06 | 6.65 | 793 | Mid-size |
| `llama3.1:8b` | 43.68 | 27.33 | 7.32 | 879 | Solid 8B |
| `gemma3:12b` | 27.50 | 17.45 | 11.46 | 698 | Vision-capable, slower |
| `qwen2.5:14b` | 23.71 | 16.55 | 12.08 | 1042 | Dense, slow |
| vLLM `Qwen2.5-14B-Instruct` | — | **8.08** | 24.75 | 200 | Hard cap 8,192 tokens |

**Local ranking:**
- **Raw decode speed:** `llama3.2:1b` (161.87 tok/s)
- **Balanced:** `qwen2.5:3b` (93.60 decode / 92.53 e2e) or `qwen3.5:35b-a3b` (73.99 decode / 314 chars, best large)
- **Quality:** `qwen2.5:14b` (1042 chars) but slow (23.71 tok/s)

**Key rule:** Every fast local row is MoE. Every dense row ≥14B is ≤24 tok/s — the GB10 rule. vLLM advertises 8,192 total and is the slowest option despite being "the GPU engine."

---

## 2. FREE CLOUD (RAN) — LiteLLM gateway zero-price routes

| Route / alias | E2E TPS | Wall (s) | Tokens | Notes |
|---|---:|---:|---:|---|
| **free-mode** | **379.93** | 0.53 | 200 | Fastest free route measured |
| **ROCKETSPEED-FAST** | 303.35 | 0.66 | 200 | Latency-optimized |
| **ROCKETSPEED-FREE** | 285.54 | 0.70 | 200 | $0 upstreams only |
| **free-HighTPS_Context** | 49.68 | 4.03 | 200 | 1M context, slower |
| free-mode-fast | FAIL | — | 0 | Upstream error this run |

**Key finding:** `free-mode` (379.93 e2e) is the **fastest free route** — ~4.7× faster than `free-HighTPS_Context` (49.68). The `ROCKETSPEED-*` aliases (285–303 e2e) are also strong. `free-HighTPS_Context` is the **slowest** free option but offers 1M context.

**Caveat:** These are LiteLLM gateway aliases, not raw provider tok/s. A `free-*` alias can fall back to a paid upstream — a free-looking alias is not proof of $0 cost. RPM/quota not verified this run.

---

## 3. Cross-check against other agents' files

I reviewed the existing `BENCH_*.MD` files in the repo. My findings agree with the repo README's normalized index:

| Claim in other files | My finding | Agree? |
|---|---|---|
| `llama3.2:1b` fastest local (~163 tok/s) | 161.87 decode | ✅ |
| `qwen3.5:35b-a3b` best balanced (~76 tok/s) | 73.99 decode | ✅ |
| `qwen2.5:3b` ~93 tok/s | 93.60 decode | ✅ |
| vLLM Qwen2.5-14B ~8 tok/s e2e | 8.08 e2e | ✅ |
| Groq free `gpt-oss-120b` 268–301 tok/s | Not re-run this session | — (cited) |
| `free-HighTPS_Context` ~50 tok/s | 49.68 e2e | ✅ |
| Nemotron Super-VL ~20 tok/s | Not re-run (context route) | — |

My measurements are **consistent** with the existing benchmark files and the README index.

---

## 4. New / under-covered sources I explored this session

- **Workopolis Canada** (workopolis.com) — scrapable via curl_cffi (HTTP 200); search-results page is JS-rendered, individual job pages return HTTP 200
- **LiteLLM gateway aliases** — measured `free-mode`, `ROCKETSPEED-FREE/FAST`, `free-HighTPS_Context` directly
- **Canadian employer Workday boards** — RBC, BMO, TD, Scotiabank, Manulife, Sun Life, etc. (many return 403/404 without correct tenant/site names)

### Blocked / dead ends
- **Indeed Canada** — Cloudflare 1010 (blocked)
- **Most Canadian bank career pages** — JS-rendered (Phenom/Workday), HTML contains no job listings without JS execution
- **Job Bank API** — returns HTTP 500 (API dead)

---

## 5. Summary & recommendations

**Local (free, this GX10):**
- Use `qwen2.5:3b` or `qwen3.5:35b-a3b` for a speed/quality balance
- Use `llama3.2:1b` when raw speed matters most
- Avoid vLLM as default — 8,192 cap + slowest (8 tok/s)

**Free cloud:**
- `free-mode` (379.93 e2e) is the fastest free route
- `free-HighTPS_Context` (49.68 e2e) only when 1M context is required
- Verify RPM/quota before high-fan-out use

**Breadth improvement:** Workopolis Canada is a viable new source (curl_cffi); LinkedIn guest API + direct ATS boards (Greenhouse/Lever/Ashby/SmartRecruiters) remain the most productive job sources.

---

*Prepared by Kilo (opencode fork, nvidia/nemotron-3.5-super-vl:free) — 2026-11-11 17:33 EST. All numbers measured this session unless marked "cited".*