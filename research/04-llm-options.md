# LLM Options for Journaling & Narration

## Summary

For a voice-journal app combining summarization, conversational follow-ups, and creative narrative rewriting, a tiered model strategy works best: a fast, cheap model (Claude Haiku 4.5, Gemini 2.5 Flash, or GPT-5.4 Mini/Nano) for live conversational turns and structured logging, and a stronger writer (Claude Opus 4.7, GPT-5/5.4, or Gemini 2.5 Pro) for end-of-day "epic story" narration. Claude Opus 4.7 currently leads creative-writing benchmarks (EQ-Bench Creative Writing Elo 2216 as of April 2026), while Haiku 4.5 hits ~80-120 tok/s with sub-second time-to-first-token, ideal for voice. For maximum privacy, a local Qwen 3 or Phi-4 model handles diary-style summarization well on consumer hardware, at the cost of weaker creative narration.

## Capabilities by Model

- **Claude Opus 4.7** — Best-in-class narrative voice, "show, don't tell" prose, and emotional nuance; 1M-token context. Top of EQ-Bench Creative Writing leaderboard ([BenchLM](https://benchlm.ai/blog/posts/best-llm-writing), [BasedAGI](https://basedagi.org/reports/best-llms-for-creative-writing)).
- **Claude Sonnet 4.6** — Strong all-around writer (EQ Elo 1991), 1M context, good balance of quality and cost ([finout.io](https://www.finout.io/blog/anthropic-api-pricing)).
- **Claude Haiku 4.5** — Fastest of the Claude tier (~78-120 tok/s, ~600-740 ms TTFT); great for voice-chat conversational turns and quick summaries ([Artificial Analysis](https://artificialanalysis.ai/models/claude-4-5-haiku/providers), [Skywork](https://skywork.ai/blog/claude-haiku-4-5-definition-speed-cost-use-cases/)).
- **GPT-5 / GPT-5.4** — Strong reasoning and creative writing (EQ Elo ~2024 for GPT-5.5); GPT-5.4 Mini/Nano are very fast (153 tok/s, 600 ms TTFT) ([pricepertoken](https://pricepertoken.com/pricing-page/model/openai-gpt-5)).
- **GPT-4.1** — Reliable, well-priced workhorse with mature Structured Outputs (99.9%+ schema compliance) ([devtk.ai](https://devtk.ai/en/blog/openai-api-pricing-guide-2026/)).
- **Gemini 2.5 Pro** — 1M+ context, strong multimodal (audio in/out), competitive reasoning ([Google AI](https://ai.google.dev/gemini-api/docs/pricing)).
- **Gemini 2.5 Flash** — Cheapest hosted "good enough" tier; lowest token overhead for structured outputs ([TLDL](https://www.tldl.io/resources/google-gemini-api-pricing)).
- **Llama 4 Scout / Maverick** — Open-weights MoE; Scout has a 10M context window, Maverick is the smarter flagship; both available via hosted APIs at very low rates ([Llama.com](https://www.llama.com/models/llama-4/), [TokenCost](https://tokencost.app/blog/llama-4-scout-vs-maverick-api-pricing)).
- **Qwen 3 (local)** — Toggleable thinking/non-thinking mode; strong privacy story (best PHI protection in medical benchmarks) ([Local AI Master](https://localaimaster.com/blog/small-language-models-guide-2026), [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S3050577125000180)).
- **Phi-4 (local)** — Compact, summarization-strong, runs alongside a browser; weaker on long creative prose ([DataCamp](https://www.datacamp.com/blog/top-small-language-models)).

## Pricing & Latency

| Model | Input $/M | Output $/M | Latency notes |
|---|---|---|---|
| Claude Opus 4.7 | $5 | $25 | Slowest of Claude tier |
| Claude Sonnet 4.6 | $3 | $15 | Mid-tier speed |
| Claude Haiku 4.5 | $1 | $5 | ~600 ms TTFT, 80-120 tok/s |
| GPT-5 | $1.25 | $10 | ~1.21 s TTFT, 64 tok/s |
| GPT-5.4 Mini | ~$0.30 | ~$2.40 | 0.60 s TTFT, 153 tok/s |
| GPT-5.4 Nano | $0.20 | $1.25 | Fastest OpenAI tier |
| GPT-4.1 | $2 | $8 | Mature, predictable |
| Gemini 2.5 Pro | $1.25 (≤200K) / $2.50 | $10 / $15 | 1M context |
| Gemini 2.5 Flash | $0.30 | $2.50 | Fast, cheap |
| Llama 4 Scout (hosted) | ~$0.08-0.15 | similar | 10M context |
| Llama 4 Maverick (hosted) | ~$0.19-0.49 | similar | Flagship quality |
| Qwen 3 / Phi-4 (local) | $0 | $0 | Hardware-bound |

Prompt caching (Claude, Gemini) and batch tier (50% off, OpenAI/Anthropic) further reduce cost for daily-summary workloads ([CloudZero](https://www.cloudzero.com/blog/claude-opus-4-7-pricing/)).

## Trade-offs

- **Creative quality vs. cost**: Opus/GPT-5 produce richer "epic story" narration but cost 5-25x Haiku/Flash for the same word count.
- **Structured output**: GPT-5/4.1 and Gemini have native constrained-decoding JSON; Claude uses tool-use with input_schema (no progressive streaming of fields) ([Medium](https://medium.com/@rosgluk/structured-output-comparison-across-popular-llm-providers-openai-gemini-anthropic-mistral-and-1a5d42fa612a), [TokenMix](https://tokenmix.ai/blog/structured-output-json-guide)).
- **Latency for voice**: Haiku 4.5 and GPT-5.4 Mini/Nano are the only hosted options that feel real-time; local Phi-4/Qwen 3 are competitive on Apple Silicon or recent NPUs.
- **Privacy**: Journals are sensitive. Hosted APIs all offer no-training agreements, but only on-device Qwen/Phi/Llama keep data fully local ([Sigma AI](https://sigma.ai/llm-privacy-security-phi-pii-best-practices/), [Private LLM](https://privatellm.app/en)).
- **Context length**: Llama 4 Scout (10M) and Claude/Gemini (1M) easily fit a year of journal entries; smaller local models cap at 32-128K.

## Recommendations for Voice Journal

1. **Conversational capture loop**: Claude Haiku 4.5 or GPT-5.4 Mini for sub-second turn-taking, structured-output tool calls to extract entities (people, mood, events) into a daily log.
2. **Daily summary**: Sonnet 4.6 or Gemini 2.5 Flash — cheap enough to run nightly, strong enough for coherent recaps.
3. **"Epic story" narration**: Claude Opus 4.7 (top creative ranking) on demand, or GPT-5.4 if already in OpenAI stack. Run weekly or on user request to control cost.
4. **Privacy mode**: Bundle Qwen 3 (8B) via Ollama/MLX for users who opt out of cloud; accept reduced narrative flair.
5. **Cost guardrails**: Use prompt caching for the user's running journal context, and batch async narration jobs for 50% discount.

Sources: see linked references throughout.
