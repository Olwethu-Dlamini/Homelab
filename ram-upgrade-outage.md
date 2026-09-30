# The Day I Added a RAM Stick and "Lost" My Server

Date: 30 September 2026

This covers one short outage: I powered the Proxmox host down to add a RAM stick, powered it back on, and then couldn't find it anywhere. No Tailscale, no LAN, no website. Same format as the other guides: what happened, how I checked, what fixed it, and what I'm taking away from it.

---

## Part 0: The starting symptom

After fitting the new RAM and powering the box back on, I couldn't reach anything:

- `pve` (the Proxmox host) and `srv1` (the VM on it) weren't reachable over Tailscale
- `ssh root@10.0.0.50` went nowhere
- The live site at `22112002.xyz` was down

---

## Part 1: Checking from the laptop first

### Tailscale status

```bash
tailscale status
```

```
100.107.46.32   pve    ...  linux  active; relay "jnb"; offline, last seen 1h ago, tx 15600 rx 0
100.72.72.58    srv1   ...  linux  offline, last seen 1h ago
```

**Both machines went offline at the same moment.** `srv1` is a VM that runs *on* `pve`, so when both vanish together, the problem is the host itself, not one service. `tx 15600 rx 0` means my laptop was sending packets and getting nothing back.

### The website

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://22112002.xyz
```

```
530
```

A Cloudflare **530** means Cloudflare is up but can't reach my origin. In other words, the tunnel on `srv1` isn't connected. That matches a host that's down.

### Gremlin 3 again: the work VPN was eating my LAN traffic

```bash
ip route get 10.0.0.50
```

```
10.0.0.50 dev wg0 src 10.8.0.6
```

My work WireGuard connection (`wg0`) was back up and claiming all of `10.0.0.0/24`, so every `ping 10.0.0.50` or `ssh root@10.0.0.50` went into the VPN instead of onto my home network. Worse, a ping to `10.0.0.10` actually came back "UP". That reply came from **something on the VPN's side**, not my `srv1`. A misleading success is more dangerous than a clean failure.

**The fix for testing** is to force traffic out of the Ethernet card, or to drop the VPN for a moment:

```bash
ping -c2 -I enp1s0 10.0.0.50       # force the real LAN
# or
sudo wg-quick down wg0              # ...test...  then: sudo wg-quick up wg0
```

### Sweeping the real LAN

```bash
for i in $(seq 1 254); do (ping -c1 -W1 -I enp1s0 10.0.0.$i >/dev/null 2>&1 && echo "10.0.0.$i") & done; wait
```

Only the router, the laptop, and a couple of other devices answered. **Nothing at `.50` or `.10`.** One new device (`10.0.0.21`) showed up while I was looking, but its MAC address was a randomised, "locally administered" one (`f2:...`), which is typical of a phone. It had no SSH or Proxmox port open either. So it wasn't the server on a new IP.

**Conclusion from the laptop side:** the host wasn't on the network at all. Even with its fans spinning, it hadn't booted into Proxmox.

---

## Part 2: The actual cause and the fix

The only thing that had changed was the **new RAM stick**. After a memory change, a PC can:

- sit on a black screen for minutes doing **memory training** (it's re-learning timings for the new configuration)
- stop at a BIOS prompt such as "memory size changed, press F1"
- fail to POST (the power-on self-test) at all: beeps, or a DRAM warning light on the board

In my case the first power-on after the upgrade never made it into Proxmox. **A restart fixed it.** On the second boot it came straight up.

---

## Part 3: Confirming it was really back

```bash
tailscale status | grep -E 'pve|srv1'
```

```
100.107.46.32   pve    ...  linux  active; relay "jnb", tx 258076 rx 1971740
100.72.72.58    srv1   ...  linux  -
```

```bash
ping -c1 -I enp1s0 10.0.0.50                                    # UP
ping -c1 -I enp1s0 10.0.0.10                                    # UP
curl -sk --interface enp1s0 -o /dev/null -w '%{http_code}\n' https://10.0.0.50:8006   # 200 (Proxmox UI)
curl -s -o /dev/null -w '%{http_code}\n' https://22112002.xyz                          # 200 (site live)
```

Host, VM, Proxmox web UI and the public site were all back. `rx` climbing from 0 to almost 2 MB was the clearest sign that traffic was flowing both ways again.

---

## Part 4: What I learned

1. **If `pve` and `srv1` drop together, look at the host.** The VM can't be up if the machine under it isn't.
2. **Cloudflare 530 means "origin unreachable".** It points at my server or tunnel, not at Cloudflare or DNS.
3. **Check `ip route get <ip>` before trusting a ping.** My work VPN hijacked `10.0.0.x` again, and it even produced a fake "UP". `ping -I enp1s0` takes the VPN out of the picture.
4. **"Fans on" doesn't mean "booted".** After any hardware change, plug in a monitor for the first boot and watch it get to the Proxmox login prompt before walking away.
5. **After adding RAM, give the first boot time**, up to about 5 minutes for memory training. If it still doesn't come up, restart it. If that fails too, reseat the stick, check the slot pairing in the motherboard manual, and try booting with only the old stick to isolate the new one.
6. **A randomised MAC (`x2:`, `x6:`, `xA:`, `xE:` as the second hex digit) is almost always a phone**, not a server that changed IP.

---

## Part 5: Worth setting up so this is less painful next time

- **VM autostart**: `qm set <vmid> --onboot 1` so `srv1` (and the site) come back on their own whenever the host boots
- **BIOS: Restore on AC Power Loss → Power On**, for real power cuts
- **An uptime monitor** (e.g. UptimeRobot on `22112002.xyz`) so I get an email instead of discovering the outage myself
- **A known_hosts entry for the Tailscale names** (`ssh root@pve` once, and accept the key) so quick checks over Tailscale work without falling back to the LAN IP

---

## Part 6: Quick reference

```bash
# 1. Is it the host or one service?
tailscale status

# 2. Is the site down because the origin is?  (530 = origin unreachable)
curl -s -o /dev/null -w '%{http_code}\n' https://22112002.xyz

# 3. Is the work VPN hijacking the LAN?
ip route get 10.0.0.50              # "dev wg0" = yes

# 4. Test the real LAN only
ping -c2 -I enp1s0 10.0.0.50
curl -sk --interface enp1s0 https://10.0.0.50:8006 -o /dev/null -w '%{http_code}\n'

# 5. Not on the LAN at all -> go to the machine with a monitor:
#    black screen = memory training (wait), F1 prompt = confirm RAM,
#    beeps / DRAM light = reseat or remove the new stick, then restart

# 6. Once it's back, confirm everything
tailscale status
curl -s -o /dev/null -w '%{http_code}\n' https://22112002.xyz   # 200
```

---

## Glossary additions from today

- **Memory training**: the first-boot routine where the motherboard tests and tunes timings for newly installed RAM. It can leave a black screen for several minutes and looks exactly like a dead machine.
- **POST (Power-On Self-Test)**: the firmware's hardware check before any OS loads. A machine that fails POST never reaches Proxmox, so it never appears on the network.
- **Cloudflare 530**: Cloudflare couldn't connect to the origin, typically because the tunnel (`cloudflared`) is down.
- **Locally administered / randomised MAC**: a MAC address whose second hex digit is 2, 6, A or E. Phones use these for privacy on Wi-Fi, so seeing one usually rules out a server.
