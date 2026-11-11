# FreeLLM — LiteLLM Proxy Model Groups & Setup Guide

Port `4000` runs a **LiteLLM proxy** (`litellm --config litellm_config.yaml --port 4000`). It is NOT vLLM, not Ollama, not your project app. It is the single OpenAI-compatible routing layer (`http://localhost:4000/v1`) that fronts many upstream LLM providers behind invented virtual model names.

## How routing / naming works

- `model_name` = the virtual name clients ask for (e.g. `ULTRA_FREE_CLOUD`, `ULTRA_FREE_TOOLS`, `ZZZ_PAID_MODE`).
- `litellm_params.model` = the real upstream model id (e.g. `groq/qwen/qwen3.8-27b`, `ollama_chat/qwen2.5:14b`, `hosted_vllm/Qwen/Qwen2.5-14B-Instruct`).
- `litellm_params.api_base` = upstream endpoint. Cloud providers use their public API base; local ones use `http://localhost:8000/v1` (vLLM) or `http://localhost:11434` (Ollama).
- `litellm_params.api_key` = `os.environ/NAME` pulled from `~/dbpasses.txt` at launch by `tools/start_litellm_proxy.sh`. Never hardcoded in config.
- `router_settings.routing_strategy: simple-shuffle` — picks a random member of the group each request (weighted by `weight` if set), spreading load.
- `router_settings.fallbacks` — on failure (429/401/5xx/timeout) the failing member cools down (`cooldown_time: 65s`) and the next fallback group is tried.
- `context_window_fallbacks` — oversize requests fall to a larger-context group.

## Naming convention (binary)

Every group starts with either `ULTRA_` (genuinely free, verified zero-cost) or `ZZZ_` (can bill, legacy, mixed, or untrusted — avoid unless you mean to spend):

- **`ULTRA_*`** — free only. Use these for zero-cost contracts.
- **`ZZZ_*`** — not free / can bill / legacy. Retained for backward compat only.

## Model groups — mapped AIs, rotation priority, rules

### ULTRA — LOCAL only (never cloud, never paid; fail closed if locals down)

| Group | Mapped AIs (backends) | api_base | Fallback rule | Notes |
|-------|----------------------|----------|---------------|-------|
| `ULTRA_FREE_TOOLS` | `hosted_vllm/Qwen/Qwen2.5-14B-Instruct` (vLLM), `ollama_chat/qwen2.5:14b`, `ollama_chat/qwen3:30b-a3b` | `localhost:8000/v1` (vLLM), `localhost:11434` (Ollama) | NONE — fails loud if all 3 down | Tool-calling verified 2026-11-04. qwen3 needs max_tokens >= 1024. |
| `ULTRA_FREE_TOOLS_FAST` | `ollama_chat/qwen2.5:3b`, `ollama_chat/llama3.2:3b`, `ollama_chat/qwen2.5:7b` | `localhost:11434` | `ULTRA_FREE_TOOLS` | Latency-first local. Cold-start 3-23s after idle; `keep_alive: 24h`. |
| `ULTRA_FREE_LOCAL` | `ollama_chat/qwen3.5:35b-a3b` | `localhost:11434` | NONE | Best balanced local: 12/14 coding, tools PASS/PASS, vision, reasoning, 262K context, complete JSON through 261,652 input tokens. `keep_alive: 24h`. |
| `ULTRA_FREE_CODING` | `ollama_chat/qwen3.5:35b-a3b` | `localhost:11434` | NONE | Coding-focused alias of `ULTRA_FREE_LOCAL` (12/14 coding measured). Same vision + reasoning + 262K context. `keep_alive: 24h`. |

### ULTRA — CLOUD only (zero-price fenced via OpenRouter `max_price: {0,0}` or Groq free plan; never paid)

| Group | Mapped AIs | Fallback rule | Notes |
|-------|-----------|---------------|-------|
| `ULTRA_FREE_CLOUD` | Groq `gpt-oss-120b/20b`, `qwen3.8-27b`; OpenRouter `nemotron-3-ultra-a55b:free`, `cohere/north-mini-code:free`, `recursive/flint-v1.0:free`, `liquid/lfm-3-1.7b:free` | `ULTRA_FREE_CLOUD_LARGE` only | Strict free spillover; never falls back to paid. |
| `ULTRA_FREE_CLOUD_FAST` | Groq `gpt-oss-120b/20b`, `qwen3.8-27b`; OpenRouter free North, Flint, LFM | `ULTRA_FREE_CLOUD`, then `ULTRA_FREE_CLOUD_LARGE` | Latency-first free cloud. |
| `ULTRA_FREE_CLOUD_TOOLS` | OpenRouter `cohere/north-mini-code:free`, `recursive/flint-v1.0:free`, `liquid/lfm-3-1.7b:free` | NONE | Tool-capable free cloud; fails closed. |
| `ULTRA_FREE_CLOUD_LARGE` | OpenRouter `nemotron-3-ultra-550b-a55b:free` (900K configured input), `nemotron-3.5-lightning:free`, `nemotron-3.5-super-vl:free` (vision) | NONE | Long-context free selector, 900K input / 8,192 output configured. |

