# The best models for this box (November 2026)

The `ai` VM runs Mistral 7B, a 2024 model. Two years is a long time here: 2026
models half its size score far higher and write faster. This page picks
replacements by job, for a CPU-only VM where Ollama gets 6GB (or 8GB, if you
raise the cap as described in [chapter 3](03-the-ai-vm-in-proxmox.md#the-budget)).

Checked on **11 November 2026**. Sizes are Ollama's download sizes for the
default tag. Scores come from [Tiny Analysis](https://tinyanalysis.com/), which
labels each score as **independent** (measured by Artificial Analysis) or
**lab-reported** (the maker's own claim). Prefer independent numbers; makers
choose the tests they look good on.

## Short answer

| Job | Model | Ollama tag | Download | Writes (est.) | Licence |
|---|---|---|---|---|---|
| **Default for everything** | Qwen3.5 4B | `qwen3.5:4b` | 3.3GB | ~4 tok/s | Apache 2.0 |
| **n8n workhorse**: extract, classify, JSON, tools | Granite 4.2 3B | `granite4.2:3b` | 2.2GB | ~6 tok/s | Apache 2.0 |
| **Fastest that's still good** | LFM3 1.7B | `lfm3:1.7b` | 1.1GB | ~12 tok/s | LFM Open License |
| **Best answers**, if you can wait (needs the 8GB cap) | Qwen3.5 9B | `qwen3.5:9b` | 6.6GB | ~2 tok/s | Apache 2.0 |
| **Embeddings** for RAG | Qwen3 Embedding 0.6B | `qwen3-embedding:0.6b` | 639MB | n/a | Apache 2.0 |

Speeds are estimates scaled from the measured Mistral run
([chapter 2](02-sizing-a-model.md#estimating-speed-before-you-download)); if
the CPU-type fix in chapter 3 lands, all of them rise. Measure your own with
[chapter 8](08-benchmark-and-troubleshoot.md).

For comparison, Mistral 7B: 4.4GB, **3 tok/s measured**.

## The picks, one by one

### Qwen3.5 4B: the new default

- **Why**: the strongest model under 4GB with independent scores behind it:
  GPQA Diamond **77.1** (independent), MMLU-Pro 79.1 (lab), IFBench 52.0
  (independent). Reads images as well as text, supports tool calling, and has
  a thinking mode you can switch off.
- **Fit**: roughly 4GB loaded at a 4K context. Comfortable in the 6GB cap.
- **Watch**: it thinks before answering unless told not to. On this CPU that
  can add a minute or more. Turn it off for routine jobs
  ([chapter 5](05-ollama-on-the-stack.md#thinking-on-or-off)).
- Pages: [Ollama](https://ollama.com/library/qwen3.5) ·
  [scores](https://tinyanalysis.com/models/qwen3.5-4b)

### Granite 4.2 3B: the n8n workhorse

- **Why**: built by IBM for exactly the boring jobs n8n gives a model:
  structured JSON output, tool use, RAG, instruction following. IFBench 74.3
  (lab), GPQA 55.9 (independent). Tiny Analysis notes it as the highest-scoring
  open model of 4B and under on the Artificial Analysis Intelligence Index v5.
- **Fit**: ~3GB with a 4K context. Small enough to stay loaded next to an
  embedding model.
- **Watch**: less general knowledge than Qwen3.5 4B. Give it clear
  instructions and a fixed output format, and it shines.
- Pages: [Ollama](https://ollama.com/library/granite4.2) ·
  [scores](https://tinyanalysis.com/models/granite-4.2-3b)

### LFM3 1.7B: when speed matters most

- **Why**: 1.1GB, so it writes roughly four times faster than Mistral and loads
  from the HDD in about 12 seconds. Liquid AI's new hybrid architecture,
  post-trained for agents and tool calling, with a 256K context.
- **Watch**: released on **27 October 2026**, two weeks before this was
  written, and its scores so far are the maker's own (IFBench 73.0, lab).
  Promising, not proven. Test it on your real prompts before relying on it.
- **Licence**: LFM Open License v1.0, not Apache. Read it before using the
  model for a business.
- Pages: [Ollama](https://ollama.com/library/lfm3) ·
  [scores](https://tinyanalysis.com/models/lfm3-1.7b)

### Qwen3.5 9B: the best answers that fit at all

- **Why**: #1 of 30 small models on Tiny Analysis's index. GPQA 80.6 and
  IFBench 66.7, both independent; MMLU-Pro 82.5 (lab).
- **Fit**: ~7.6GB at a 4K context. **Does not fit the current 6GB cap**; raise
  `OLLAMA_MEM` to `8g` first.
- **Watch**: about 2 tok/s. Use it for overnight or batch jobs where quality
  beats speed, not for anything interactive.
- Pages: [Ollama](https://ollama.com/library/qwen3.5) ·
  [scores](https://tinyanalysis.com/models/qwen3.5-9b)

### Qwen3 Embedding 0.6B: for search and RAG

- **Why**: small (639MB), 32K context, good multilingual retrieval, and the
  same family as the chat model. Embedding is the fast "reading" phase only, so
  it's quick even on this CPU.
- **Smaller option**: `embeddinggemma-2:270m` (~300MB), released in October
  2026.
- **Watch**: pick one embedding model and keep it. Vectors from different
  models don't mix; switching means re-embedding every document in Qdrant.
- Pages: [Ollama](https://ollama.com/library/qwen3-embedding) ·
  [EmbeddingGemma 2](https://ollama.com/library/embeddinggemma-2)

## Worth trying

| Model | Tag | Download | Why it's interesting | Why it's not a default |
|---|---|---|---|---|
| Granite 4.2 8B | `granite4.2:8b` | 5.3GB | #3 of 30 small models (index 62.7), Apache 2.0, answers without thinking | ~6.2GB at 4K context: needs the 8GB cap; ~2.5 tok/s |
| LFM2.5 8B-A1B | `lfm2.5:8b` | 5.2GB | MoE with ~1B active: 8B-class memory, near-1B speed | Fills the 6GB cap; scores mostly lab-reported |
| Bonsai 2 9B | `bonsai-2:9b` | 3.2GB | **Ternary** weights: a 9B model in the space of a 4B, Apache 2.0, vision and tools | CPU speed of its 2-bit kernels unknown here; benchmark it first |

## Considered and skipped

| Model | Why not, on this box |
|---|---|
| Gemma 4 E4B / E2B | Lower scores than Qwen3.5 at similar sizes (Small LM Index 40.9 and 37.4), and larger files (6.6GB / 4.6GB) |
| Nanbeige4.2 3B, Spark-X2.6 4B | Top of Tiny Analysis's 8GB list, but on lab-reported scores only and not in Ollama's library |
| MiMo-V2.6 9B (coding) | 6.6GB and ~2 tok/s. Coding wants a big model and long context: use a hosted model |
| gpt-oss 20B | 14GB |
| Qwen3.6 35B-A3B, Nemotron 3.5 Lightning 30B-A3B, Laguna XS 2.1 | Excellent on CPUs thanks to ~3B active, but 19 to 24GB |
| LFM2.5 24B-A2B, Granite 4.2 30B | 15GB and 18GB |
| Mistral 7B (current) | Two years older and weaker than Qwen3.5 4B, slower, and bigger |

## The upgrade that would change the list

The skipped MoE models (30B to 35B total, ~3B active) would write at about the
speed of a 3B model here: a big jump in quality at no cost in speed. They need
RAM, not a GPU. With **32GB** in the host, `ai` could get ~24GB and run them.

The CPU supports far more than 16GB; the motherboard's slot count and maximum
decide. Check its manual and `dmidecode -t memory` (which also shows how many
slots are free) before buying.

## Pulling them

```bash
# inside ai
sudo docker exec ollama ollama pull qwen3.5:4b
sudo docker exec ollama ollama pull granite4.2:3b
sudo docker exec ollama ollama pull qwen3-embedding:0.6b
sudo docker exec ollama ollama list
```

Models you stop using still take disk space: `ollama rm <tag>`.
[Chapter 5](05-ollama-on-the-stack.md) makes one of them the stack's default.

## Sources

- Ollama library, sizes and tags: [ollama.com/library](https://ollama.com/library)
- Scores and memory estimates: [Tiny Analysis small-model leaderboard](https://tinyanalysis.com/),
  [best LLMs for an 8GB machine](https://tinyanalysis.com/best/8gb-gpu)
- Independent measurements: [Artificial Analysis](https://artificialanalysis.ai/)
