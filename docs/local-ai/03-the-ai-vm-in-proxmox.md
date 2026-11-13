# The ai VM in Proxmox

Model choice is half the speed. The other half is how Proxmox presents the
hardware to the VM. This page goes through the settings that matter for
inference, in the order worth checking: cheapest and most likely first.

## Where things stand

VM 101, `ai`, was created on 1 October 2026 with **4 cores, 12GB RAM,
ballooning off, a 300GB VirtIO SCSI disk and the guest agent on**. It runs
Debian 13 and the Docker Compose stack in `/opt/automation`. Inside it, Docker
gives Ollama at most **3 CPUs and 6GB** (`OLLAMA_CPUS`, `OLLAMA_MEM` in the
stack's `.env`).

Confirm the current settings on the host before changing anything:

```bash
# on pve
qm list
qm config 101 | grep -E '^(name|cores|sockets|cpu|memory|balloon|numa|scsi0):'
lscpu | grep -E 'Model name|^CPU\(s\)'
```

An 8th-gen desktop i5 (i5-8400, 8500, 8600) has **6 cores and 6 threads**: no
hyper-threading. `lscpu` tells you which one this is.

## 1. CPU type: does the VM see AVX2?

This is the most likely reason the VM got only a third of the RAM's bandwidth
in [chapter 1](01-can-a-cpu-run-ai.md).

Ollama's CPU code (from llama.cpp) has fast paths for the **AVX2** and **FMA**
instructions, which every 8th-gen i5 has. But a VM only sees the instructions
its **CPU type** exposes. Recent Proxmox versions default new VMs to
`x86-64-v2-AES`, a safe baseline that **does not include AVX2** (that arrives
in `x86-64-v3`). Without it, inference falls back to slower code paths.

The [Proxmox in practice](../hypervisors/04-proxmox-in-practice.md) doc and the
home server guide both recommend CPU type `host`. Check whether VM 101 got it:

```bash
# inside ai
grep -o -w -E 'avx2|fma|f16c' /proc/cpuinfo | sort -u
```

Three lines (`avx2`, `f16c`, `fma`) means the VM sees them. Nothing means it
doesn't. Ollama also logs what it detected when it loads a model:

```bash
sudo docker logs ollama 2>&1 | grep -i -E 'avx2|cpu.*features' | tail -3
```

### Fixing it

```bash
# on pve
qm set 101 --cpu host
qm shutdown 101 && qm start 101
```

The VM must be **powered off and on again**. A reboot from inside the guest
keeps the same QEMU process, and with it the old CPU type.

This stops n8n, WAHA and the public webhooks for about a minute. Do it when no
workflow is due.

`host` ties the VM to this exact CPU, which only matters when moving VMs
between different machines. With one host, there's no downside.

## 2. Cores and the Docker CPU cap

Reading the prompt is pure arithmetic, so it scales with cores. Writing is
mostly limited by RAM, but one or two threads can't saturate the memory bus
either. Today Ollama is held to 3 of the VM's 4 cores.

Do this in two steps and measure after each
([chapter 8](08-benchmark-and-troubleshoot.md)):

1. **Let Ollama use all 4 VM cores.** In `/opt/automation/.env` set
   `OLLAMA_CPUS=4`, then redeploy. No VM restart. n8n, WAHA and Qdrant are
   mostly idle and keep their own caps.
2. **Give the VM all 6 host cores.** vCPUs are just threads to the host, so
   giving out more vCPUs than physical cores in total (overcommitting) is
   normal. srv1 stays responsive if you give it a bigger share when both want
   the CPU:

   ```bash
   # on pve: srv1's VM id from `qm list`
   qm set 101 --cores 6
   qm set <srv1-id> --cpuunits 200    # default weight is 100; 200 wins ties 2:1
   qm shutdown 101 && qm start 101
   ```

   Then `OLLAMA_CPUS=5` in `.env`, leaving one core's worth for everything
   else in the VM.

If step 2 doesn't beat step 1 by a clear margin, go back to 4 cores. More
threads than the memory bus can feed only adds heat.

Leave **NUMA off**: this is a single-socket machine.

## 3. Memory: channels, ballooning and the budget

### Is the RAM running in dual channel?

Bandwidth doubles with two sticks in the right slots. On the host:

```bash
dmidecode -t memory | grep -E 'Locator|Size|Configured Memory Speed' | grep -v 'No Module'
```

You want two 8GB sticks on **different channels** (labels like `ChannelA` and
`ChannelB`, or `DIMM A1` and `DIMM B1`), both at the same configured speed. Two
sticks in the same channel, or mismatched speeds, cost bandwidth, and on a CPU
bandwidth *is* writing speed.

### Ballooning stays off

Ballooning lets Proxmox take unused RAM back from a running VM. That's wrong
for a VM whose job is to load a 4GB model on demand: the RAM must be there the
moment a request arrives. VM 101 already has it off; keep it that way:

```bash
qm config 101 | grep balloon     # expect: balloon: 0
```

### The budget

The host has 16GB: Proxmox itself (keep ~2GB free), srv1, and `ai` with 12GB.
Inside `ai`, every service has a hard cap (a Docker `mem_limit` for the
containers, a systemd `MemoryMax` for mail-poll, which runs on a timer, not in
Docker):

| Service | Cap | Measured (Oct 2026) |
|---|---|---|
| Ollama | 6GB | 6.0GB peak with Mistral loaded; 13MB unloaded |
| n8n | 1.5GB | 0.4 to 0.8GB |
| Qdrant | 1.5GB | under 0.1GB, no collections yet |
| WAHA | 768MB | ~350MB of its own |
| bridge (container) + mail-poll (systemd) | 128MB + 64MB | ~25MB |
| **Total caps** | **~10GB** | **1.2 to 1.8GB used with no model loaded** |

There is room to raise Ollama's cap to **8GB** if you want a 9B model
([chapter 4](04-the-best-models.md)). The caps would then add up to the VM's
12GB, which is fine because Qdrant and n8n run far below theirs. Watch for
swap after the change: the VM's swap is on the spinning disk, and a model that
spills into swap writes at a token per minute.

```bash
# inside ai
free -m
cat /proc/pressure/memory      # "some avg60" should stay near 0.00
```

## 4. The disk: why the first answer is slow

Models live in the `ollama_models` Docker volume, on the VM's disk, on the
host's single 500GB HDD. Loading Mistral took **48.1s** from cold, about
90MB/s, which is what a spinning disk does.

- After a load, Linux keeps the file in the VM's page cache while RAM allows,
  so the next load is usually much faster. The VM reported ~10GB of
  reclaimable cache when idle, so a small model tends to stay cached.
- **Smaller models load faster**: a 2.2GB model needs about half Mistral's
  time from cold.
- `OLLAMA_KEEP_ALIVE=2m` frees RAM quickly but means a workflow that runs every
  10 minutes pays the load every time. [Chapter 5](05-ollama-on-the-stack.md)
  covers the trade-off.

**13 November 2026: the same load took 213.3s**, about 20MB/s, four times
slower than in October. Writing speed was unchanged (3.1 tok/s), so the CPU
and RAM are fine and the disk is not. The cause isn't known yet. The two
likely ones, and how to tell them apart:

- **Something else using the HDD at the same time** (the other VM, a backup,
  a scrub). Watch the host while a model loads: `iostat -x 2` on `pve` shows
  the disk near 100% busy, and which VM is reading.
- **The disk is failing.** A dying HDD retries slow sectors, and reads crawl.
  `smartctl -a /dev/sdX` on `pve`: non-zero `Reallocated_Sector_Ct`,
  `Current_Pending_Sector` or `Offline_Uncorrectable` mean replace it soon,
  and back up first.

An SSD remains the single best hardware upgrade for this box: it would cut cold
loads to a few seconds and help every VM.

## 5. Things that don't help here

- **The integrated GPU (Intel UHD 630).** It uses the same system RAM, so it
  can't write faster than the CPU's memory bus allows, and passing an iGPU into
  a VM is fiddly. Not worth it on this machine.
- **Moving Ollama to an LXC container.** An LXC would save the guest kernel's
  memory and share the host's page cache, but `ai` is built, documented and
  working. The gain is a few hundred MB.
- **More swap.** Swap on an HDD is a safety net against crashes, not extra
  space for models.

## Summary of commands

```bash
# --- on pve ---
qm config 101 | grep -E '^(cores|cpu|memory|balloon):'
qm set 101 --cpu host                   # AVX2 for the guest
qm set 101 --cores 6                    # optional, step 2
qm set <srv1-id> --cpuunits 200         # optional, keep srv1 snappy
qm shutdown 101 && qm start 101         # needed for cpu/cores changes
dmidecode -t memory | grep -E 'Locator|Size|Configured Memory Speed' | grep -v 'No Module'

# --- inside ai ---
grep -o -w -E 'avx2|fma|f16c' /proc/cpuinfo | sort -u
free -m; cat /proc/pressure/memory
```