### ZZZ — can bill, legacy, mixed, or untrusted

| Group | Mapped AIs | Notes |
|-------|-----------|-------|
| `ZZZ_PAID_MODE` | Anthropic haiku, deepseek, moonshot, aimlapi paid, hypereal gpt-5.5-instant, openai gpt-4o-mini (quota exhausted), xiaomi mimo, xai grok, bluesmind premium | Premium frontier — can bill. |
| `ZZZ_PAID_MODE_FAST` | xai grok-4-fast, anthropic haiku, deepseek, bluesmind gpt-5-chat | Paid, >=50 tok/s — can bill. |
| `ZZZ_PAID_MODE_LARGE` | Anthropic haiku, deepseek, hypereal gpt-5.5, bluesmind long-context | Paid long-context — can bill. |
| `ZZZ_HYBRID_MODEL` | Groq, NVIDIA NIM, Gemini, GitHub Models, Hypereal | Mixed — can reach paid. Backward-compat alias. |
| `ZZZ_HYBRID_MODEL_LARGE` | Gemini, NVIDIA NIM, Fireworks, Hypereal | Mixed large — can reach paid. Backward-compat alias. |
| `ZZZ_BAD_LEGACY_MIXED_F_MODE` | Mixed cloud free + trial/paid | NOT a zero-cost guarantee. Opt-in compatibility only. |
| `ZZZ_BAD_LEGACY_MIXED_F_MODE_FAST` | Historical mixed fast set | Not zero-cost guaranteed. |
| `ZZZ_BAD_LEGACY_MIXED_F_MODE_TOOLS` | Historical mixed tool set | Includes paid-capable providers. |
| `ZZZ_BAD_LEGACY_MIXED_F_MODE_LARGE` | Old mixed long-context pool | Not zero-cost guaranteed. |

## On images / vision

No separate image group was created — mixing local and cloud in one group would break the local-only fail-closed contract. Vision capability is marked on the models that have it:
- `ULTRA_FREE_LOCAL` / `ULTRA_FREE_CODING` (qwen3.5:35b-a3b): `supports_vision: true`
- `ULTRA_FREE_CLOUD_LARGE` (nemotron-3.5-super-vl:free): `supports_vision: true`

Route image workloads to those groups directly rather than creating a mixed local/cloud vision alias.

## Local models on this machine (currently running)

- **vLLM**: `Qwen/Qwen2.5-14B-Instruct` on `http://localhost:8000/v1`. Used by `ULTRA_FREE_TOOLS`.
- **Ollama** (`ollama serve`): `http://localhost:11434`. Local `llama-server` instances on dynamic ports (`qwen2.5:14b`, `qwen3:30b-a3b`, `qwen2.5:3b`, `llama3.2:3b`, `qwen2.5:7b`, `qwen3.5:35b-a3b`, etc.). Used by all `ULTRA_FREE_TOOLS*` and `ULTRA_FREE_LOCAL*` groups.
- **LiteLLM proxy**: `http://localhost:4000`.

Ollama Cloud is NOT routed — its key is an SSH ed25519 public key, not a bearer token (skipped per config comment).

## Setup on another PC / archiving to GitHub

### Repo / remote situation
- `origin` = `git@github.com:eltonaguiar/findtorontoevents_antigravity.ca.git` (this repo).
- `freellm` remote = `/home/eaguiar2015/FREELLM` (a LOCAL path, NOT GitHub yet).
- `litellm_config.yaml` is tracked. `~/dbpasses.txt` is NOT tracked (outside repo, `chmod 600`). `.env.example` is gitignored and its real-key values were stripped to placeholders.

### Steps to publish as eltonaguiar/FREELLM on GitHub
1. Create the empty GitHub repo `eltonaguiar/FREELLM` (do NOT initialize with README).
2. In a clean worktree/clone containing only the FreeLLM files (`litellm_config.yaml`, `tools/start_litellm_proxy.sh`, `tools/litellm_smart_cooldown.py`, `DOCKER_VLLM.MD`, this setup doc, `.gitignore`):
   - `git remote add freellm https://github.com/eltonaguiar/FREELLM.git` (or git@github.com:eltonaguiar/FREELLM.git)
   - `git branch -M main`
   - `git push -u freellm main`
