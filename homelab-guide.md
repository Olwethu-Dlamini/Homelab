# My Homelab: Proxmox + Tailscale + Cloudflare Tunnel
### A step-by-step guide written for beginners (by a beginner)

This document explains what I built, why each piece exists, and how to rebuild it from scratch. If you're teaching someone else, walk them through the sections in order — each one builds on the last.

---

## 0. The Big Picture (read this first)

Before touching any commands, understand the shape of what we're building. There are **4 layers**, and confusing them is the #1 source of beginner mistakes:

| Layer | What it is | Example in this guide |
|---|---|---|
| **Physical machine** | The actual computer sitting in a room | An old desktop: i5 8th gen, 8GB RAM (upgraded to 16GB), 500GB HDD |
| **Hypervisor** | Software that turns 1 computer into a host for many virtual computers | Proxmox VE |
| **VM (Virtual Machine)** | A full fake computer running inside the hypervisor, with its own OS | `srv1` running Debian |
| **Services** | Programs running inside the VM | Cloudflare Tunnel, a website, later n8n |

**Why virtualize at all?** One physical machine can pretend to be many separate servers. Each VM thinks it's alone on its own hardware. This lets you isolate things (a website server vs an automation server) without buying multiple computers.

**Why does networking get complicated?** Because now there are multiple "layers" of network address:
- The physical machine has a LAN IP (e.g. `10.0.0.50`)
- Each VM has its own LAN IP (e.g. `10.0.0.10`)
- Your ISP gives your router one shared, non-public IP (CGNAT) — meaning **nobody on the internet can reach your home network directly**, which is the core problem the rest of this guide works around

---

## 1. Installing Proxmox (the hypervisor)

Proxmox is installed like a normal OS — boot from a USB, click through their installer.

### Key screen: Management Network
This is where you tell the machine its permanent identity on your home network.

