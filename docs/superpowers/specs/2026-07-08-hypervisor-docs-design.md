# Hypervisor Documentation — Design

**Date:** 2026-07-08
**Repo:** Homelab (`github.com/Olwethu-Dlamini/Homelab`)

## Goal

Expand the Homelab repo with clear, vendor-neutral documentation about
hypervisors, landing on Proxmox VE as the concrete worked example. The work is
delivered as a series of small, logically-scoped commits, each pushed to
`origin/main`, so it doubles as a hands-on lesson in the git
`add → commit → push` workflow.

## Scope

**In scope**
- A root `README.md` that indexes the repository.
- A `docs/hypervisors/` tree of four Markdown documents.
- Cross-linking the existing `home-server-guide.html` to the new docs.
- Reverting the noise-only line-ending change currently in `LICENSE`.

**Out of scope (YAGNI)**
- Deploy scripts, cloud-init, or real config artifacts.
- Restructuring/moving the existing HTML guide.
- Alternatives beyond a short comparison (ESXi, XCP-ng, KVM/libvirt).

## Structure

```
README.md                     ← new: indexes the repo
LICENSE                       ← revert line-ending noise
home-server-guide.html        ← add links into docs/
docs/
  hypervisors/
    01-what-is-a-hypervisor.md        Type 1 vs Type 2, why virtualize
    02-how-virtualization-works.md    VT-x/AMD-V, KVM, IOMMU/passthrough
    03-choosing-a-platform.md         ESXi / XCP-ng / KVM-libvirt / Proxmox
    04-proxmox-in-practice.md         VMs vs LXC, storage, networking, backups
```

## Document contents

1. **01-what-is-a-hypervisor.md** — Definition; Type 1 (bare-metal) vs Type 2
   (hosted); why homelabbers virtualize (consolidation, isolation, snapshots).
2. **02-how-virtualization-works.md** — Hardware assist (Intel VT-x / AMD-V),
   KVM as the Linux kernel hypervisor, IOMMU and PCIe passthrough, the
   host/guest boundary.
3. **03-choosing-a-platform.md** — Landscape table: VMware ESXi, XCP-ng,
   plain KVM/libvirt, Proxmox VE. Trade-offs and why we use Proxmox as the
   example (open-source, LXC + KVM, web UI, strong homelab community).
4. **04-proxmox-in-practice.md** — Concrete Proxmox: VMs vs LXC containers,
   storage options, bridge networking, backups and snapshots.

## Commit & delivery plan

One logical change per commit; **push after each commit** so the full loop is
repeated and learned.

1. Revert the `LICENSE` line-ending change.
2. Add `README.md` + `docs/hypervisors/` skeleton.
3. Add `01-what-is-a-hypervisor.md`.
4. Add `02-how-virtualization-works.md`.
5. Add `03-choosing-a-platform.md`.
6. Add `04-proxmox-in-practice.md`.
7. Cross-link `home-server-guide.html` to the new docs.

For each commit, the exact `git` commands are shown and explained before running.

## Success criteria

- Four Markdown docs render cleanly on GitHub.
- `README.md` links to every doc and the HTML guide.
- Seven focused commits, each with a clear message, all pushed to `origin/main`.
- The reader understands hypervisors generically, then can act on Proxmox.
