# What is a hypervisor?

A **hypervisor** is software that lets one physical computer run many
independent operating systems at the same time. Each of those operating
systems runs inside a **virtual machine (VM)** — a software-defined computer
with its own virtual CPU, memory, disks, and network cards.

The hypervisor's job is to sit between the real hardware and those VMs, hand
each VM a slice of the machine, and keep them isolated from one another. The
physical machine is the **host**; each VM running on it is a **guest**.

```
        ┌───────────┐  ┌───────────┐  ┌───────────┐
        │  Guest A  │  │  Guest B  │  │  Guest C  │   ← virtual machines
        │ (Ubuntu)  │  │ (Windows) │  │ (pfSense) │
        └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
              └────────┬─────┴──────────────┘
                 ┌─────┴──────┐
                 │ Hypervisor │                        ← divides + isolates
                 └─────┬──────┘
                 ┌─────┴──────┐
                 │  Hardware  │                        ← one physical PC
                 └────────────┘
```

## Type 1 vs Type 2

Hypervisors come in two flavours, distinguished by what sits underneath them.

### Type 1 — "bare-metal"

The hypervisor **is** the base operating system. It installs directly onto the
hardware, and everything else runs as a guest on top of it.

- Examples: **Proxmox VE**, VMware ESXi, XCP-ng, Microsoft Hyper-V (Server).
- Lower overhead, better performance and isolation.
- The right choice for an always-on server or homelab.

```
[ VM ] [ VM ] [ VM ]
└──────┬───────────┘
   Hypervisor
   Hardware
```

### Type 2 — "hosted"

The hypervisor is an **application** running on top of a normal desktop OS
(Windows, macOS, Linux). The host OS boots first; the hypervisor is just
another program.

- Examples: VirtualBox, VMware Workstation/Player, Parallels.
- More overhead (guests go through the host OS), but trivial to install.
- Great for testing on your laptop; not ideal for a dedicated server.

```
[ VM ] [ VM ]
└───┬──────┘
 Hypervisor (app)
 Host OS (Windows/macOS/Linux)
 Hardware
```

> The line can blur. Linux's **KVM** turns the kernel itself into a Type 1-style
> hypervisor, even though Linux is a general-purpose OS. More on that in
> [How virtualization works](02-how-virtualization-works.md).

## Why virtualize?

Running VMs on one box buys you a lot:

- **Consolidation** — one capable machine replaces several under-used ones.
  Your firewall, media server, and dev box share the same hardware.
- **Isolation** — a crash, misconfiguration, or compromise in one VM doesn't
  take down the others. Each guest is walled off.
- **Snapshots** — freeze a VM's exact state before a risky change and roll back
  in seconds if it breaks.
- **Portability** — a VM is just files. Back it up, clone it, or move it to
  another host.
- **Efficient use of resources** — allocate CPU/RAM per workload and reclaim it
  when idle, instead of one job hogging a whole machine.

## Hypervisors vs containers

VMs are not the only way to isolate workloads. **Containers** (Docker, LXC)
share the host's kernel and isolate at the process level, so they're lighter
and start faster — but they can't run a different OS kernel.

- **VM:** full virtual hardware + its own kernel. Strong isolation, heavier.
- **Container:** shared kernel, isolated userspace. Light, fast, less isolated.

They're complementary. A common homelab pattern is a Type 1 hypervisor running
a few VMs, with containers running *inside* one of those VMs. Proxmox happens to
do both — KVM VMs and LXC containers side by side (see
[Proxmox in practice](04-proxmox-in-practice.md)).

---

Next: **[How virtualization works](02-how-virtualization-works.md)** →