- **Management interface**: pick the **ethernet** port, never wifi. Hypervisors need to "bridge" network traffic to VMs, and wifi hardware generally can't do this properly (most wifi routers reject traffic using a "foreign" device's identity, which is exactly what a VM needs to send).
- **Hostname (FQDN)**: a name for the machine, in `something.something.something` format. We used `pve.22112002.xyz`. This does **not** need to be a real, published domain record — it's just a label the machine uses to refer to itself.
- **IP Address (CIDR)**: a fixed ("static") address on your home network, e.g. `10.0.0.50/24`. Don't leave this on DHCP — servers need an address that never changes, or you'll lose track of where they are.
- **Gateway**: your router's address, e.g. `10.0.0.1`.
- **DNS**: `1.1.1.1` (Cloudflare's public DNS) — though see the DNS troubleshooting note in Section 6, some ISPs block this.

**How to find your network's numbers if you don't know them:** on any device already connected to your home network, run:
```bash
ip a      # shows your current IP
ip r      # shows your gateway (the "default via ..." line)
```
Keep the first three numbers the same as your other devices, change the last number to something unused (like `.50`).

### After install
```bash
# Remove the "enterprise" repos (they need a paid subscription you don't have)
rm /etc/apt/sources.list.d/pve-enterprise.list
rm /etc/apt/sources.list.d/ceph.list

# Add the free repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt full-upgrade -y
```

You now manage everything from a browser at `https://<your-static-ip>:8006` — this **is** the interface for the machine going forward. The physical keyboard/monitor on the box itself is now only for emergencies.

**Important mindset shift:** once Proxmox is installed, that PC is no longer "a computer you sit at." It has no desktop, no browser. It becomes an *appliance* — like a router — that lives in a corner and is controlled remotely.

---

## 2. Creating your first VM

Click **Create VM** in the Proxmox web UI. The tabs, and what actually matters in each:

| Tab | What to set | Why |
|---|---|---|
| General | A name (e.g. `srv1`) | Just an identifier |
| OS | Select an ISO you've uploaded | This is the installer disk for the VM's OS |
| System | Tick **Qemu Agent** | Lets Proxmox see the VM's IP address and manage it cleanly |
| Disks | Bus: SCSI, size in GB, tick **Discard** | Discard lets the VM tell Proxmox when space is freed, so thin-provisioned disks don't grow forever |
| CPU | Number of cores, Type: **host** | "host" passes through your real CPU's features for best performance |
| Memory | MB of RAM, Ballooning on | Ballooning lets the VM give back unused RAM to the host |
| Network | Bridge `vmbr0`, Model **VirtIO** | VirtIO is a much faster virtual network card than the emulated default |

**Uploading an OS ISO:** node → `local` storage → ISO Images → **Download from URL** (paste the distro's direct download link) — easier than downloading to your laptop and re-uploading.

**OS choice note:** Classic CentOS is discontinued. Its modern equivalents are **Rocky Linux** or **AlmaLinux**. For a lightweight server we used **Debian** — it uses less RAM idle than Rocky and is the same family Proxmox itself runs on.

### Installing the OS inside the VM
Open the VM → **Console**. Pick the plain **Install** (not Graphical) — same result, lighter on the VM console's mouse handling.

**The screen that trips everyone up: Software Selection.**
- Pressing **Enter** on this screen means "continue with what's ticked" — it does **not** toggle anything.
- Use **arrow keys** to move between items, **Space** to tick/untick, **Tab** to jump to the Continue button.
- For a lightweight server, you want exactly two things ticked: **SSH server** and **standard system utilities**. Untick every desktop environment (GNOME, Xfce, etc.) — a server doesn't need a graphical desktop.

**Partitioning:** "Guided – use entire disk" is safe — it only affects the VM's own virtual disk, never your real physical drive.

**GRUB:** say yes to installing it, to `/dev/sda`. GRUB is the bootloader — the piece of software that actually starts your OS when the VM powers on. Skipping this causes an endless "Booting from Hard Disk..." hang with nothing happening, because there's nothing bootable on the disk yet.

**After first boot**, before anything else, from the VM's console:
```bash
sudo apt update && sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```
Then eject the install ISO: VM → **Hardware** → CD/DVD Drive → Edit → **Do not use any media**. Otherwise it may try to boot the installer again.

---

## 3. SSH: controlling the VM from your laptop

SSH lets you get a command-line session on a remote machine, securely, over the network. Once it works you never need the Proxmox console again for daily use.

```bash
ssh username@10.0.0.10
```
First connection asks you to trust a "host key fingerprint" — type `yes`. Then enter the password you set during install.

**Common beginner trap:** typos in IP or username silently produce misleading errors. `ssh svr1@...` failing with "Permission denied" when the real user is `srv1` looks like a wrong password, but it's a wrong username. Always double check the exact characters.

### sudo vs su — a critical distinction
- `sudo <command>` — stay logged in as your normal user, borrow admin power for **one command**. This is the everyday-safe way to do admin tasks.
- `su -` — fully **become** the root (admin) user until you type `exit`. Useful for one-time setup (like installing `sudo` itself, which is a chicken-and-egg problem), but risky to leave open — every command runs with full power, including mistakes.

If a fresh minimal install says `sudo: command not found` or `you are not in the sudoers file`, fix it once via `su -`:
```bash
su -
apt update
apt install -y sudo curl
usermod -aG sudo yourusername
exit
```
Then **log out and back in** — group membership changes only apply on a fresh login, not the current session.

---

## 4. The CGNAT problem — why you can't just "open a port"

Traditionally, hosting something at home means: get a public IP from your ISP, forward a port on your router, done. This **does not work** on many home/mobile ISPs because of **CGNAT** (Carrier-Grade NAT) — your ISP shares one public IP across many customers, so there's no personal public IP to forward a port on.

This blocks both:
- Remote access to your own servers (SSH, the Proxmox UI) from outside your home
- Publishing anything to the public internet (a website)

Two different free tools solve these two different problems:

| Problem | Tool | How it works |
|---|---|---|
| I want to reach *my own* stuff from anywhere | **Tailscale** | Creates a private mesh network between your devices; no public IP needed |
| I want the *public internet* to reach a service | **Cloudflare Tunnel** | Your server dials **out** to Cloudflare; the public connects to Cloudflare, which relays in |

Both work by avoiding the need for an inbound connection entirely — your server always calls out, never waits for someone to call in.

---

## 5. Tailscale — private access from anywhere

Tailscale is WireGuard (a VPN protocol) under the hood, but it solves CGNAT by having every device "phone home" to a coordination server that introduces devices to each other, after which they talk directly and privately, end-to-end encrypted.

**Install on every device you want in the mesh** — the host, any VM, your laptop, your phone:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
Each run prints a login URL. Open it, sign in with the **same account** every time, click Authorize.

**Why the laptop needs to join too, not just the servers:** Tailscale is a mesh, not a one-way broadcast. A device can only *reach* other members if it is itself a member — the same way you can't access files on a LAN you're not plugged into.

**Check who's connected:**
```bash
tailscale status
```
Every device gets a permanent `100.x.x.x` address that works no matter what physical network it's on.

**Turn on MagicDNS** (in the Tailscale admin console, under DNS) to use names instead of numbers:
```bash
ssh srv1@srv1
ssh root@pve
```

**Free tier note:** the free plan supports up to 100 devices — a home lab with a handful of machines uses a tiny fraction of that.

---

## 6. Troubleshooting network gremlins we actually hit

These are worth understanding because they'll happen again.

### "Could not resolve host" (DNS failures)
DNS translates names (`tailscale.com`) into numbers computers actually use. If `ping 1.1.1.1` (a raw number) works but `ping tailscale.com` (a name) fails, it's specifically DNS that's broken, not your internet connection.

Some ISPs block direct requests to public DNS resolvers like `1.1.1.1`, forcing everyone through the router's own DNS. Fix: point the machine's DNS at your router instead:
```bash
echo "nameserver 10.0.0.1" > /etc/resolv.conf
```

### "SSL certificate problem: certificate is not yet valid"
This almost always means **the machine's clock is wrong**. HTTPS certificates have validity start dates; a clock stuck in the past makes current certificates look "not yet valid."
```bash
timedatectl                    # check current time + sync status
timedatectl set-ntp true       # try auto-sync
date -s "2026-08-03 21:30:00"  # manual fallback if NTP is blocked
```

### An existing VPN swallowing your traffic
If you already run a VPN (in our case, a WireGuard client, `wg0`, for a separate work network) it may be configured with very broad "AllowedIPs" — meaning it grabs *all* traffic matching certain address ranges, including traffic meant for your local network or even Tailscale. Symptom: pings to a local/Tailscale IP get answered by the VPN's gateway address instead of timing out or succeeding normally.
```bash
sudo wg-quick down wg0    # test with VPN off
sudo wg-quick up wg0      # bring it back after
```
If turning it off fixes the issue, both networks can usually coexist fine day-to-day — this is just a diagnostic step, not a permanent requirement to keep it off.

### Cloudflare Tunnel stuck retrying / "no recent network activity" (QUIC timeout)
`cloudflared` defaults to a protocol called QUIC, which runs over UDP. Some ISPs block or throttle UDP oddly, causing endless retry loops even though your normal internet works fine. Fix: force the tunnel onto HTTP/2 (which uses normal TCP) by adding one line to the tunnel's config file:
```yaml
protocol: http2
```

---

## 7. Cloudflare Tunnel — publishing a website with no public IP

The tunnel works like this: your server **initiates an outbound connection** to Cloudflare and keeps it open. When a visitor requests `yourdomain.com`, Cloudflare receives it at their edge and relays it down that already-open tunnel to your server. Your router/ISP never needs to accept an inbound connection — CGNAT becomes irrelevant.

### Steps
```bash
# 1. Install cloudflared
curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared

# 2. Authorize against your domain (opens a URL — use any browser, even on another device)
cloudflared tunnel login

# 3. Create a named tunnel — note the UUID it prints
cloudflared tunnel create homelab

# 4. Point your domain's DNS at the tunnel (creates the CNAME record automatically — no manual A/AAAA record needed, and none is possible anyway without a public IP)
cloudflared tunnel route dns homelab yourdomain.com
```

### Config file — maps a public hostname to a local port
```bash
sudo mkdir -p /etc/cloudflared
sudo nano /etc/cloudflared/config.yml
```
```yaml
tunnel: <your-tunnel-uuid>
credentials-file: /home/youruser/.cloudflared/<your-tunnel-uuid>.json
protocol: http2
ingress:
  - hostname: yourdomain.com
    service: http://localhost:8080
  - service: http_status:404
```
The last line (`http_status:404`) is required — it's a catch-all for any hostname that isn't explicitly listed above.

### Run it permanently
```bash
sudo cloudflared service install
sudo systemctl status cloudflared
```
Should show `active (running)`. This survives reboots.

### Prove it works
```bash
mkdir ~/www && echo "<h1>it works</h1>" > ~/www/index.html
cd ~/www && python3 -m http.server 8080
```
Then visit `https://yourdomain.com` from **mobile data**, not home wifi — this proves it's genuinely reachable from the outside internet, not just your own LAN.

### Reading a "Cloudflare Tunnel error / 1033"
This means DNS and the tunnel record are correct, but nothing is currently listening on the local port the config points to (or the tunnel service isn't running). It's a "plumbing connected, nothing plugged into the other end" error, not a DNS problem.

---

## 8. Where this leaves us, and what's next

**Built so far:**
- Proxmox hypervisor running on physical hardware, reachable at a static LAN IP
- One VM (`srv1`) with SSH access
- Tailscale mesh across host + VM + laptop → access from anywhere, no CGNAT issue
- Cloudflare Tunnel → a real domain serving a page from `srv1`, publicly, with no port forwarding and no public IP

**Planned next:**
- Hardware: adding a second RAM stick (8GB → 16GB) to comfortably run more services
- A second VM (or container) to separate concerns: one for website hosting, one for automation/AI-agent tooling (e.g. n8n) — keeping public-facing services isolated from internal automation
- Deploying a real site/app onto `srv1`, replacing the test page
- Note on "AI agents": actual LLM inference needs GPU power this hardware doesn't have. The realistic and correct architecture is running orchestration tools (like n8n) locally, which call out to hosted LLM APIs (e.g. OpenRouter) for the actual model reasoning — the box coordinates, it doesn't compute the AI itself.

---

## 9. Glossary (for explaining this to someone else)

- **Hypervisor** — software that lets one physical computer run several independent virtual computers.
- **VM (Virtual Machine)** — a fake computer running inside a hypervisor, with its own OS, indistinguishable from a real machine to software running on it.
- **LXC container** — a lighter alternative to a VM that shares the host's kernel instead of running a full separate OS; uses less RAM.
- **CIDR** — a way of writing an IP address plus how many devices it can share (e.g. `/24` means 256 addresses). `10.0.0.50/24` = "this device is `.50`, and it's part of a `10.0.0.x` network."
- **Static IP** — an address you fix in place; the opposite is DHCP, where the router assigns whatever's free.
- **CGNAT** — your ISP shares one public IP across many households, so no individual customer gets their own; makes classic "port forwarding" hosting impossible.
- **DNS** — the phonebook that turns names (`tailscale.com`) into numbers (an IP) that computers actually connect to.
- **SSH** — a secure way to get a command-line session on another computer over a network.
- **sudo vs su** — `sudo` borrows admin rights for one command; `su -` fully becomes the admin user until you exit.
- **Tailscale** — a private mesh VPN service that lets your own devices reach each other securely from anywhere, bypassing CGNAT.
- **Cloudflare Tunnel** — a service where your server makes an outbound-only connection to Cloudflare, who then relays public internet traffic to it — no inbound ports needed.
- **GRUB** — the bootloader; the software that actually starts your OS when a machine powers on.
