# How virtualization works

A hypervisor has to convince each guest OS that it owns a real computer, while
actually sharing one physical machine among many guests — safely and with as
little overhead as possible. This page covers the machinery that makes that
possible.

## The core problem: rings and privilege

A CPU runs code at different **privilege levels** (on x86, "rings"). A normal OS
kernel expects to run at the most privileged level (ring 0) so it can touch
hardware directly. But if several guest kernels all think they own ring 0,
they'd collide.

Early virtualization solved this in software (trapping and rewriting privileged
instructions), which was clever but slow. Modern virtualization solves it in
**hardware**.

## Hardware assist: Intel VT-x and AMD-V

Modern CPUs include virtualization extensions:

- **Intel VT-x** (and **VT-d** for I/O — see IOMMU below)
- **AMD-V** (and **AMD-Vi** for I/O)

These add a new CPU mode so guests can run their kernels at full speed in a
"guest" context, while the hypervisor stays in control from a separate "host"
context. When a guest does something that needs supervision, the CPU **traps**
back to the hypervisor (a *VM exit*), which handles it and resumes the guest.

> **You must enable this in the BIOS/UEFI.** It's often off by default and
> labelled *Intel VT-x / Intel Virtualization Technology*, *SVM Mode* (AMD), or
> similar. Without it, Type 1 hypervisors won't install and VMs run painfully
> slowly (or not at all).

To check on an existing Linux host:

```bash
# Non-empty output means the CPU flag is present
grep -oE 'vmx|svm' /proc/cpuinfo | sort -u

# vmx = Intel VT-x,  svm = AMD-V
```

## KVM: the Linux kernel as a hypervisor

**KVM (Kernel-based Virtual Machine)** is a Linux kernel module that turns the
kernel into a Type 1 hypervisor by driving VT-x/AMD-V directly. Linux becomes
the thin, privileged layer that schedules VMs like it schedules processes.

KVM handles CPU and memory virtualization; a companion component
(commonly **QEMU**) emulates the rest of a machine — virtual disks, network
cards, BIOS/UEFI. Management tools (**libvirt**, and platforms like **Proxmox**)
sit on top to give you an API or UI.

```
Proxmox / libvirt / virt-manager     ← management
        QEMU                          ← device emulation (disk, NIC, ...)
        KVM (kernel module)           ← CPU + memory virtualization
        VT-x / AMD-V (CPU)            ← hardware assist
```

This is why "Linux with KVM," Proxmox, and oVirt all share the same engine
underneath — they're different front-ends over KVM+QEMU.

## Memory virtualization

Each guest has its own idea of physical memory, but there's only one real RAM.
The CPU provides **Second Level Address Translation** (Intel *EPT*, AMD *RVI/NPT*)
so the hardware translates guest addresses to host addresses automatically,
without the hypervisor intervening on every memory access. That's what makes
guest memory nearly as fast as native.

## I/O: paravirtualization vs passthrough

Emulating a real network card or disk controller in software is accurate but
slow. Two techniques speed I/O up:

### Paravirtualized drivers (virtio)

Instead of pretending to be a real Intel NIC, the hypervisor exposes an
efficient **virtio** device, and the guest loads a matching `virtio` driver.
The guest *knows* it's virtualized and cooperates, cutting overhead
dramatically. Use virtio disks and NICs whenever the guest supports them
(Linux does out of the box; Windows needs the virtio driver ISO).

### IOMMU and PCIe passthrough

Sometimes a guest needs a *real* device — a GPU, an HBA, a network card — with
full native performance. The **IOMMU** (Intel **VT-d**, AMD **AMD-Vi**) makes
this safe by remapping and isolating the device's memory access so a guest can
drive hardware directly without being able to stomp on host or other-guest
memory.

**PCIe passthrough** assigns a physical device to a single VM. Common uses:

- A GPU passed to a VM for gaming, transcoding, or ML.
- A disk controller (HBA) passed to a NAS VM (e.g. TrueNAS) so it sees the raw
  drives.
- A NIC passed to a firewall/router VM (e.g. pfSense/OPNsense).

Requirements: VT-d/AMD-Vi enabled in BIOS, IOMMU enabled in the hypervisor, and
the device sitting in a clean **IOMMU group**. Setup is fiddlier than a normal
VM — it's the advanced end of homelab virtualization.

## Overcommitment

Because guests rarely use all their resources at once, hypervisors let you
**overcommit** — allocate more virtual CPUs or RAM than physically exist, and
let the scheduler share the real hardware. Handy, but overdo it and everything
slows down (or, for RAM without swap/ballooning, VMs get killed). Size for your
real peak load.

---

← Prev: [What is a hypervisor?](01-what-is-a-hypervisor.md) ·
Next: **[Choosing a platform](03-choosing-a-platform.md)** →
