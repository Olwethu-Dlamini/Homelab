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

### Write-ups

- **[Giving n8n a public doorbell](n8n-public-webhooks.md)** — making n8n's
  webhooks reachable from the internet through srv1's Cloudflare Tunnel and
  Caddy, while the editor stays on Tailscale. Mistakes included.

## License

Released into the public domain — see [LICENSE](LICENSE).
