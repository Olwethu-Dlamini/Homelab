# An M.2 SSD for the host

The single change that would make models load in seconds instead of a minute.
It does nothing for how fast a model writes: that is set by the CPU and RAM
([chapter 1](01-can-a-cpu-run-ai.md)).

## What's in the box today

Checked on `pve` on 13 November 2026, read-only:

| Part | What it is |
|---|---|
| Machine | HP ProDesk 600 G4 SFF, Q370 chipset, i5-8500 |
| Disk | one Seagate ST500DM002, 500GB, 7,200 rpm, on SATA. It holds Proxmox and every VM |
| M.2 storage slot | **one, PCIe 3.0 x4 (NVMe), 2230 or 2280** ([HP spec sheet](https://jp.ext.hp.com/content/dam/jp-ext-hp-com/jp/ja/ec/lib/products/desktops/spec_pdf/600g4sf_bto.pdf)). Empty: `lspci` shows no NVMe controller |

To check it yourself:

```bash
# on pve
cat /sys/class/dmi/id/product_name          # HP ProDesk 600 G4 SFF
lsblk -d -o NAME,SIZE,ROTA,TRAN,MODEL       # ROTA 1 = spinning disk
lspci | grep -iE 'nvme|non-volatile'        # nothing = the M.2 slot is empty
```

## What it would change

| Storage | Sequential read | Mistral 7B (4.4GB) cold load | `qwen3.5:2b` (1.3GB) |
|---|---|---|---|
| The HDD, measured 13 Nov 2026 | ~70MB/s (after `GOMAXPROCS=1`) | 63.5s | ~20s |
| SATA SSD (2.5") | ~550MB/s | ~10s | ~3s |
| **M.2 NVMe, PCIe 3.0 x4** | ~3,000 to 3,500MB/s raw, perhaps 1,500 to 2,500MB/s inside the VM | **~5 to 10s** | **~2 to 4s** |

Only the HDD row is measured. The SSD rows are worked out from typical drive
speeds; a load also includes a few seconds of Ollama's own setup, so it never
falls to the raw disk time.

What stays the same: writing (2.8 to 3.1 tok/s for Mistral) and reading the
prompt (~18.8 tok/s). Those come from the CPU and memory, not the disk
([chapter 8](08-benchmark-and-troubleshoot.md#results)).

What else gets faster: everything on the host. VM boots, updates, backups and
the website VM all share the one HDD today.

## Which drive

- **M.2 2280, NVMe, PCIe 3.0 x4.** A PCIe 4.0 drive works too, but this slot
  runs it at 3.0 speed, so don't pay extra for one.
- **A drive with DRAM cache or at least HMB**, from a known brand. Proxmox and
  several VMs write constantly; the cheapest DRAM-less drives slow down badly
  under that.
- **Size:** 500GB holds Proxmox, both VMs and a handful of models. 1TB leaves
  room to keep the HDD for backups only.

## Fitting it

1. Shut the host down from Proxmox (Datacenter → `pve` → Shutdown), unplug it.
2. Open the side panel; the M.2 slot is on the motherboard. Fit the drive and
   its screw (2280 = the long position).
3. Plug in a monitor for the first boot and watch it reach the Proxmox login
   prompt before walking away (see [the RAM upgrade write-up](../../ram-upgrade-outage.md)).
4. Check Linux sees it:

   ```bash
   # on pve
   lsblk -d -o NAME,SIZE,ROTA,TRAN,MODEL      # a new nvme0n1, ROTA 0
   ```

## Putting it to use

Two ways, smallest first:

- **Move only `ai`'s disk.** In Proxmox: Datacenter → `pve` → Disks → LVM-Thin
  → Create: Thinpool on `nvme0n1`. Then VM 101 → Hardware → Hard Disk → Disk
  Action → Move Storage → the new pool. It moves while the VM runs. Models,
  n8n and Qdrant all land on the SSD.
- **Move everything** (Proxmox itself and both VMs): a reinstall onto the SSD
  and a restore from backups. More work; only worth it if the HDD is failing.

After the move, measure a cold load the way
[chapter 8](08-benchmark-and-troubleshoot.md#measuring-the-comparable-way)
describes (empty the page cache first) and add the row to its results table.

## Before you buy: is the HDD failing?

The HDD loaded Mistral at about 90MB/s in October and about 20MB/s on
13 November, before the `GOMAXPROCS` fix
([chapter 3](03-the-ai-vm-in-proxmox.md#4-the-disk-why-the-first-answer-is-slow)).
If it is dying, the SSD stops being an upgrade and becomes the replacement, and
a backup comes first:

```bash
# on pve, read-only
smartctl -a /dev/sda | grep -E 'Reallocated_Sector_Ct|Current_Pending_Sector|Offline_Uncorrectable'
```

Any non-zero raw value on those three means plan the move now.
