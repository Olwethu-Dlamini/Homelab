# Homelab

Notes and guides for turning an old PC into a capable home server — from
networking fundamentals through running your own services.

**🌐 Live site:** https://olwethu-dlamini.github.io/Homelab/

## Guides

- **[Home Server Guide (v2)](home-server-guide-v2.html)** — the full walkthrough:
  foundations, building the base, deploying apps, and running it like a pro.
  Open the file in a browser.
- *(v1: [`home-server-guide.html`](home-server-guide.html) — kept for history.)*

## Documentation

### [Hypervisors](docs/hypervisors/) — virtualization from first principles

Vendor-neutral concepts first, then Proxmox VE as the concrete example.

1. [What is a hypervisor?](docs/hypervisors/01-what-is-a-hypervisor.md)
2. [How virtualization works](docs/hypervisors/02-how-virtualization-works.md)
3. [Choosing a platform](docs/hypervisors/03-choosing-a-platform.md)
4. [Proxmox in practice](docs/hypervisors/04-proxmox-in-practice.md)

### [Local AI](docs/local-ai/) — language models on Proxmox, without a GPU

Running models on the `ai` VM's CPU: what limits speed, the VM settings that
matter, the best small models as of November 2026, and n8n/RAG with them.

1. [Can a CPU run AI?](docs/local-ai/01-can-a-cpu-run-ai.md)
2. [Sizing a model](docs/local-ai/02-sizing-a-model.md)
3. [The ai VM in Proxmox](docs/local-ai/03-the-ai-vm-in-proxmox.md)
4. [The best models for this box](docs/local-ai/04-the-best-models.md)
5. [Ollama on the stack](docs/local-ai/05-ollama-on-the-stack.md)
6. [A chat window: Open WebUI](docs/local-ai/06-chat-ui-open-webui.md)
7. [n8n and RAG with local models](docs/local-ai/07-n8n-and-rag.md)
8. [Benchmarking and troubleshooting](docs/local-ai/08-benchmark-and-troubleshoot.md)

### Write-ups

- **[Giving n8n a public doorbell](n8n-public-webhooks.md)** — making n8n's
  webhooks reachable from the internet through srv1's Cloudflare Tunnel and
  Caddy, while the editor stays on Tailscale. Mistakes included.

## License

Released into the public domain — see [LICENSE](LICENSE).
