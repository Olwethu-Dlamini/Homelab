# Hypervisors

Understanding virtualization from first principles, then applying it with
Proxmox VE as the worked example.

Read in order:

1. **[What is a hypervisor?](01-what-is-a-hypervisor.md)**
   Type 1 vs Type 2, and why you'd virtualize at all.
2. **[How virtualization works](02-how-virtualization-works.md)**
   The hardware and kernel machinery: VT-x/AMD-V, KVM, IOMMU, passthrough.
3. **[Choosing a platform](03-choosing-a-platform.md)**
   ESXi, XCP-ng, KVM/libvirt, Proxmox — trade-offs and why we pick Proxmox.
4. **[Proxmox in practice](04-proxmox-in-practice.md)**
   VMs vs LXC, storage, networking, backups and snapshots.

These docs are platform-agnostic where possible so the ideas transfer even if
you later switch hypervisors.