3. Do NOT push `~/dbpasses.txt`, `.env`, or any file containing real keys.

### Where to change the port
Edit `tools/start_litellm_proxy.sh`:
- `PORT="${PORT:-4000}"` — default port.
- Or pass `--port <N>` at launch: `bash tools/start_litellm_proxy.sh --background --port 5000`.
- Clients then point `base_url` to `http://localhost:<PORT>/v1`.

### If the new PC has vLLM / Ollama / local models
- vLLM on `:8000`: add a `hosted_vllm/...` entry with `api_base: http://localhost:8000/v1` to `ULTRA_FREE_TOOLS`.
- Ollama on `:11434`: add `ollama_chat/<model>` entries with `api_base: http://localhost:11434` to local groups.
- Slow PC / no GPU: use the small local models (`qwen2.5:3b`, `llama3.2:3b`, `qwen2.5:7b`) in `ULTRA_FREE_TOOLS_FAST` with `keep_alive: 24h`. Or rely purely on cloud free groups (`ULTRA_FREE_CLOUD*`) which need no local hardware.
- If no local models at all: delete/ignore the `ULTRA_FREE_TOOLS*` and `ULTRA_FREE_LOCAL*` entries; the cloud free groups still work. Local groups fail closed (no silent billing).

### Where to put keys and format
- File: `~/dbpasses.txt` (outside the repo, `chmod 600`).
- Format used by `tools/start_litellm_proxy.sh` `parse_key()`: a label line (e.g. `GROQ FREE KEY:`) followed on the next non-empty line by the raw value. Values must NOT be:
  - a pure uppercase identifier (e.g. `GROQ_SUPER`),
  - ending in `:`,
  - a URL,
  - a `====` section header.
  Example:
  ```
  ====FREE_APIS====
  GROQ FREE KEY:
  gsk_abc123...
  OPEN ROUTER API KEY
  sk-or-v1-xyz...
  ====PAID_APIS====
  ANTHROPIC
  sk-ant-...
  ```
- `start_litellm_proxy.sh` exports each into the env var the config references (`os.environ/GROQ_API_KEY`, etc.) then launches LiteLLM.

### Where to acquire keys (signup links)

Free:
- OpenRouter: https://openrouter.ai/signup → keys https://openrouter.ai/settings/keys
- Google AI Studio (Gemini): https://aistudio.google.com/ → keys https://aistudio.google.com/app/apikeys
- Groq: https://console.groq.com/ → keys https://console.groq.com/keys
- Cerebras: https://cloud.cerebras.ai/ → keys https://cloud.cerebras.ai/platform/user/api-keys
- DeepSeek: https://platform.deepseek.com/ → keys https://platform.deepseek.com/api_keys
- xAI / Grok: https://console.x.ai/ → keys https://console.x.ai/team/[TEAM]/api-keys
- Inception: https://platform.inception.ai/ → Settings → API Keys
- HuggingFace: https://huggingface.co/ → tokens https://huggingface.co/settings/tokens
- GitHub Models: https://github.com/marketplace/models → tokens https://github.com/settings/tokens
- Together: https://api.together.xyz/ → API keys page
- Mistral: https://console.mistral.ai/ → API keys
- Fireworks: https://fireworks.ai/ → API keys
- NVIDIA NIM: https://build.nvidia.com/ → API key
- Nous Research: https://inference-api.nousresearch.com/ → API key
- DeepInfra: https://deepinfra.com/ → API keys
- AIMLAPI: https://aimlapi.com/ → API keys
- Hypereal: https://api.hypereal.cloud/ → API key
- Cloudflare Workers AI: https://workers-ai.cloudflare.com/ → account ID + API token
- Ollama: local run `ollama serve` — no key needed for local models; Ollama Cloud uses SSH JWT (not supported in this config)
- vLLM: self-hosted — no key needed (set `--api-key` if desired, match `api_key: local` in config)

Paid:
- Anthropic: https://console.anthropic.com/ → API keys
- Moonshot: https://platform.moonshot.ai/ → API keys
- OpenAI: https://platform.openai.com/ → API keys

### Key leak prevention (mandatory)
- `~/dbpasses.txt` — `chmod 600`, NEVER commit, outside repo.
- `.env` — gitignored, never commit.
- `litellm_config.yaml` — references `os.environ/NAME`, never raw keys.
- Before any `git push`: run `git diff --cached` and grep for `sk-`, `csk-`, `gsk-`, `AIza`, `ghp_` to confirm no keys staged.
- Rotate any key that previously appeared in a tracked file (the Cerebras key that was in `.env.example` history must be considered exposed).
