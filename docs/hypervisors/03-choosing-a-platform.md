# Choosing a platform

Once you've decided to run a Type 1 hypervisor, you have to pick one. Here are
the four you're most likely to weigh for a homelab, what they're good at, and
why the rest of these docs use **Proxmox VE** as the example.

## The landscape

| Platform | Type | Engine | Cost / License | Containers | Best for |
|---|---|---|---|---|---|
| **Proxmox VE** | 1 | KVM + QEMU, LXC | Open source (AGPL); paid support/enterprise repo | LXC built-in | Homelabs, small business, all-rounder |
| **VMware ESXi** | 1 | VMkernel | Proprietary; free tier withdrawn/limited | No (VMs only) | Enterprise shops, VMware skills |
| **XCP-ng** | 1 | Xen | Open source; optional paid support | No (VMs only) | Xen fans, Citrix-style setups |
| **KVM + libvirt** | 1 | KVM + QEMU | Open source (part of Linux) | Via separate tools | Full control, scripting, minimalists |
| *Hyper-V* | 1 | Hyper-V | Windows feature | No (VMs only) | Windows-centric environments |

## The options in brief

### Proxmox VE
A Debian-based distribution that bundles KVM (for VMs) **and** LXC (for
containers) behind a clean web UI, with clustering, backups, and storage
(ZFS, Ceph, LVM) built in. Open source; a subscription buys the stable
enterprise repo and support, but the free "no-subscription" repo works fine.
The pragmatic default for most homelabs.

### VMware ESXi
The long-time enterprise standard: rock-solid, huge ecosystem, polished
tooling. Downsides for a homelab: it's proprietary, the free edition has been
restricted/withdrawn following Broadcom's acquisition, and hardware
compatibility is pickier. Worth it mainly if you're building VMware skills for
work.

### XCP-ng
An open-source Xen-based platform (a community continuation of XenServer),
managed with **Xen Orchestra**. Solid and enterprise-flavoured. Smaller
community than Proxmox and no built-in containers, but a strong choice if you
prefer the Xen hypervisor.

### KVM + libvirt (plain Linux)
Skip the appliance and run KVM directly on your favourite Linux distro, managed
with `virsh`/`virt-manager` or Cockpit. Maximum control and nothing you didn't
choose — but you assemble backups, a UI, storage, and networking yourself.
Great if you want to understand every layer or automate everything.

## Why these docs use Proxmox

For a homelab specifically, Proxmox hits the best balance:

- **Open source and free** to run, with no artificial feature gating.
- **VMs *and* containers** in one tool — KVM for full VMs, LXC for lightweight
  Linux workloads (see [Proxmox in practice](04-proxmox-in-practice.md)).
- **A real web UI** plus a scriptable API and CLI — approachable now, automatable
  later.
- **Batteries included** — ZFS, snapshots, scheduled backups, and clustering
  without bolting on extra projects.
- **Huge community** — enormous amount of homelab-specific guides and help.

None of this means Proxmox is "correct" and the others are wrong. Because these
docs teach the underlying concepts (all four are ultimately Type 1 hypervisors
over the same hardware features), what you learn transfers if you switch.

## A quick decision guide

- **Just want it to work, one box, VMs + containers** → Proxmox VE.
- **Building VMware skills for a job** → ESXi.
- **Prefer Xen / want Xen Orchestra** → XCP-ng.
- **Want total control and to script everything** → KVM + libvirt.

---

← Prev: [How virtualization works](02-how-virtualization-works.md) ·
Next: **[Proxmox in practice](04-proxmox-in-practice.md)** →
