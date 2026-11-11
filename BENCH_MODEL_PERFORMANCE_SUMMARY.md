
# Model Performance Summary

This document summarizes the performance of various language models, categorized by their operational environment and cost. The metrics considered are capabilities, tokens per second (TPS), and context window size.

## Our Current Models - LOCALLY RUN (FREE)

These models are hosted and run on local infrastructure, offering free usage without direct token costs, though they consume local compute resources.

| Model Name | Capabilities | Advertised Context Window | Tested Input Context | Measured Mean TPS (Short Call) | Notes |
|---|---|---|---|---|---|
| **Qwen2.5-14B** | Text generation | 8,192 tokens (total) | N/A | N/A (Not benchmarked in provided data) | Runs on local vLLM (port 8000). Specific performance metrics not detailed in the provided data. |

## Our Current Models - FREE (CLOUD)

These models are accessible via cloud endpoints at no direct cost, leveraging free tiers or promotional offers.

| Model Name | Capabilities | Advertised Context Window | Tested Input Context | Measured Mean TPS (Short Call) | Notes |
|---|---|---|---|---|---|
| **NVIDIA Nemotron 3.5 Lightning** | Text → text (designed for faster agent workloads) | 1,000,000 tokens | ~779,000 tokens | **52** | Part of `free-HighTPS_Context` alias. Rotates with Super-VL. Failed JSON formatting twice at high context. |
| **NVIDIA Nemotron 3.5 Super-VL** | Text, images, and video → text | 1,000,000 tokens | ~779,000 tokens | **20** | Part of `free-HighTPS_Context` alias. Rotates with Lightning. Passed JSON formatting at high context. |

**Free Cloud Model Comparison:**
- **Nemotron 3.5 Lightning** is approximately **2.6x faster** than Super-VL in short-call TPS, while offering the same advertised context window.
- Both free cloud models retained facts at approximately **779K input tokens**, well within their advertised 1M token context window.
- The `free-HighTPS_Context` alias allows for **900,000 input + 8,192 output** tokens, with a **240-second timeout**.

## Our Current Models - PAID (CLOUD)

These models are accessed via cloud endpoints and incur costs based on usage (tokens, calls, etc.).

| Model Name | Capabilities | Advertised Context Window | Tested Input Context | Measured Mean TPS (Short Call) | Notes |
|---|---|---|---|---|---|
| **Mercury-3** | Text → text | 1,000,000 tokens | N/A | **548** | Significantly faster than free cloud models. |
| **Grok 4.7** | Text → text | 500,000 tokens | N/A | **99** | Offers half the context window of Mercury-3 and free Nemotron models. |
| **DeepSeek V4 Pro** | Text → text | 1,048,576 tokens | N/A | **62** | Offers a large context window, comparable to Kimi. |
| **Kimi K3.1 Pro** | Text → text | 1,048,576 tokens | N/A | **34** | Offers a large context window, comparable to DeepSeek. |

**Paid Cloud Model Comparison:**
- **Mercury-3** is the fastest model benchmarked, offering ~10.6x the TPS of Lightning and ~5.5x that of Super-VL, with a similar context window.
- **Grok 4.7** is ~1.9x faster than Lightning but has a significantly smaller context window (500K vs 1M tokens).
- **Kimi K3.1 Pro** and **DeepSeek V4 Pro** offer the largest context windows (1,048,576 tokens) but have lower TPS compared to Mercury-3 and Grok. Kimi and DeepSeek's TPS are comparable to the free Nemotron models, despite their larger context.
- The TPS figures for Kimi and DeepSeek are noted as potentially less useful for final answer throughput due to consuming output budgets on reasoning without a final answer in the specific test calls.

**Performance Efficiency Considerations:**
- **Locally Run (FREE):** Qwen2.5-14B is available locally, but its performance metrics (TPS, context) were not provided in the benchmark data. Its efficiency would depend on local hardware.
- **Free Cloud:** Nemotron 3.5 Lightning offers a good balance of speed and large context for a free tier model, though Super-VL is slower.
- **Paid Cloud:** Mercury-3 stands out for raw speed. For large context needs, Kimi and DeepSeek are options, though their TPS is lower. Grok offers a mid-range context window with better speed than Kimi/DeepSeek.

## Rate Limits and Concurrency

Rate limits are critical for maintaining stability and avoiding upstream throttling.

| Model Group | Rate Limit (RPM) | Concurrency/Timeout | Notes |
|---|---|---|---|
| **Free Cloud (Nemotron)** | N/A (Provider-enforced) | 240s Timeout | No paid fallback; quota exhaustion fails request. |
| **Paid Cloud (Moonshot)** | 5,000 RPM | N/A | High throughput; 3M TPM. |
| **Free Mode (Gemini)** | 1-15 RPM (varies) | 65s Cooldown | Enforced per deployment; uses multiple API keys. |

**Further Research/Considerations:**
- The provided data does not include information on other paid cloud providers or locally run models compatible with GX10.
- The performance of Kimi and DeepSeek in the provided benchmarks may not fully represent their capabilities for generating complete answers due to their output behavior in the specific test calls.
- Sustained performance and cost-effectiveness for specific workloads would require more extensive testing beyond the short-call benchmarks provided.
