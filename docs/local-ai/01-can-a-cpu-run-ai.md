# Can a CPU run AI?

Yes, slowly. This page explains why it is slow, which part of the machine sets
the speed, and which jobs are still worth giving a CPU-only box.

## What the earlier guides got wrong

The homelab guide, the journey write-up and the RAM upgrade report all said
something like: *running LLM inference locally needs a GPU I don't have, so the
box orchestrates and a hosted API does the thinking.*

The architecture advice was sound, but "needs a GPU" was too strong. Since
October 2026 the `ai` VM has run Mistral 7B on the CPU alone, inside the n8n
stack. It works. What a GPU buys is **speed**, not the ability to run at all.

## How a model writes text

A language model produces one **token** (roughly ¾ of a word) at a time. Each
token goes through two different kinds of work:

| Phase | What happens | What limits it on a CPU |
|---|---|---|
| **Reading the prompt** (prefill) | The model processes every token you sent, many at once | **Compute**: cores, clock speed, AVX2 |
| **Writing the answer** (decode) | For each new token, the model reads **all of its weights** from RAM once | **Memory bandwidth**: how many GB/s the RAM can deliver |

The second row is the surprise. To write one token, a 4.4GB model streams
4.4GB through the CPU. To write ten tokens a second it needs 44GB/s. More cores
barely help once the memory bus is full; a GPU is fast mostly because its
memory is 10 to 30 times faster than desktop DDR4.

## What this box actually measured

From the `ai` VM on 7 October 2026 (recorded in the stack's
`docs/SYSTEM-DESIGN.md`), Mistral 7B at Q4_K_M, about 4.4GB:

| Measurement | Result |
|---|---|
| Writing speed | 56 tokens in 18.4s, **about 3 tokens/s** |
| Reading speed | 41 tokens in 2.5s, **about 16 tokens/s** |
| Cold load (model not in RAM) | **48.1s** |

Turn the writing speed into bandwidth: 4.4GB × 3 tokens/s ≈ **13GB/s**.

An 8th-gen i5 with two DDR4-2666 sticks in dual channel has a theoretical peak
of 2 channels × 8 bytes × 2666 million transfers/s ≈ **42.7GB/s**. Real
programs reach maybe 70% of that. So the VM was getting roughly a third of what
the RAM can deliver. [Chapter 3](03-the-ai-vm-in-proxmox.md) lists the likely
reasons, and the first one (the VM's CPU type) costs nothing to check.

## Turning speeds into waiting time

A request costs roughly:

```
time ≈ load time (if cold) + prompt tokens ÷ reading speed + answer tokens ÷ writing speed
```

With Mistral's numbers:

| Job | Prompt | Answer | Waiting time |
|---|---|---|---|
| Classify a short message ("payment / question / other") | 200 | 5 | ~14s |
| Extract four fields from an email as JSON | 600 | 60 | ~58s |
| Answer from a knowledge base (RAG) | 1,500 | 200 | ~2.7 min |
| Same, but the model was unloaded | 1,500 | 200 | ~3.5 min |

That is why the stack's design notes already say the payment bot uses rules,
not AI: rules answer in milliseconds, Mistral took minutes.

## What a CPU is good for

- **Background jobs nobody watches**: tag, classify, route, summarise, pull
  fields out of text into JSON. Thirty seconds is fine when no human waits.
- **Short prompts with short answers.** Writing speed hurts most on long
  answers; reading speed hurts most on long prompts.
- **Embeddings** for search and RAG. Embedding models are tiny (a few hundred
  MB) and only do the fast "reading" phase.
- **Private chat** where you accept waiting, and where the text must not leave
  the house.

## What it is bad at

- Live chat with customers who expect an answer in seconds.
- Long documents in the prompt (every 1,000 tokens costs a minute or more of
  reading on a 7B model).
- Agent loops that call the model five or ten times per task.
- Coding assistants. Those want big models and long contexts: use a hosted
  model or a real GPU.

## The plan for the rest of this series

The measured numbers point at two levers:

1. **Make the VM faster at what it already runs**: CPU type, cores, Docker
   limits ([chapter 3](03-the-ai-vm-in-proxmox.md)).
2. **Run smaller, newer models.** Mistral 7B is from 2024. A 2026 model half
   its size scores far higher on tests and writes about twice as fast, because
   there are half as many bytes to stream per token
   ([chapters 2](02-sizing-a-model.md) and [4](04-the-best-models.md)).
