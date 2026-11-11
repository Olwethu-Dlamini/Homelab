# Sizing a model

Before downloading anything, two questions: **will it fit in RAM**, and **how
fast will it write**? Both answers come from a little arithmetic.

## Parameters

A model's size is its number of **parameters** (weights), in billions: `4b`,
`9b`, `27b`. More parameters usually means a better model, a bigger file and
slower writing.

Newer models of the same size keep getting better. A 2026 4B model beats the
2024 Mistral 7B on most tests, so "biggest that fits" is no longer the rule.
"Newest small model that does the job" is.

## Quantization: bits per weight

Weights are trained as 16-bit numbers. **Quantization** stores them with fewer
bits, trading a little quality for a much smaller file:

| Format | Bits per weight (about) | Size of a 7B model | Quality |
|---|---|---|---|
| F16 / BF16 | 16 | ~14GB | Original |
| Q8_0 | 8.5 | ~7.5GB | Almost identical |
| **Q4_K_M** | ~4.8 | **~4.4GB** | Small loss; the usual default |
| Q3 / Q2 | 3 / 2 | ~3.5GB / ~2.7GB | Noticeably worse on normal models |

Rule of thumb for Q4_K_M: **GB ≈ billions of parameters × 0.6**. Mistral 7B
(7.2B) → 4.4GB, which is what Ollama downloaded.

Two exceptions worth knowing:

- **QAT** tags (quantization-aware training, e.g. `gemma4:e4b-it-qat`): the
  model was trained to be quantized, so 4-bit loses less.
- **Ternary** models (e.g. Bonsai 2) are *trained* with weights of −1, 0 or +1.
  They are about half the size of a 4-bit model and are meant to keep close to
  full quality. Interesting for a RAM-starved box; see
  [chapter 4](04-the-best-models.md).

### Reading an Ollama tag

`qwen3.5:4b` is shorthand for a full tag like `qwen3.5:4b-q4_K_M`. The tags
page of each model lists every variant with its download size. On this box:

- Use the **default** tag (Q4_K_M) or a `qat` tag.
- Ignore `mlx` tags (Apple Silicon only) and `nvfp4` / `mxfp8` tags (number
  formats designed for recent GPUs).
- Use `q8_0` only for very small models where you have RAM to spare.

## Memory: weights + context + overhead

The RAM a loaded model needs is roughly:

```
RAM ≈ model file + KV cache (grows with context) + ~0.5GB runtime overhead
```

### Context and the KV cache

The **context window** is how many tokens the model can see at once: your
prompt, any documents you paste in, the conversation so far, and its answer.
For every token in the context, the model keeps a **KV cache** entry so it
doesn't recompute it. For Mistral 7B:

```
per token = 2 (K and V) × 32 layers × 8 KV heads × 128 dims × 2 bytes (f16)
          = 128KB
4,096 tokens  → 0.5GB
32,768 tokens → 4GB
```

Three things follow:

1. **Ollama's default context is 4,096 tokens.** Anything longer is cut off,
   which quietly breaks RAG prompts that stuff in many documents. Raise it with
   `OLLAMA_CONTEXT_LENGTH` (or `num_ctx` per request), and pay for it in RAM.
2. **A 256K context on the model card is a maximum, not a requirement.** You
   choose how much to allocate.
3. **The cache can be quantized.** `OLLAMA_KV_CACHE_TYPE=q8_0` halves it with
   very little quality loss ([chapter 5](05-ollama-on-the-stack.md)).

`OLLAMA_NUM_PARALLEL` multiplies the context: 2 parallel requests at 4K cost an
8K cache. The stack keeps it at 1.

## Dense vs mixture of experts (MoE)

- A **dense** model uses every weight for every token.
- An **MoE** model has many "expert" blocks and uses only a few per token. Its
  name often says both numbers: `24b-a2b` means 24B in total, about 2B
  **active**.

On a CPU this matters a lot. RAM must hold **all** the weights, but writing
speed depends on the **active** weights only. An 8B-A1B model needs the RAM of
an 8B model but writes about as fast as a 1B one. MoE is the best deal a CPU
box can get, as long as the whole model fits.

The big, popular MoE models of 2026 (30B to 35B total, 3B active) are
excellent on CPUs but need 18 to 25GB of RAM. They don't fit here; more RAM
would unlock them ([chapter 4](04-the-best-models.md#the-upgrade-that-would-change-the-list)).

## Thinking models

Many 2026 models can **think** first: they write a hidden chain of reasoning,
then the answer. It improves hard answers, but thinking tokens are written at
the same 3 to 6 tokens/s as everything else. Three hundred tokens of thinking
is one to two minutes before the first word of the answer.

For classification, extraction and short replies, turn thinking off
([chapter 5](05-ollama-on-the-stack.md#thinking-on-or-off)). Keep it for the
rare question that really needs it.

## Estimating speed before you download

From [chapter 1](01-can-a-cpu-run-ai.md), this VM delivered about **13GB/s**
to Mistral while writing, and read prompts at about 16 tokens/s for a 7.2B
model. Scale from there:

```
writing tokens/s ≈ 13 ÷ (GB read per token)       GB read per token ≈ active-weight file size
reading tokens/s ≈ 115 ÷ (billions of active parameters)
```

| Model file (Q4) | Active params | Writes (est.) | Reads (est.) |
|---|---|---|---|
| 6.6GB dense 9B | 9B | ~2 tok/s | ~13 tok/s |
| 4.4GB dense 7B (Mistral, measured) | 7.2B | **3 tok/s** | **16 tok/s** |
| 3.3GB dense 4B | 4B | ~4 tok/s | ~29 tok/s |
| 2.2GB dense 3B | 3B | ~6 tok/s | ~38 tok/s |
| 1.1GB dense 1.7B | 1.7B | ~12 tok/s | ~68 tok/s |
| 5.2GB MoE 8B-A1B | ~1B | ~10+ tok/s | ~100 tok/s |

Everything except the Mistral row is an **estimate** from that one
measurement. Two cautions:

- Files of multimodal models include a vision part that isn't read per text
  token, so they write a little faster than their file size suggests.
- If [chapter 3](03-the-ai-vm-in-proxmox.md) recovers more of the RAM's
  bandwidth, every row speeds up by the same factor.

Measure, don't trust: [chapter 8](08-benchmark-and-troubleshoot.md) has the
commands.

## Checklist: will it fit?

The stack caps Ollama at 6GB (`OLLAMA_MEM`, chapter 5 explains raising it).

```
model file + KV cache at your context + 0.5GB  <  OLLAMA_MEM
```

| Model file | KV cache, 4K context (rough) | Total | Fits 6GB? | Fits 8GB? |
|---|---|---|---|---|
| 1.1GB | ~0.2GB | ~1.8GB | yes | yes |
| 2.2GB | ~0.3GB | ~3GB | yes | yes |
| 3.3GB | ~0.4GB | ~4.2GB | yes | yes |
| 5.2GB | ~0.3GB | ~6GB | barely | yes |
| 6.6GB | ~0.5GB | ~7.6GB | **no** | yes, at 4K |

The KV numbers vary by architecture; Ollama prints the real figure in its log
when it loads a model, and `ollama ps` shows the total.
