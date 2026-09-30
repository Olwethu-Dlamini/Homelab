# The RAM Upgrade, Part Two: 16GB and Working

Date: 30 September 2026

This is the follow-up to [The Day I Added a RAM Stick and "Lost" My Server](ram-upgrade-outage.md). That one covered the first boot after the upgrade, when the host never came up and I spent a while convinced it was a network problem. This one closes the upgrade out: the Proxmox host now has **16GB of RAM**, up from 8GB, and it's running.

---

## Part 0: Where it stands

- The host (`pve`) shows **16GB** of memory
- `srv1` is back, and so is the site at `22112002.xyz`
- The upgrade I listed under "Planned next" in the homelab guide is now done

---

## Part 1: Why 16GB in the first place

The original plan was two separate VMs:

- one for **hosting websites** (what `srv1` does now)
- one for **automation / AI-agent tooling** such as n8n

On 8GB that didn't fit. The Proxmox host needs its own share before any guest starts, and each VM runs a full operating system that takes a slice before doing anything useful. Two proper VMs on 8GB would have meant squeezing them both, or giving up on the separation and using containers as a compromise.

Keeping those two roles apart matters to me. A website other people visit shouldn't share a blast radius with internal automation that has access to my own accounts. 16GB makes that separation affordable.

---

## Part 2: How to confirm the RAM is really there

Three places to check, from quickest to most detailed.

### The Proxmox web UI

**Datacenter → pve → Summary**. The **RAM usage** line shows used / total. The total will read a little under 16, usually 15.x. That's normal (see Part 5).

### On the host

```bash
free -h
```

The `total` column on the `Mem:` row is the number that matters.

### Per stick, per slot

```bash
dmidecode -t memory | grep -E "Locator|Size|Speed|Part Number" | grep -v "No Module"
```

This lists each slot, what's in it, and the speed it's actually running at. It's the quickest way to spot a stick that isn't detected, or two sticks running at different speeds.

---

## Part 3: What the extra 8GB is for

The next step is the **second VM** for automation. A starting budget, to adjust once I see real usage:

| Where | Rough share | Why |
|---|---|---|
| Proxmox host | ~2GB | The hypervisor itself, plus headroom so the host never swaps |
| `srv1` (websites) | ~4GB | Debian minimal, the web server, the Cloudflare tunnel, the sites |
| Second VM (automation) | ~4–6GB | n8n and whatever workflows grow around it |
| Spare | the rest | For a container or a test VM without shutting anything down |

To change a VM's memory later (in MB; the VM needs a restart for it to take effect):

```bash
qm set <vmid> --memory 4096
```

One thing 16GB still doesn't change: **there's no GPU**, so the automation VM orchestrates and calls a hosted model API (such as OpenRouter) for the AI part. It doesn't run the models itself.

---

## Part 4: Carried over from the outage report

These were on the "set up so this is less painful next time" list, and they're still worth doing now that the upgrade is finished:

- **VM autostart**: `qm set <vmid> --onboot 1` so `srv1` (and the site) come back whenever the host boots
- **BIOS: Restore on AC Power Loss → Power On**, for real power cuts
- **An uptime monitor** (e.g. UptimeRobot on `22112002.xyz`) so I get an email instead of finding out myself
- **A known_hosts entry for the Tailscale name**: `ssh root@pve` still fails with `Host key verification failed`, so quick checks over Tailscale don't work yet. Run `ssh root@pve` once by hand and accept the key.
- **The work VPN still claims `10.0.0.x`.** `ip route get 10.0.0.50` showed `dev wg0` again today. Either drop `wg0` while testing, or narrow the VPN's `AllowedIPs` so it stops covering my home subnet.

---

## Part 5: What I learned

1. **"16GB" shows as a little under 16.** The firmware and the integrated graphics reserve some memory before the OS starts, so the total reads 15.x. If it's near 15.5, all the RAM is there. If it's near 8, one stick isn't being detected.
2. **Check the numbers after a hardware change, don't assume.** `free -h` and `dmidecode` take seconds and confirm the upgrade actually took.
3. **More RAM doesn't fix the disk.** Everything still runs on one spinning 500GB HDD. If this grows into something other people rely on, an SSD is the upgrade that matters most now.

---

## Part 6: Quick reference

```bash
# Total RAM the host sees
free -h

# Each stick: slot, size, speed
dmidecode -t memory | grep -E "Locator|Size|Speed|Part Number" | grep -v "No Module"

# VMs and their memory
qm list
qm config <vmid> | grep -E "^(name|memory|balloon|onboot):"

# Change a VM's memory (MB) / make it start on boot
qm set <vmid> --memory 4096
qm set <vmid> --onboot 1

# Is the work VPN hijacking the LAN?  ("dev wg0" = yes)
ip route get 10.0.0.50
```

---

## Glossary additions from today

- **Reserved memory**: RAM the firmware and integrated graphics set aside at boot. The OS never sees it, which is why a 16GB machine reports slightly less.
- **Memory ballooning**: a Proxmox feature that lets the host take unused RAM back from a VM while it runs. Set with `--balloon`. It's useful once there are two VMs competing for memory.
- **Swap**: disk space used as overflow when RAM runs out. On a spinning HDD it's very slow, which is why the host keeps headroom instead of relying on it.
