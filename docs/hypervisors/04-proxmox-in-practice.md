# Proxmox in practice

The concrete payoff. This page maps the concepts from the earlier docs onto
Proxmox VE's actual features: VMs vs containers, storage, networking, and
backups/snapshots. It's an orientation, not a click-by-click install — for the
full guided install see the [home server guide](../../home-server-guide-v2.html).

## VMs vs LXC containers

Proxmox runs two kinds of guest, and choosing well matters.

### KVM virtual machines (QEMU)
Full virtual machines with their own kernel — exactly the
[Type 1 virtualization](01-what-is-a-hypervisor.md) described earlier.

- Run **any** OS: Linux, Windows, BSD, appliances.
- Strongest isolation; required for PCIe/GPU passthrough.
- Heavier: each VM boots a full OS and reserves its own RAM.
- Use for: Windows, firewalls (pfSense/OPNsense), NAS (TrueNAS), anything
  non-Linux, or anything needing passthrough.

**Tips:** use the **VirtIO SCSI** controller and **VirtIO** network model for
speed (see [paravirtualization](02-how-virtualization-works.md)); on Windows,
mount the virtio driver ISO during install. Install the **QEMU guest agent** in
the VM so Proxmox can cleanly shut it down and report its IP.

### LXC containers
Lightweight Linux containers that share the host kernel — the
[container model](01-what-is-a-hypervisor.md#hypervisors-vs-containers).

- Linux only, and they use the host's kernel.
- Very light: start in seconds, tiny RAM/disk footprint, dense.
- Can be **privileged** or **unprivileged** — prefer **unprivileged** for
  safety (guest root ≠ host root).
- Use for: lightweight Linux services where a full VM is overkill.

> **Rule of thumb:** reach for an **LXC** for a simple Linux service; reach for a
> **VM** when you need a different OS, hard isolation, or passthrough. Many
> people also run **one VM with Docker inside** for app containers, keeping
> Docker off the hypervisor host itself.

## Storage

Proxmox separates *what* you store from *where*. Common backends:

- **ZFS** — checksumming, snapshots, compression, and software RAID in one.
  Popular for reliability; wants a fair bit of RAM and prefers whole disks (not
  hardware RAID underneath it).
- **LVM / LVM-thin** — simple, efficient local volumes; thin provisioning gives
  fast VM snapshots.
- **Directory** — a plain filesystem path; simplest, good for ISOs, templates,
  and backups.
- **Ceph** — distributed storage across a cluster; powerful but overkill for a
  single node.

Content types a storage can hold: VM **disk images**, **container volumes**,
**ISO images**, **CT templates**, and **backups**. You assign which types live
where under *Datacenter → Storage*.

## Networking

By default Proxmox creates a **Linux bridge** called `vmbr0` — a virtual switch.
Your physical NIC joins the bridge, and each guest gets a virtual NIC on it, so
guests appear as normal devices on your LAN and pull IPs from your router.

Building blocks:

- **Bridge (`vmbrX`)** — virtual switch guests plug into. `vmbr0` usually maps
  to your real NIC.
- **VLANs** — tag traffic to segment networks (e.g. isolate IoT or a DMZ) on one
  physical link.
- **NAT / internal bridges** — a bridge with no physical NIC, for
  host-only/internal networks behind the host.
- **Passthrough NIC** — hand a physical card straight to a firewall VM via
  [IOMMU](02-how-virtualization-works.md#iommu-and-pcie-passthrough).

Give the Proxmox host itself a **static IP** — you don't want the management
address moving.

## Backups and snapshots

Two different tools people often confuse:

- **Snapshot** — a point-in-time capture of a VM/CT's state (optionally
  including RAM) on the *same* storage. Instant, great for "before I change
  something" rollbacks. **Not** a backup — if the storage dies, so does the
  snapshot. Needs a storage type that supports it (ZFS, LVM-thin, qcow2).
- **Backup (`vzdump`)** — a full, self-contained archive of a guest written
  elsewhere (another disk, NAS, or **Proxmox Backup Server**). Survives host
  failure. Schedule these under *Datacenter → Backup*.

Best practice: **snapshots for quick rollbacks, scheduled backups for real
safety**, and keep at least one backup copy off the host. Test a restore
occasionally — an untested backup is a hope, not a backup.

## A sensible first-node layout

A common, low-drama starting point:

1. Install Proxmox to a small SSD; give the host a **static IP**.
2. Add a data disk/pool (ZFS or LVM-thin) for VM/CT storage.
3. Create **one Ubuntu Server VM** with VirtIO disk/NIC + guest agent, and run
   **Docker** inside it for your apps.
4. Add **LXC** containers for lightweight Linux services as needed.
5. Configure **scheduled backups** to a separate disk or NAS from day one.
6. Snapshot before every risky change.

From here, the [home server guide](../../home-server-guide-v2.html) takes over
with the hands-on install, Docker, reverse proxy, DNS, and hardening steps.

---

← Prev: [Choosing a platform](03-choosing-a-platform.md) ·
Back to [Hypervisors index](README.md)
