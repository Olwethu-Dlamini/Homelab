# My Homelab Journey: Proxmox, Tailscale, and Cloudflare Tunnel
### Written the way I actually lived it — mistakes, confusion, and all

I'm writing this the way I'd explain it to someone at my own level, because I *am* at my own level. I made real mistakes building this and I'm keeping them in, on purpose, because the mistake and the fix together are what actually teach you something. If you just read a clean list of correct commands, you learn the commands. If you read what went wrong and why, you learn how to *debug*, which is the actual skill.

---

## Part 0: What I was trying to build, and why it's harder than it sounds

I had one physical PC — an 8th-gen i5, spinning HDD (500GB), started with 8GB RAM (later upgraded to 16GB). My goal: turn it into a homelab. Not just "run a VM" — I wanted:

1. A hypervisor (Proxmox) so I could run multiple isolated virtual servers on one machine
2. At least one VM I could SSH into and control like a real server
3. My own domain (`22112002.xyz`, already sitting in Cloudflare) actually pointing at something I host myself
4. Eventually: one "server" for hosting websites (mine and other people's), and a separate one for automation/AI-agent tooling like n8n

The complication that shaped almost every decision after this: **I'm on wifi with a shared/CGNAT IP address. I don't get my own public IP.** This one fact is why half of this guide exists. Let me explain it properly because it's the single most important concept here.

### Why CGNAT breaks the "normal" way of self-hosting

The traditional way to host something at home: your ISP gives you a public IP, you log into your router, you "forward" a port (say, port 22 for SSH, or port 443 for a website) to your server's local address, and now anyone on the internet who connects to `your-public-ip:443` gets routed straight to your server.

**CGNAT (Carrier-Grade NAT)** means your ISP doesn't give *you* a public IP — it gives one public IP to a whole pool of customers and shares it, using its own internal NAT to sort out who gets what traffic. You have no port to forward, because you don't own the public IP in the first place. Port forwarding on your own router does nothing, because your router's "public" side isn't actually public — it's just another private address one layer up, inside your ISP's network.

This is why I couldn't just "open a port" and be done with it. I needed tools that don't depend on receiving inbound connections at all — tools where *my* server reaches *out* to something, instead of waiting for the world to reach *in*.

This is also, directly, **why plain self-hosted WireGuard wasn't an option for the "access my server from anywhere" problem.** Self-hosted WireGuard needs one side to be reachable — normally the server, with a port forwarded on a public IP. Without a public IP, there's no address for the WireGuard client (my laptop) to dial when I'm away from home. I don't own an address that exists on the public internet. It's not a configuration problem I could fix with more effort — it's structurally impossible without a public IP or some relay in front of it. This is exactly the gap Tailscale fills, and I get into that properly in Part 5.

---

## Part 1: Installing Proxmox — my first real mistakes

Proxmox is the hypervisor — software that turns one physical machine into a host for many virtual ones. You install it USB-boot style, like installing any OS.

### Mistake #1: I didn't understand IP vs hostname, and I mixed them up

On the **Management Network** screen, there are separate fields:
- **Hostname (FQDN)** — a *name* for the machine, like `pve.22112002.xyz`
- **IP Address (CIDR)** — the actual *numeric address*, like `10.0.0.50/24`

I actually typed my domain name into places expecting a number at one point, because in my head "put your domain here" applied to every field on the screen. It doesn't. The hostname is just a label the machine calls itself — it doesn't need to exist anywhere in real DNS, and typing it doesn't connect anything to the internet. The IP address is the actual number other devices on my network use to find this machine. These are two completely different concepts wearing similar clothing, and conflating them is an extremely common first mistake.

**The fix/lesson:** whenever a setup screen asks for "hostname" vs "IP," stop and ask: *does this field want a name a human reads, or a number a computer routes to?* If unsure, the format is the tell — IPs are four numbers separated by dots (plus a `/number` for CIDR), hostnames are words separated by dots.

### Mistake #2: I thought I needed internet access *during* the install to get an IP

I got confused and asked whether I needed to connect to the internet right away to "get" an IP address. This revealed a misunderstanding worth naming directly: **I'm not "getting" an IP from anywhere at this stage. I'm choosing one and telling the installer to use it.**

A static IP isn't handed to you by an authority — you pick an unused number inside your own network's range (found by checking what your router already hands out to other devices via `ip a` on any already-connected device) and claim it for this machine, permanently. No internet connection is needed to type a number into a form.

Internet-slash-network access only starts mattering *after* install, once I want to run `apt update`, install packages, or reach the outside world from the box.

### Mistake #3: I picked "wifi" as an option without realizing hypervisors basically can't use wifi properly

I have wifi available, and briefly considered using it for the Proxmox host. This doesn't really work. Hypervisors "bridge" the network to virtual machines — meaning each VM effectively needs to send traffic that looks like it's coming from its own separate device on the network. Wifi hardware and most access points reject this behavior for VMs (a wifi card is built to be one device with one identity, not a bridge for many). Ethernet doesn't have this restriction. So: **the physical server needs a wired ethernet connection, full stop.** My laptop, which just *accesses* the server, can stay on wifi without issue — the restriction is specifically about the hypervisor box bridging traffic to guests.

### Mistake #4: DHCP showing up when I expected a number

At one point the IP field showed "DHCP" instead of a number. This wasn't broken — it meant no ethernet cable was plugged in yet, so the installer had nothing to detect and just displayed a placeholder. The fix was physically plugging in the cable (which, in my case, meant literally moving a cable from a different computer over to the Proxmox machine) and/or just typing the static IP manually regardless of what the placeholder showed.

**Finding the right numbers:** I ran `ip a` and `ip r` on a computer that *was* already connected via that ethernet cable, which told me the real network range (`10.0.0.x`, gateway `10.0.0.1`). I used `10.0.0.50/24` for Proxmox — same first three numbers as everything else on the network, a different, unused last number.

### What I actually ended up entering
| Field | Value | Why |
|---|---|---|
| Management interface | the ethernet NIC | wifi can't bridge to VMs |
| Hostname (FQDN) | `pve.22112002.xyz` | just a self-referential label, reused my real domain for tidiness |
| IP (CIDR) | `10.0.0.50/24` | static, outside DHCP's range, matches my LAN |
| Gateway | `10.0.0.1` | my router |
| DNS | `1.1.1.1` initially (this came back to bite me — see Part 6) |

### Right after install
```bash
rm /etc/apt/sources.list.d/pve-enterprise.list
rm /etc/apt/sources.list.d/ceph.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list
apt update && apt full-upgrade -y
```
This removes references to Proxmox's paid subscription repos (which I don't have access to and shouldn't be pointed at) and switches to the free "no-subscription" repo, which is fully functional for a homelab.

### The mental shift I had to make
I asked, genuinely, "why can't I use this PC anymore?" — because after install, there's no desktop, no browser, just a black console screen. This confused me until I reframed it: **Proxmox isn't an operating system you use directly, it's an appliance, like a router.** You don't sit at your router typing into it every day — you configure it once, then manage it remotely, and it just runs in a corner. That's exactly the relationship I now have with this machine. Everything happens from my laptop, at `https://10.0.0.50:8006`, in a browser. The physical screen and keyboard on the box are now purely an emergency fallback.

---

## Part 2: Creating a VM — where I learned that "Enter" isn't always "confirm"

A VM is a fully virtual computer with its own OS, running inside Proxmox. I wanted a lightweight server, initially thinking "CentOS" — but classic CentOS is discontinued; its real modern equivalents are **Rocky Linux** or **AlmaLinux**. I ended up going with **Debian**, since it's lighter on RAM at idle and it's literally the same family Proxmox itself is built on.

### Setting it up
The Create VM wizard walks through tabs. The settings that actually matter, and *why*:

- **Qemu Guest Agent (tick it)** — a small helper program that runs inside the VM and reports information (like its IP address) back to Proxmox. Without it, the Summary page can't show you the VM's IP at all — which is exactly the confusion I hit later (see Mistake below).
- **Disk bus: SCSI + Discard ticked** — Discard means when I delete files inside the VM, the underlying disk space actually gets reclaimed by Proxmox, instead of the virtual disk file growing forever even as the VM's own filesystem frees space.
- **CPU Type: host** — passes through the real CPU's actual feature set to the VM, for the best possible performance match.
- **Network model: VirtIO** — a virtualization-aware network driver that's dramatically faster than the default emulated hardware.

### Mistake #5: I hit Enter on the software-selection screen without reading it

This was the single most time-costly mistake in the whole project. During the Debian installer's **Software Selection** screen, I pressed Enter, assuming it would move me forward after confirming my choices. It didn't ask me anything — it just accepted whatever was pre-ticked (which included a full **desktop environment**, GNOME) and started installing it.

**Why this mattered:** installing a full desktop environment on a headless server (a) wastes a large amount of disk space and RAM I explicitly don't have to spare, and (b) meant I was now sitting through a much longer install on a slow spinning HDD than necessary, for something I'd never use, since this machine has no monitor of its own and I only ever access it through SSH or the Proxmox console.

**The actual mechanics of that screen**, which I didn't know at the time:
- **Arrow keys** move between options
- **Space** toggles a checkbox on/off
- **Enter/Tab** only matters once you've navigated to the "Continue" button itself

The correct end state has exactly two boxes ticked: **SSH server** and **standard system utilities**. Every desktop environment entry should be unticked.

### Mistake #6: I killed the install partway through, which corrupted the boot process

Once I realized the desktop environment was installing, I stopped the VM to restart the install cleanly. This left the virtual disk in a half-written state — no working bootloader, an incomplete filesystem. The result was the next mistake:

### Mistake #7: "Booting from Hard Disk" that never went anywhere

After restarting, the VM got stuck at a black screen saying it was booting from the hard disk, forever. Two different causes can produce this exact symptom, and I hit both across this project:

1. **The install genuinely never finished** (my case, from stopping it mid-way) — there's no bootloader on the disk yet, so "booting from hard disk" has nothing to actually boot.
2. **The boot order in Proxmox has the CD-ROM below the hard disk** — even with a real install finished and a real bootloader, if the ISO is still attached and boot order has other quirks, you can get similar-looking symptoms.

The fix, in order: stop the VM properly, check **Options → Boot Order** to make sure the CD-ROM/ISO is enabled and higher in the boot order than the hard disk, start the VM again, and this time run the installer all the way through to a genuine "Installation complete" message — including saying **yes** to installing GRUB (the bootloader) to `/dev/sda`. GRUB is the actual software that turns on the OS when the VM boots; skipping this step is exactly what produces an unbootable disk.

Once GRUB was properly installed and the ISO ejected (Hardware → CD/DVD Drive → **Do not use any media**), the VM booted straight to a login prompt like a real server should.

### Mistake #8: no `sudo`, no `curl` — the minimal install strips more than you expect

Fresh into the new VM, two things bit me back to back:

```
srv1@srv1:~$ sudo apt install -y qemu-guest-agent
-bash: sudo: command not found
```

Debian's *minimal* install doesn't include `sudo` at all if you set a root password during setup (it assumes you'll just `su -` when needed). This meant every "just run sudo" instinct I had from other systems didn't work yet.

**The fix required a specific order**, because you can't `sudo install sudo` — there's no sudo yet to do the installing:
```bash
su -                      # become root directly, using the ROOT password (not your user's)
apt update
apt install -y sudo curl
usermod -aG sudo srv1     # add my normal user to the group allowed to use sudo
exit
```

### Mistake #9: pasting a whole block of commands at once, racing the password prompt

I pasted several lines together, including the `su -` command *and* the commands meant to run only after `su -` finished asking for a password. The terminal doesn't wait patiently for a multi-line paste to resolve one prompt at a time — it fired the later commands before root access was actually established, so they ran as my normal unprivileged user and failed with permission errors like:
```
Error: Could not open lock file /var/lib/apt/lists/lock - open (13: Permission denied)
```

**The lesson:** when a command is going to prompt for input (a password, a yes/no confirmation), don't paste it bundled together with commands meant to run *after* that prompt resolves. Run the prompting command by itself, wait for it to actually finish, then run the next one.

I also once tried to log in with `su -` again after already exiting, and pasted a comment line (`# root password`) as if it were an actual command — comments starting with `#` in a guide are notes-to-self, not something to type into the terminal.

### Mistake #10: `svr1` vs `srv1` — the classic transposed-letters bug
```
ssh svr1@100.72.72.58
...
Permission denied (publickey,password)
```
My actual username was `srv1` (s-r-v), and I kept typing `svr1` (s-v-r). This produces a *misleading* error — "permission denied" sounds like a wrong password, but the real problem was an account that simply doesn't exist by that spelling, so no password would ever work. When "permission denied" happens right after typing a password you're sure is correct, check the *username* character by character before assuming the password is wrong.

---

## Part 3: SSH and the sudo/su distinction, properly explained

Once inside a VM's console, the two most important admin patterns are:

**`sudo <command>`** — you remain logged in as your everyday, low-privilege user. For just *this one command*, you temporarily borrow admin rights (after typing your password to prove it's really you). This is the pattern to use almost all the time, because a typo in a `sudo`-prefixed command only affects that one line.

**`su -`** — you fully *become* the root user, using root's own password. Your prompt changes from `$` to `#` as a visible warning sign. Every single command now runs with full system power, whether you meant it to or not, until you type `exit`. I used this exactly once, deliberately, to install `sudo` itself — a genuine chicken-and-egg situation where sudo doesn't exist yet to bootstrap itself.

**Rule I settled on for myself:** if my prompt shows `#`, finish the specific task and `exit` immediately. Day-to-day work happens as my normal user, with `sudo` in front of anything that needs elevated rights.

---

## Part 4: Why not just WireGuard? (the reasoning, in full)

I asked directly at one point: why not just set up WireGuard myself instead of using Tailscale, especially since I generally prefer self-hosted tools over managed services (this is a real, consistent preference of mine from other projects — I self-host Qdrant, use Docker Compose over managed platforms, etc.).

Here's the honest reasoning for why this specific case is different:

**Plain, self-hosted WireGuard requires one side of the connection to be reachable at a known address.** Normally that's the server: you'd forward a UDP port on your router to your WireGuard server, and give that port + your public IP to any client that wants to connect. Clients (like my laptop, out in the world) dial in to that known, fixed, public address.

**I don't have a public IP at all — that's the entire premise of CGNAT.** There is no address to forward a port on, because the "public-facing" side of my own router isn't actually public; it's just another private address one layer up inside my ISP's shared pool. Self-hosting the WireGuard *server* portion doesn't fail because I configured it wrong — it fails because there's no reachable address to put in the configuration in the first place. It's not a skill gap, it's a missing prerequisite.

**The only way around this** is putting something with a real public IP in front of the connection, acting as a rendezvous point — either:
1. Rent a cheap VPS with a real public IP and run something like Headscale (a self-hosted alternative to Tailscale's coordination server) on it, or
2. Use Tailscale's own free coordination service, which is exactly this, already running, at no cost.

**Tailscale is, underneath, still WireGuard.** It's not a different, less secure protocol — it uses the same modern cryptography, and my actual traffic between devices is end-to-end encrypted and usually flows *directly* between them once they're introduced (with Tailscale's servers only used briefly to broker that introduction, and occasionally as a relay if direct connection truly can't be established through very hostile networks). So this isn't a case of abandoning self-hosting principles for convenience — it's recognizing that this *particular* piece (the public rendezvous point) genuinely cannot be self-hosted without renting a machine that has a public IP, which defeats a chunk of the "everything on my own hardware" goal anyway. Tailscale's free tier does the identical job at zero cost and zero additional hardware.

I did keep my *existing* WireGuard VPN (a separate `wg0` connection I already use for other purposes, unrelated to this homelab) running alongside Tailscale. They're two separate WireGuard-based networks that generally coexist fine — see Part 6 for the one specific way they interfered with each other.

---

## Part 5: Setting up Tailscale — the actual steps, and what "device accepted" really means

Install on every device that needs to be part of the mesh — this included the Proxmox host, the VM, and my own laptop. Skipping the laptop would have meant I had no device *inside* the mesh to actually reach the others from — a mesh with only servers in it and no client is useless to me personally.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Each run prints a URL. Opening it and logging in (with the *same account* every single time — this part matters, using different accounts per device would put them in separate, non-connected tailnets) registers that specific machine's cryptographic identity against my account.

**What "accepted" actually does under the hood:** the coordination server now knows this device belongs to me, assigns it a permanent `100.x.x.x` address that stays the same no matter what physical network the device is on, and tells every other device already in my tailnet how to reach it directly. After that hand-off, the devices talk to each other peer-to-peer, encrypted, without needing the coordination server anymore for the actual data.

Checking the roster:
```bash
tailscale status
```
Three lines, one per device, each with its own `100.x` address.

I turned on **MagicDNS** in the Tailscale admin console afterward, which lets me use plain names instead of memorizing numbers — `ssh srv1@srv1` instead of typing the IP.

One clarifying misconception I had: I thought the free tier's device limit was something like 3 (based on how many machines I currently had). It's actually 100 devices on the free "Personal" plan — my 3 machines use a tiny fraction of that, and I could add several more servers, my phone, anything, without hitting a paywall.

---

## Part 6: All the networking gremlins, explained properly (not just fixed)

These weren't random bad luck — each one taught me something about how networks actually behave, and I'll probably hit variations of these again.

### Gremlin 1: "Could not resolve host" — DNS being silently blocked
```
curl: (6) Could not resolve host: tailscale.com
```
but
```
ping -c 2 1.1.1.1   →   worked fine
```
**What this split result actually proves:** my internet connection itself was fine (raw IP traffic worked), but the *specific step of turning a name into a number* — DNS resolution — was failing. I'd set my DNS server to `1.1.1.1` (Cloudflare's public resolver) during install, which is normally a completely reasonable choice. The actual cause turned out to be that my ISP appears to block or interfere with direct queries to public DNS resolvers, silently forcing everyone through the ISP's own DNS instead.

**The fix** was pointing DNS at my own router instead of a public resolver:
```bash
echo "nameserver 10.0.0.1" > /etc/resolv.conf
```
which immediately started resolving names correctly. The general debugging principle here, worth keeping: *if pinging a raw number works but pinging a name doesn't, the problem is specifically DNS, not "the internet."*

### Gremlin 2: "SSL certificate is not yet valid" — a wrong clock, not a broken cert
```
curl: (60) SSL certificate problem: certificate is not yet valid
```
This looked like a security/certificate problem, but the actual cause was much simpler and easy to miss: **the VM's system clock was wrong** (stuck months in the past, likely because the VM inherited a stale clock at boot and never synced). HTTPS certificates carry a validity *start* date; if your own clock thinks "now" is before that date, the cert legitimately looks invalid from the machine's point of view, even though it's perfectly fine in reality.
```bash
timedatectl                    # check
timedatectl set-ntp true       # try to auto-correct via internet time servers
date -s "2026-08-03 21:30:00"  # manual fallback if NTP itself is blocked
```
**The lesson:** certificate errors aren't always about certificates. Check the clock first — it's a five-second check that rules out an entire category of confusing failures.

### Gremlin 3: my own existing VPN swallowing traffic meant for something else
```
ping -c 3 10.0.0.50
From 10.8.0.1 icmp_seq=1 Destination Host Unreachable
```
I was trying to ping my Proxmox server's LAN address, but the reply came from `10.8.0.1` — that's not my router, that's the gateway address of my *other*, pre-existing WireGuard VPN (`wg0`, used for unrelated work purposes). This meant my laptop's own VPN configuration was intercepting traffic destined for `10.0.0.x` addresses and routing it into the VPN tunnel instead of onto my actual home network — almost certainly because that VPN's configuration uses very broad "AllowedIPs" rules that accidentally capture more traffic than intended.

**Fix for testing:** temporarily bring the VPN down, confirm the real target becomes reachable, then bring it back up once done:
```bash
sudo wg-quick down wg0
sudo wg-quick up wg0
```
Later, once Tailscale was fully set up, I confirmed both VPNs (the old `wg0` and the new Tailscale) could run *simultaneously* without conflict — the earlier conflict was specifically about `wg0` vs my plain LAN address, not `wg0` vs Tailscale's separate `100.x` address range.

**The lesson:** if a device that should obviously be reachable gives "Destination Host Unreachable" from a completely unexpected IP, that unexpected IP is a huge clue — it's telling you exactly which piece of software is wrongly intercepting the traffic.

### Gremlin 4: Cloudflare Tunnel stuck in a retry loop — UDP being blocked
```
ERR Failed to dial a quic connection error="failed to dial to edge with quic: timeout: no recent network activity"
INF Retrying connection in up to 2s
```
The tunnel software (`cloudflared`) defaults to a protocol called **QUIC**, which runs over UDP. My ISP — the same one that was quietly interfering with DNS — also appears to throttle or block this kind of UDP traffic, causing an endless connect-fail-retry loop even though my regular internet worked completely normally.

**Fix:** force the tunnel to use a different, TCP-based protocol instead, by adding one line to its config file:
```yaml
protocol: http2
```
After that change and reinstalling the service, the tunnel connected immediately and stayed in an `active (running)` state.

**The broader lesson from gremlins 1 and 4 together:** my specific ISP seems to interfere with certain kinds of "unusual" outbound traffic (external DNS queries, UDP-based QUIC) while leaving totally normal traffic alone. This is common on some consumer and mobile-style ISPs. Whenever something that "should just work" mysteriously times out or hangs, forcing a fallback to a more conventional protocol (regular DNS via the router, HTTP/2 over TCP instead of QUIC) is a reasonable and often successful troubleshooting step.

---

## Part 7: Cloudflare Tunnel — getting the actual domain live

The problem this solves is the exact mirror image of Tailscale's problem: Tailscale gets *me* into *my own* network from anywhere; Cloudflare Tunnel gets *the public internet* into a specific service on my network, despite having no public IP to receive that connection on.

**How it actually works:** `cloudflared`, running on my VM, opens an *outbound* connection to Cloudflare and holds it open persistently. When someone out on the internet requests `22112002.xyz`, Cloudflare's edge receives that request (they own real, public IPs) and forwards it down the already-open tunnel to my VM. My side never had to accept an inbound connection — from my ISP's point of view, my VM just made a normal outgoing connection, the same as visiting any website, which CGNAT has no problem with at all.

```bash
# Install
curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared

# Authorize against my domain (opens a URL — fine to open on a different device, like my phone, since the VM has no browser)
cloudflared tunnel login

# Create the tunnel, note the UUID it prints
cloudflared tunnel create homelab

# Point the domain at it — this auto-creates a CNAME record; I don't need, and can't have, an A/AAAA record, because I have no public IP to put in one
cloudflared tunnel route dns homelab 22112002.xyz
```

Config file (this is the part where I also added the `protocol: http2` fix from Gremlin 4):
```yaml
tunnel: <tunnel-uuid>
credentials-file: /home/srv1/.cloudflared/<tunnel-uuid>.json
protocol: http2
ingress:
  - hostname: 22112002.xyz
    service: http://localhost:8080
  - service: http_status:404
```
That last catch-all line is required — without it, any hostname not explicitly listed above gets rejected outright.

Installed as a persistent service so it survives reboots:
```bash
sudo cloudflared service install
sudo systemctl status cloudflared
```

**Proof it actually worked**, and why I specifically tested it this way: I stood up a trivial test page and requested it from my phone on **mobile data**, deliberately *not* on my home wifi:
```bash
mkdir ~/www && echo "<h1>22112002.xyz is live</h1>" > ~/www/index.html
cd ~/www && python3 -m http.server 8080
```
Testing from home wifi would only prove it works on my own LAN, which was never in question. Testing from mobile data proves a genuine stranger anywhere on the internet could load it — the actual goal.

**Reading a "Cloudflare Tunnel error / Error 1033"** when it appeared: this specifically means the tunnel and DNS routing are correctly set up, but *nothing is currently listening on the local port the config points to* (or the tunnel service itself isn't running). It's a "the pipe is connected, but nothing's plugged into this end of it" error — not a DNS problem and not a tunnel-setup problem, which is an important distinction when trying to figure out what to actually fix.

---

## Part 8: Where I am now, and what's next

**Currently working, end to end:**
- Proxmox hypervisor, installed correctly, reachable at a static LAN address, no sleep, survives reboots
- One VM (`srv1`), Debian, minimal, SSH access with proper sudo rights
- Tailscale mesh across the host, the VM, and my laptop — I can reach either server from literally anywhere, with no public IP and no port forwarding, and it coexists fine with my separate pre-existing VPN
- Cloudflare Tunnel live on `22112002.xyz`, serving a real page to the actual public internet, again with no public IP and no port forwarding

**Planned next, and my reasoning for each:**
- **RAM upgrade (8GB → 16GB):** the original plan of two separate VMs (one for hosting websites, one for automation/AI tooling) was too tight on 8GB once I accounted for host overhead plus two full guest OS's. 16GB makes running two proper VMs comfortable instead of forcing me into containers as a compromise.
- **Second server (website hosting vs automation, kept separate):** deliberately isolating these two roles — a website that other people might eventually visit shouldn't share a blast radius with internal automation tooling that has broader access to my own accounts/services. Same reasoning that pushed me toward Tailscale + Tunnel as two *separate* tools for two separate problems, applied one level up.
- **"AI agents" reality check:** I want to run things like n8n for automation, but actually running an LLM's *inference* locally needs a GPU I don't have. The correct architecture is n8n (or similar) running locally as the orchestrator, calling out to a hosted model API (like OpenRouter, which I've already used in an earlier project) for the actual model reasoning. The box coordinates workflows; it doesn't need to be the thing doing the heavy AI computation.
- **Disk is still the long-term weak point:** everything above works, but it's all running on one spinning HDD. If this grows into something people other than me actually rely on, an SSD is the single upgrade that would matter more than any further RAM increase.

---

## Part 9: Glossary, written the way I'd want it explained

- **Hypervisor** — software (Proxmox, in my case) that lets one physical computer host several independent virtual computers at once.
- **VM (Virtual Machine)** — a fully virtual computer with its own OS, running inside a hypervisor. From the inside, software running in it can't tell it's not a real, standalone machine.
- **CIDR notation** — an IP address plus a `/number` that says how many addresses share that network, e.g. `10.0.0.50/24` means "this device is `.50`, and shares a network with everything from `10.0.0.0` to `10.0.0.255`."
- **Static IP vs DHCP** — static means I fixed the address myself, permanently; DHCP means the router hands out whatever's free, which can change over time — bad for anything you need to reliably find again, like a server.
- **CGNAT** — my ISP shares one public IP address across many customers using its own internal NAT, meaning I never get a public IP of my own, and traditional port-forwarding-based self-hosting is not possible without a workaround.
- **DNS** — the system that turns names (`tailscale.com`) into the actual numeric addresses (IPs) computers use to connect to each other.
- **sudo vs su** — `sudo` borrows admin rights for a single command while staying as my normal user; `su -` fully becomes the admin (root) user until I explicitly exit, which is riskier to leave open.
- **WireGuard** — a modern, fast VPN protocol. It's the underlying encryption technology inside both my pre-existing personal VPN *and* Tailscale — Tailscale isn't a different, less secure alternative to WireGuard, it's WireGuard with an added coordination layer that solves the "no public IP to dial" problem.
- **Tailscale** — a service built on WireGuard that lets my own devices find and reach each other directly and securely from anywhere, without needing a public IP, by using a shared coordination server just to introduce devices to each other.
- **Cloudflare Tunnel** — a service where my server makes an outbound-only connection to Cloudflare, and the public internet's requests get relayed down that connection — meaning I can publish a real website with no public IP and no forwarded ports.
- **GRUB** — the actual bootloader software that starts an operating system when a machine (real or virtual) powers on; if it's missing or not installed, the machine has nothing to boot into, even if an OS is technically present on the disk.
- **QUIC vs HTTP/2** — two different ways for two computers to talk over the internet; QUIC uses UDP and is newer/often faster, HTTP/2 uses the older, more universally-tolerated TCP. Some ISPs are stricter about UDP, which is why falling back to HTTP/2 fixed my tunnel connection issue.
