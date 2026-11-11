# Local AI on Proxmox, without a GPU

How to run language models on the homelab's own hardware: an old 8th-gen i5,
16GB of RAM, one spinning 500GB disk, and no graphics card. The worked example
is the `ai` VM (VM 101), which already runs n8n, Qdrant, WAHA and Ollama in one
Docker Compose stack under `/opt/automation`.

The earlier guides said local inference "needs a GPU I don't have". That was
too strong. A CPU runs small modern models fine; it just runs them slowly, and
the whole game is picking models that are small enough to be useful at CPU
speed. This series is about doing that well.

Read in order:

1. **[Can a CPU run AI?](01-can-a-cpu-run-ai.md)**
   What actually limits speed (memory bandwidth, not cores), what the first
   model measured, and which jobs suit a CPU.
2. **[Sizing a model](02-sizing-a-model.md)**
   Parameters, quantization, context and the KV cache, dense vs MoE, and the
   arithmetic for "will it fit, and how fast will it be?"
3. **[The ai VM in Proxmox](03-the-ai-vm-in-proxmox.md)**
   The VM settings that matter for inference: CPU type and AVX2, cores,
   memory and ballooning, the slow disk.
4. **[The best models for this box (November 2026)](04-the-best-models.md)**
   What to run instead of Mistral 7B, by job, with sizes and sources.
5. **[Ollama on the stack](05-ollama-on-the-stack.md)**
   The Compose settings, switching models, turning thinking off, the API.
6. **[A chat window: Open WebUI](06-chat-ui-open-webui.md)**
   A private ChatGPT-style page for the models, reached over Tailscale only.
7. **[n8n and RAG with local models](07-n8n-and-rag.md)**
   Chat, embeddings and Qdrant from n8n, and when to call a hosted model.
8. **[Benchmarking and troubleshooting](08-benchmark-and-troubleshoot.md)**
   Measuring tokens per second, a results table to fill in, and the usual
   failures.

The [hypervisor docs](../hypervisors/) explain the Proxmox ideas these pages
lean on (VM vs LXC, VirtIO, CPU type `host`).

Model picks age fast. Every size and score here was checked on
**11 November 2026**; check the linked pages before trusting them later.
