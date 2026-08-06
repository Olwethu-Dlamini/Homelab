# LinkedIn Series: From "What Is a Hypervisor?" to a Live Server in My House
### 29 posts, one continuous story. Intro first, then the concepts, then the real build with every mistake kept in.
### Post 2-3 per week, in order. Repo link goes in the comments, not the post body.
### Every post has an image suggestion in a quote block under the title. The suggestion is a note for you, NOT part of the post text. Delete it before posting.
### General image rules: your real photos and real terminal screenshots beat stock images and AI art every time. Dark terminal, tight crop, one idea per image. Reuse the same diagram style across the series so your posts become visually recognizable in the feed.

---

## Post 1: I turned the old PC in my house into a real server. This series is everything I learned.

> **Image suggestion:** A real photo of the actual PC. On the floor or a desk, cables visible, slightly messy is good. Phone camera, natural light. If you want a stronger hook, place your laptop next to it showing the live site. Real hardware photos massively outperform stock images and AI art on LinkedIn.

There is a PC in my house that had no job.

An 8th-gen i5, a 500GB hard drive that still spins like it is 2012, and 8GB of RAM. Not good enough to game on. Too good to throw away. You probably have one like it, or know someone who does.

That machine now runs my own private cloud. It hosts a live website at my own domain, 22112002.xyz, that anyone in the world can visit. I can SSH into it from anywhere on the planet. And soon it will run my automation tooling too.

Total monthly cost of the software and services making that possible: zero.

Here is what makes this worth writing about. I did it in Eswatini, on a shared wifi connection, on a connection with no public IP address, the kind of setup most home and mobile connections around the world actually have. Every guide I found assumed conditions I simply do not have. So I had to actually understand what I was doing, not just follow steps.

And I made mistakes. Real ones. I installed a full desktop environment on a server with no screen. I corrupted a boot disk by being impatient. I debugged a "wrong password" for an embarrassing amount of time before noticing I had misspelled my own username.

This series is the whole thing, in order, mistakes included:

First the concepts, explained the way I wish someone had explained them to me. What virtualization actually is. What a hypervisor does. How a virtual machine can run at full speed without knowing it is fake. Why snapshots are not backups.

Then the build itself. Every challenge, every wrong theory, every fix, and the reasoning behind every decision, including the tools I looked at and deliberately rejected.

My promise for the series: no step lists you copy without understanding. When I show a command, I explain what it does and why it exists. When something broke, you will see my wrong first theory before you see the fix, because the gap between those two is where the learning lives.

If you have ever wanted your own server but assumed you needed money, fancy hardware, or better internet, stay with me. You need an old machine, a cheap domain, and patience with your own mistakes.

The learning starts in the next post, with the question everything else is built on: what is a hypervisor?

#homelab #selfhosting #linux #devops #learninpublic #eswatini

---

## Post 2: What is a hypervisor? (The question I should have answered before touching my server)

> **Image suggestion:** Remake the layered ASCII diagram (guests, hypervisor, hardware) as a clean graphic in your style, dark background, three colored boxes on top of one. You already do pixel-perfect infographics, this is a 30-minute job that becomes your series' visual signature.

Before I tell you about all the things that broke while building my home server, let me answer the question I get most often when I mention it:

What even is a hypervisor?

Here is the simplest way I can put it. A hypervisor is software that lets one physical computer pretend to be many computers at the same time.

Each of those pretend computers is called a virtual machine, a VM. And a VM is not some watered-down thing. It has its own CPU, its own memory, its own disks, its own network card. All virtual, all carved out of the real hardware underneath. The operating system inside a VM genuinely believes it owns a whole computer. It has no idea it is sharing.

The hypervisor sits in the middle and does two jobs. It hands each VM a slice of the real machine. And it keeps them walled off from each other, so a crash in one cannot touch the others.

Some vocabulary you will see everywhere once you enter this world: the physical machine is called the host. Every VM running on it is a guest.

```
   Guest A      Guest B      Guest C     <- virtual machines
  (Debian)    (Windows)    (pfSense)
      |            |            |
      +------------+------------+
                   |
              Hypervisor                 <- divides and isolates
                   |
               Hardware                  <- one physical PC
```

Now, why would anyone do this instead of just installing one OS like a normal person?

Because one capable machine can replace several under-used ones. In my case, one old PC with an 8th-gen i5 runs my web hosting and will soon run my automation tooling too. Separate servers, one electricity bill.

Because isolation saves you. When I misconfigure something on one VM, and trust me, I do, the other one keeps running like nothing happened.

Because a VM is just files on the host. You can snapshot it before a risky change and roll back in seconds. You can back it up. You can clone it. Try doing that with a physical computer.

That mental model, one real machine hosting several fake ones that do not know they are fake, is the foundation of everything I built. I put together a whole set of notes explaining these concepts properly, link in the comments.

Next post: not all hypervisors are the same, and the difference explains why VirtualBox on your laptop feels nothing like a real server.

#homelab #virtualization #linux #selfhosting

---

## Post 3: Type 1 vs Type 2 hypervisors, or why VirtualBox is not a server

> **Image suggestion:** Side-by-side diagram: Type 1 stack vs Type 2 stack, same visual style as post 2. Alternatively a photo of your laptop running VirtualBox next to the headless tower, captioned 'same idea, different worlds'.

If you have ever run VirtualBox on your laptop, you have already used a hypervisor. So why does building a server feel so different?

Because hypervisors come in two flavours, and the difference is what sits underneath them.

Type 2, the one you probably know, is just an application. Your laptop boots Windows or macOS or Linux like normal, and the hypervisor runs on top as another program, next to your browser and your music. VirtualBox, VMware Workstation, Parallels. All Type 2.

```
 [ VM ]  [ VM ]
    |       |
 Hypervisor (an app)
 Host OS (Windows / macOS / Linux)
 Hardware
```

It is perfect for testing things on your laptop. It is also carrying dead weight for a server, because every VM's work travels through a full desktop OS that exists mostly to show you a wallpaper.

Type 1 removes that layer entirely. The hypervisor IS the operating system. It installs directly onto the hardware, bare metal as people say, and everything else runs as a guest on top.

```
 [ VM ]  [ VM ]  [ VM ]
    |       |       |
    Hypervisor
    Hardware
```

Proxmox VE, which I use, is Type 1. So are VMware ESXi, XCP-ng, and Hyper-V. Less overhead, better performance, stronger isolation. The right shape for a machine that runs 24/7 in a corner and never shows anyone a desktop.

This is exactly the mental shift that confused me on day one. I installed Proxmox on my PC, saw a black screen with a login prompt, and genuinely wondered if I had broken the computer. Where was the desktop?

There is no desktop. That is the point. A Type 1 hypervisor turns your PC into an appliance, like a router. You configure it once, then manage everything remotely from a browser on another machine. The box just hums in the corner doing its job.

One line that blurs nicely: Linux has a kernel module called KVM that turns the Linux kernel itself into a Type 1 style hypervisor. Proxmox is actually built on that. Which brings up a fair question. How does a VM not notice it is fake? How do three operating systems all believe they control the same CPU without fighting?

That is the next post, and the answer involves a BIOS setting that has ruined many people's first day.

#virtualization #proxmox #homelab #linux

---

## Post 4: How does a VM not know it is fake?

> **Image suggestion:** Photo of your actual BIOS screen showing the virtualization setting (Intel Virtualization Technology / VT-x). Everyone recognizes a BIOS photo taken with a phone, and it is a screen almost nobody ever screenshots, so it stands out. Backup option: terminal screenshot of the grep vmx command with output.

This is my favourite part of the whole topic, because the answer is genuinely clever.

Here is the problem. A CPU runs code at different privilege levels. An operating system kernel expects to run at the MOST privileged level, where it can touch hardware directly. It is built on the assumption that it is the boss.

Now put three guest operating systems on one hypervisor. All three kernels believe they are the boss of the same CPU. If they all actually ran at full privilege, they would trample each other instantly.

Early virtualization solved this in software. The hypervisor would catch dangerous instructions from guests and quietly rewrite them. Clever, but slow.

Modern CPUs solve it in hardware. Intel calls it VT-x, AMD calls it AMD-V. These extensions add a special guest mode to the CPU itself. A guest kernel runs at full native speed, feeling like the boss, until it tries something that needs supervision. At that exact moment the CPU traps back to the hypervisor, which handles the situation and resumes the guest. The guest never notices the interruption.

So the VM is not slowly emulated. It runs on the real CPU, at real speed, inside a hardware-enforced sandbox. That is why VMs today feel nearly native.

Now the practical part, and the one that bites beginners.

These extensions are often switched OFF from the factory. Buried in your BIOS or UEFI settings under names like Intel Virtualization Technology, VT-x, or SVM Mode on AMD. If it is off, a Type 1 hypervisor will refuse to install, or your VMs will crawl, and nothing on screen will clearly tell you why.

On an existing Linux machine, ten seconds tells you if your CPU has it:

```
grep -oE 'vmx|svm' /proc/cpuinfo | sort -u
```

Any output means yes. vmx is Intel, svm is AMD.

One more piece completes the picture. CPU and memory get virtualized by the hardware, but a computer is also disks, network cards, a BIOS. On Linux, the KVM kernel module drives those CPU extensions, and a program called QEMU emulates the rest of the machine. Proxmox is a management layer on top of that pair. So when I click "Create VM" in a friendly web interface, this is the actual stack underneath:

```
Proxmox (web UI, API)
QEMU (fake disks, fake network cards)
KVM (kernel module driving the CPU)
VT-x / AMD-V (the hardware itself)
```

There is a lesson hiding in that stack, and it is one of my core beliefs about virtualization. There are two ways to give a VM hardware: lie to it perfectly (emulate a real Intel network card, slow but compatible with anything) or tell it the truth and cooperate (special virtio drivers where the guest knows it is virtual, dramatically faster). Modern Linux ships virtio support out of the box. Whenever you see VirtIO as an option in Proxmox, take it.

Full notes on all of this in the repo, link in the comments.

Next post: VMs are not the only way to slice up a machine. Containers do it too, and picking between them wrong wastes your hardware.

#virtualization #kvm #proxmox #linux #homelab

---

## Post 5: VMs vs containers is not a war. Here is when I use each.

> **Image suggestion:** Diagram of the nesting-doll pattern: hardware > hypervisor > VM > Docker containers inside it. Or a Proxmox sidebar screenshot showing a VM and an LXC container side by side, since the different icons make the point instantly.

Every homelab conversation eventually arrives here. Someone says just use Docker, someone else says VMs are proper isolation, and beginners leave more confused than they came.

Both camps are describing real tradeoffs. Let me untangle it the way I wish someone had for me.

A VM is a full fake computer. It has virtual hardware and boots its OWN kernel. From the inside, it is indistinguishable from a real machine. That buys you the strongest isolation, and the freedom to run any OS at all. Linux host, Windows guest, no problem. The cost: every VM boots a whole operating system and reserves its own chunk of RAM.

A container shares the host's kernel. It isolates at the process level, wrapping an application in its own view of the filesystem and network while running on the kernel that already exists. That makes containers tiny and nearly instant to start. The cost: Linux only, and the isolation wall is thinner, because everyone is sharing one kernel.

Shorter version. A VM fakes the hardware. A container fakes the operating system.

So the decision is not which is better. It is which problem you have:

Need a different OS, hard isolation, or real hardware passed through to the guest? VM.
Need a lightweight Linux service without paying for a whole extra OS? Container.

Here is the part that surprised me when I got into Proxmox: it does both, natively, side by side. KVM virtual machines and LXC containers in the same interface. LXC containers are like Docker's heavier cousins, full little Linux systems sharing the host kernel, brilliant for small always-on services.

And the pattern you see all over serious homelabs combines everything: a hypervisor on the metal, a few VMs on top, and Docker containers running INSIDE one of those VMs. Nesting dolls, each layer isolating a different kind of risk. You keep Docker off the hypervisor itself, so your app experiments can never destabilize the layer everything depends on.

On my hardware this stops being theory very fast. I started with 8GB of RAM. Every guest OS I boot eats a slice of it before doing any useful work. Choosing a container where a container is enough, and spending VM overhead only where the isolation genuinely matters, is what makes a modest machine feel bigger than it is.

Next post: the platform question. Why I run Proxmox and not ESXi, XCP-ng, or plain KVM, including the honest cases for each of them.

#docker #containers #virtualization #proxmox #homelab

---

## Post 6: Why Proxmox? (An honest comparison, including when NOT to pick it)

> **Image suggestion:** Screenshot of your real Proxmox web dashboard, the summary view with CPU and RAM graphs. Blur nothing except anything sensitive; real dashboards with real uptime read as credibility.

When you decide to run a real Type 1 hypervisor at home, four names come up. I picked Proxmox VE, but I want to give you the honest landscape, because "just use what I use" teaches nothing.

VMware ESXi is the long-time enterprise standard. Rock solid, huge ecosystem, polished tooling. It is what half the data centers on earth run. But it is proprietary, the free edition has been restricted and pulled around following the Broadcom acquisition, and it is pickier about what hardware it accepts. The genuine reason to run it at home is if you are deliberately building VMware skills for your day job. That is a legitimate reason. It just was not mine.

XCP-ng is the open source continuation of XenServer, built on the Xen hypervisor and managed through Xen Orchestra. Solid, enterprise-flavoured, properly free. Smaller community than Proxmox and no built-in containers. A strong pick if you specifically prefer Xen.

Plain KVM with libvirt is the minimalist path. Skip the appliance entirely, run KVM directly on your favourite Linux distro, manage it with virsh or virt-manager. Maximum control, nothing installed that you did not choose. The catch is that YOU assemble the web UI, the backup system, the storage management, the networking. Wonderful if you want to understand every layer or automate everything from scratch. A lot of yak shaving if you just want servers.

And Proxmox VE. A Debian-based system that bundles KVM for full VMs and LXC for containers behind one clean web interface, with snapshots, scheduled backups, ZFS, and clustering already built in. Open source. There is a paid subscription for the stable enterprise repo and support, but the free no-subscription repo is fully functional. It is what my homelab runs on.

Why it won for me, concretely:

It is free with no feature gating, which matches everything else in my build.
VMs and containers in one tool means I decide per workload, not per platform.
A real web UI now, a scriptable API later, so it grows with me.
Batteries included. On 8GB of RAM and a spinning hard drive, I have no spare resources to feed extra bolted-on management software.
And the community is enormous, which matters more than people admit. Whatever breaks at 11pm, someone has already written about it.

The part I actually want you to remember, though, is this: the concepts underneath all four are identical. Type 1 hypervisors, driving the same VT-x or AMD-V hardware features, mostly over the same KVM or Xen engines. Learn the concepts on any of them and the knowledge transfers when you switch. Tools are temporary. Understanding is portable.

Quick decision guide, my honest version:
Want one box that just works, VMs plus containers? Proxmox.
Building VMware skills for employment? ESXi.
Love Xen? XCP-ng.
Want to hand-assemble everything and script it? Plain KVM.

Next post covers the practical Proxmox knowledge I actually use weekly, including the one confusion that costs people their data: snapshots are not backups.

#proxmox #vmware #homelab #selfhosting #opensource

---

## Post 7: Snapshots are not backups (and other Proxmox lessons I use every week)

> **Image suggestion:** Screenshot of the Proxmox snapshot panel next to the backup panel, or a simple two-column graphic: 'Snapshot: same disk, instant, dies with the drive' vs 'Backup: elsewhere, survives the host'. This post's idea is so quotable the graphic may get shared on its own.

These are the practical rules I actually operate by, and one of them protects your data from a mistake that catches almost everyone.

First, the rule of thumb for guests. Proxmox gives you two kinds: full KVM virtual machines and lightweight LXC containers. My rule: reach for a container when it is a simple Linux service. Reach for a VM when I need a different OS, hard isolation, or real hardware handed to the guest. And if a container has to be created privileged, I pause and reconsider, because unprivileged is the safe default. Guest root should never equal host root.

Second, the settings I never skip when creating a VM, because each earns its place. VirtIO for the disk controller and the network card, so the guest cooperates with the hypervisor instead of being lied to slowly. And the QEMU guest agent installed inside the VM, so Proxmox can shut it down cleanly and actually show me its IP address. I once stared at a blank IP field in the summary page for a while before learning that last one.

Third, and this is the big one. Snapshots and backups are different tools, and confusing them will eventually cost someone their data. Maybe you. So:

A snapshot is a point-in-time capture of a VM's state, stored on the SAME disk as the VM. It is instant, and it is perfect for the moment right before a risky change. Upgrade goes wrong, roll back in seconds.

But if that disk dies, the snapshot dies with it. A snapshot on a failing drive protects you from nothing.

A backup is a full, self-contained archive of the guest written SOMEWHERE ELSE. Another disk, a NAS, a Proxmox Backup Server. It survives the death of the host itself.

The practice that follows: snapshot before every risky change, scheduled backups for actual safety, and at least one copy living off the machine. And test a restore once in a while. An untested backup is a hope, not a backup.

This distinction is personal for me. My entire homelab currently lives on one spinning 500GB hard drive from another era. Snapshots make me brave day to day. Backups are what let me sleep.

Last thing, on networking, because it confused me early. Proxmox creates a virtual switch called vmbr0 and plugs your physical network port into it. Every VM gets a virtual cable into that same switch. The result is that your VMs appear on your home network as if they were separate physical machines, pulling IPs from your router like any laptop or phone would. Once I pictured it as a switch instead of something mystical, Proxmox networking stopped being scary.

That covers the groundwork. Hypervisors, how the trick works, VMs versus containers, why Proxmox, and how to not lose your data. The full written notes are in my repo, link in the comments.

From here, the story gets messy: what actually happened when I built this thing for real, including network surprises I never saw coming and one keystroke that cost me hours. That starts in the next post.

#proxmox #homelab #backups #sysadmin #selfhosting

---
## Post 8: The theory is done. Here is what happened when I actually built it.

> **Image suggestion:** Photo of the machine mid-setup: side panel off, or the moment of plugging in the ethernet cable. Signals 'the hands-on part starts now'.

Everything so far, hypervisors, virtualization, why Proxmox, snapshots versus backups, that was the map.

From this post on, it is the territory. The actual build, in the order it happened, with nothing cleaned up.

And I will tell you now: knowing the theory did not save me from a single one of the mistakes coming in these posts. I understood what a hypervisor was and still typed my domain name into a field asking for an IP address. I knew my server was headless and still managed to install a desktop environment on it.

Concepts tell you where you are going. Only the mistakes teach you the road.

Here is how I will write this part. When something broke, you get my wrong first theory before you get the fix, because the gap between those two is where the actual learning lives. When I chose a tool, you get the options I rejected and why, because "why not X" teaches more than "how to Y." And when a command appears, it comes with the reason it exists, never as a step to copy blind.

One piece of context that shapes everything ahead: I am on wifi, behind a shared ISP connection, with no public IP address. That one fact shaped every single decision in this build, and the next post explains exactly why.

If you have an old machine and an ordinary home internet connection, you have everything this build required.

#homelab #selfhosting #linux #devops #learninpublic
---

## Post 9: The one networking concept that decided my entire architecture: CGNAT

> **Image suggestion:** Two screenshots side by side: whatismyip.com result vs your router's WAN IP page (blur the actual numbers). The mismatch IS the CGNAT lesson in one image. Alternative: a simple diagram of many houses sharing one public IP.

Before I installed anything, I had to face a problem most self-hosting guides never mention.

The traditional way to host from home works like this: your ISP gives you a public IP address. You log into your router, forward a port (443 for a website, 22 for SSH) to your server, and anyone on the internet who connects to your-public-ip:443 reaches your server. Done.

That entire model depends on one assumption: that you own a public IP.

I do not.

My ISP uses CGNAT, Carrier-Grade NAT. Instead of giving each customer their own public IP, the ISP gives one public IP to a whole pool of customers and shares it, using its own internal NAT to sort out whose traffic is whose.

Here is what that actually means in practice:

- Port forwarding on my router does nothing. My router's "public" side is not actually public. It is just another private address, one layer up, inside my ISP's network.
- There is no port to forward, because I do not own the public IP in the first place.
- Nobody on the internet can initiate a connection to me. Ever.

This is not a configuration problem. You cannot fix it with more effort or better settings. It is structurally impossible without either a public IP or some kind of relay in front of you.

So the entire build had to be designed around one principle:

My server must always reach OUT to something. It can never wait for the world to reach IN.

Outbound connections work fine under CGNAT. Your machine visits websites all day, and that is all outbound. The trick is finding tools built entirely around outbound connections.

Two tools do exactly that, and they solve two different problems:

1. Tailscale, so I can reach my own servers from anywhere
2. Cloudflare Tunnel, so the public internet can reach my websites

We will get to both. But first, the Proxmox install, where I made my first embarrassing mistakes within the first five minutes.

If you are on mobile internet, fixed wireless, or a budget fiber plan, there is a good chance you are behind CGNAT too. Quick check: compare the IP your router reports on its WAN side to what whatismyip shows you. If they differ, welcome to the club.

#networking #CGNAT #selfhosting #homelab
---

## Post 10: My first Proxmox mistake: I did not know the difference between a hostname and an IP address

> **Image suggestion:** Photo of the actual Proxmox installer network screen on your monitor, phone camera. The hostname and IP fields visible. Authentic installer photos are rare on LinkedIn and instantly signal 'I actually did this'.

Confession time.

On the Proxmox installer's Management Network screen, there are separate fields:

- Hostname (FQDN): a name for the machine, like pve.22112002.xyz
- IP Address (CIDR): the actual numeric address, like 10.0.0.50/24

I typed my domain name into places expecting a number. In my head, "put your domain here" applied to every field on the screen.

It does not. These are two completely different concepts wearing similar clothing.

The hostname is just a label the machine calls itself. It does not need to exist in real DNS anywhere. Typing it connects nothing to the internet. You could name your server pve.i-made-this-up.fake and it would work exactly the same.

The IP address is the actual number other devices on your network use to find this machine.

Here is the rule I use now, and it applies to every setup screen you will ever see:

When a field asks for "hostname" vs "IP," stop and ask: does this field want a name a human reads, or a number a computer routes to?

If you are unsure, the format is the tell. IPs are four numbers separated by dots, plus a /number for CIDR. Hostnames are words separated by dots.

My second misconception on the same screen: I thought I needed internet access during the install to "get" an IP address.

Wrong mental model entirely. A static IP is not handed to you by some authority. You pick an unused number inside your own network's range and claim it. That is it. No internet connection is needed to type a number into a form.

How did I find the right numbers? I ran two commands on a computer already connected to the same network:

```
ip a    # shows the network range in use
ip r    # shows the gateway (the router)
```

That told me the network was 10.0.0.x with the gateway at 10.0.0.1. I picked 10.0.0.50/24 for Proxmox. Same first three numbers as everything else, an unused last number, outside the range the router hands out automatically.

Internet access only starts mattering AFTER the install, when you want to run apt update or reach the outside world from the box.

Small concepts. But get them wrong and nothing downstream makes sense.

#proxmox #linux #homelab #networking
---

## Post 11: Why your homelab server cannot run on wifi (and mine almost did)

> **Image suggestion:** Photo of the ethernet cable going into the server's NIC, close up. Simple, physical, matches the 'sometimes the fix is walking over with a cable' lesson.

I have wifi available. The server has a wifi card. So during the Proxmox install, I briefly considered just using wifi for the host.

This does not work, and the reason taught me something about how virtualization actually functions.

A hypervisor "bridges" the network to its virtual machines. Each VM needs to send traffic that looks like it is coming from its own separate device on the network, with its own identity, its own MAC address.

Wifi hardware and most access points reject this behavior. A wifi card is built to be ONE device with ONE identity on the wireless network. It is not designed to impersonate a whole crowd of virtual machines. Ethernet has no such restriction.

So the rule is simple: the physical hypervisor box needs a wired ethernet connection, full stop.

Important nuance: this only applies to the hypervisor. My laptop, which merely accesses the server through a browser and SSH, stays on wifi with zero issues. The restriction is specifically about the box that bridges traffic to guests.

This led directly to my next confusion. During install, the IP field showed "DHCP" instead of a number, and I assumed something was broken.

Nothing was broken. No ethernet cable was plugged in yet, so the installer had nothing to detect and displayed a placeholder. The fix was physically walking over and moving a cable from another computer to the Proxmox machine. Sometimes the fix is not in software.

Here is what I actually entered, and why:

| Field | Value | Why |
|---|---|---|
| Management interface | the ethernet NIC | wifi cannot bridge to VMs |
| Hostname (FQDN) | pve.22112002.xyz | just a label, reused my domain for tidiness |
| IP (CIDR) | 10.0.0.50/24 | static, outside DHCP range, matches my LAN |
| Gateway | 10.0.0.1 | my router |
| DNS | 1.1.1.1 | this choice came back to bite me later, more on that in a future post |

And right after install, one housekeeping step every Proxmox homelab needs:

```
rm /etc/apt/sources.list.d/pve-enterprise.list
rm /etc/apt/sources.list.d/ceph.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list
apt update && apt full-upgrade -y
```

This removes the paid subscription repos (which you do not have access to on a free homelab) and switches to the free no-subscription repo, which is fully functional.

Next post: the mental shift that finally made Proxmox click for me. It involves realizing your server is not a computer anymore.

#proxmox #virtualization #homelab #networking
---

## Post 12: "Why can't I use this PC anymore?" The mental shift that made Proxmox click

> **Image suggestion:** Photo of the black Proxmox console screen showing just the login prompt and the management URL. This IS the 'where is the desktop?' moment. One of the best images of the whole series.

I touched on this moment back in the Type 1 vs Type 2 post, but it deserves its own telling, because it is the adjustment that makes or breaks your first week.

After installing Proxmox, I sat in front of the machine and felt genuinely confused.

No desktop. No browser. No apps. Just a black console screen with a login prompt and a URL.

I actually asked, out loud: why can't I use this PC anymore?

The answer required reframing what the machine now IS.

Proxmox is not an operating system you use directly. It is an appliance, like a router.

Think about your router for a second. You do not sit at it typing into it every day. You configured it once, and now it just runs in a corner doing its job. You manage it remotely, from a browser, on the rare occasion you need to change something.

That is exactly the relationship you have with a hypervisor. Everything happens from my laptop now, in a browser, at:

```
https://10.0.0.50:8006
```

The physical screen and keyboard on the box are purely an emergency fallback for the day networking breaks badly enough that I cannot reach it remotely.

Once that clicked, everything else about server administration made more sense too:

- The machine is headless. It does not need a GUI because no human sits at it.
- Every GB of RAM spent on a desktop environment is a GB stolen from the VMs, which are the entire point.
- "Using" the server means SSH and web interfaces, always, from somewhere else.

This sounds obvious written down. It was not obvious sitting in front of a black screen wondering if I had broken my computer.

If you are coming from a lifetime of desktop computing, this is the single biggest adjustment. The computer stops being a place you go and becomes a service you consume.

Hold onto this idea, because in the next post I make the exact mistake this mindset should have prevented: I accidentally installed a full desktop environment, GNOME and all, onto a headless VM. It cost me more time than any other mistake in this entire project, and it happened because of one keystroke.

#proxmox #homelab #linux #sysadmin
---

## Post 13: The VM settings that actually matter (and why each one exists)

> **Image suggestion:** Screenshot of the Proxmox Create VM wizard with the four settings visible (guest agent tick, SCSI + discard, CPU host, VirtIO). Circle or arrow the four in your accent color.

Time to create the first virtual machine.

Quick detour first: I originally wanted "CentOS" because that is the name I knew for lightweight servers. Turns out classic CentOS is discontinued. Its real modern equivalents are Rocky Linux and AlmaLinux. I ended up choosing Debian instead: lighter on RAM at idle, and it is literally the same family Proxmox itself is built on.

I mentioned VirtIO and the guest agent back in the concept posts. This is where they stopped being theory. The Proxmox Create VM wizard has a lot of tabs and a lot of settings. Most defaults are fine. Four settings genuinely matter, and I want to explain WHY each one exists, because "just tick this" teaches nothing:

1. Qemu Guest Agent: tick it.
This is a small helper program that runs inside the VM and reports information back to Proxmox, like the VM's IP address. Without it, the Proxmox summary page cannot show you the VM's IP at all. I learned this the confusing way, staring at a blank IP field wondering what I broke.

2. Disk bus: SCSI, with Discard ticked.
Discard means that when you delete files inside the VM, the underlying disk space actually gets reclaimed by Proxmox. Without it, the virtual disk file only ever grows, even as the VM's own filesystem frees space. On a 500GB spinning drive, I cannot afford disk files that grow forever.

3. CPU Type: host.
This passes the real CPU's actual feature set through to the VM instead of emulating a generic processor. Best possible performance match, at the cost of portability between different physical hosts. For a homelab with one host, portability does not matter.

4. Network model: VirtIO.
A virtualization-aware network driver, dramatically faster than the default emulated hardware. The default exists for compatibility with ancient operating systems. Debian is not ancient.

The pattern behind all four: virtualization has two modes. Pretend to be real hardware (slow, compatible with everything) or admit to the guest that it is virtual and cooperate (fast, needs guest support). Modern Linux supports cooperation everywhere. Always choose cooperation.

None of these settings caused me problems. The Debian installer inside the VM, on the other hand, was about to hand me the most expensive mistake of the whole build. That is the next post.

#proxmox #virtualization #debian #homelab
---

## Post 14: One keystroke cost me hours: the Enter key is not always "confirm"

> **Image suggestion:** Screenshot or photo of the Debian software selection screen with GNOME ticked, or the black 'Booting from Hard Disk...' screen. The second one is funnier and pairs perfectly with the story.

This was the single most time-costly mistake of the entire project.

I was in the Debian installer, inside my new VM, on the Software Selection screen. A list of checkboxes: desktop environments, web server, SSH server, standard utilities.

I pressed Enter, assuming it would confirm my choices and move me forward.

It did not ask me anything. It accepted whatever was pre-ticked, which included a full GNOME desktop environment, and started installing it.

On a headless server. That has no monitor. That I will only ever access through SSH.

Why this hurt:

- A full desktop wastes a large amount of disk and RAM I explicitly do not have to spare (8GB total, remember)
- I was now sitting through a much longer install on a slow spinning hard drive, for software I would never see

The mechanics of that screen, which I did not know at the time:

- Arrow keys move between options
- SPACE toggles a checkbox on or off
- Enter/Tab only matter once you have navigated to the Continue button itself

The correct end state for a server has exactly two boxes ticked: SSH server, and standard system utilities. Every desktop environment entry unticked.

Then I made it worse.

When I realized GNOME was installing, I stopped the VM mid-install to start over cleanly. That left the virtual disk in a half-written state: no working bootloader, an incomplete filesystem.

So on the next boot: a black screen. "Booting from Hard Disk..." Forever.

I now know two different causes produce that exact symptom, and I eventually hit both:

1. The install genuinely never finished (my case). There is no bootloader on the disk, so "booting from hard disk" has nothing to boot.
2. The boot order in Proxmox has the CD-ROM below the hard disk, so the machine tries the empty disk first.

The fix, in order: stop the VM properly. Check Options, then Boot Order, and make sure the CD-ROM/ISO is enabled and ABOVE the hard disk. Start again. Run the installer all the way to a genuine "Installation complete" message. And critically, say YES when it asks to install GRUB to /dev/sda.

GRUB is the bootloader, the actual software that starts the operating system when the machine powers on. Skip it and you have an OS on disk with nothing capable of starting it. A car with no ignition.

Once GRUB was in and the ISO ejected (Hardware, CD/DVD Drive, "Do not use any media"), the VM booted straight to a login prompt like a real server should.

Lessons, plural:

1. Read installer screens. Enter is not universally "next."
2. Never kill an OS install partway. Let it finish, then wipe and redo if needed.
3. When a machine "boots from hard disk" into nothing, ask two questions: is there actually a bootloader on that disk, and is the boot order even pointing where I think it is?

#linux #debian #homelab #sysadmin #learninpublic
---

## Post 15: sudo: command not found. The minimal install strips more than you expect.

> **Image suggestion:** Terminal screenshot of the real error: bash: sudo: command not found. Crop tight, dark terminal. Error screenshots are the most engaging developer content on LinkedIn.

Fresh into my newly working Debian VM, feeling good, I typed my first admin command:

```
srv1@srv1:~$ sudo apt install -y qemu-guest-agent
-bash: sudo: command not found
```

Wait. What?

Here is what I did not know: Debian's minimal install does not include sudo at all if you set a root password during setup. The installer assumes that if you gave root a password, you intend to use su to become root when needed.

This creates a genuine chicken-and-egg problem. You cannot "sudo install sudo." There is no sudo yet to do the installing.

The fix requires a specific order:

```
su -                      # become root directly, using the ROOT password
apt update
apt install -y sudo curl
usermod -aG sudo srv1     # add my normal user to the sudo group
exit
```

(Yes, curl was missing too. Minimal means minimal.)

Then I made a new mistake ON TOP of the fix.

I pasted several of those lines together in one go, including the su - command AND the commands meant to run only after su - finished asking for the password.

The terminal does not wait patiently for a multi-line paste to resolve one prompt at a time. It fired the later commands before root access was established, so they ran as my normal unprivileged user and failed:

```
Error: Could not open lock file /var/lib/apt/lists/lock - open (13: Permission denied)
```

The lesson: when a command is going to prompt for input (a password, a yes/no confirmation), never paste it bundled with the commands meant to run after that prompt resolves. Run the prompting command by itself. Wait. Then continue.

Bonus embarrassment from the same session: I once pasted a comment line, literally "# root password", into the terminal as if it were a command. Lines starting with # in a guide are notes to the reader, not input.

While we are here, the sudo vs su distinction, properly:

sudo <command>: you stay logged in as your everyday low-privilege user and borrow admin rights for that ONE command. A typo only affects that one line. This is the pattern for 99% of daily work.

su -: you fully BECOME root, using root's own password. Your prompt changes from $ to # as a visible warning. Every command now runs with full system power until you type exit.

The rule I settled on: if my prompt shows #, finish the specific task and exit immediately. I have used su - exactly once on purpose, for this exact bootstrap situation, where sudo did not exist yet to install itself.

#linux #debian #sysadmin #homelab
---

## Post 16: "Permission denied" was lying to me. The bug was two letters swapped.

> **Image suggestion:** Terminal screenshot of the SSH permission denied, with svr1 vs srv1 circled or highlighted. Let people spot the typo themselves before reading, it makes the post interactive.

I set up SSH to my new VM. Typed my password carefully. Got this:

```
ssh svr1@100.72.72.58
...
Permission denied (publickey,password)
```

Tried again. Same password, typed slower. Denied.

I started doubting the password. Started wondering if SSH was misconfigured. Started composing theories about key authentication settings.

Look closely at that command.

My username was srv1. S, R, V. As in "server 1."

I was typing svr1. S, V, R.

Two letters, transposed. An account that simply does not exist by that spelling. No password on earth would ever work.

Here is why this mistake is worth a whole post: the error message actively misleads you.

"Permission denied" sounds like a credentials problem. Wrong password, wrong key, wrong auth settings. So that is where your brain goes, and that is where you burn your time.

But SSH deliberately will not tell you "that user does not exist." That would let attackers probe a server to discover which usernames are valid. So a nonexistent user and a wrong password produce the identical error, on purpose. It is a security feature that doubles as a debugging trap.

The rule I extracted:

When "permission denied" happens right after typing a password you are SURE is correct, check the username character by character before assuming anything about the password.

More generally: the error message tells you what the system observed, not what you did wrong. "Permission denied" means "authentication failed," and a typo'd username is one of several ways to fail authentication. The message is accurate. My interpretation of it was not.

Cheap mistakes like this one build the most durable habits. I now read usernames letter by letter before I ever suspect a password. Total cost of the habit: three seconds. Total cost of not having it: however long you spend investigating an SSH configuration that was fine all along.

Next post: the big architectural question. I can reach my server from my couch. How do I reach it from anywhere in the world, with no public IP? And why I, a die-hard self-hoster, chose NOT to self-host the answer.

#ssh #linux #debugging #homelab
---

## Post 17: Why I did not self-host WireGuard (from someone who self-hosts everything)

> **Image suggestion:** Diagram: two devices behind CGNAT walls with a rendezvous point (coordination server) above introducing them, then a direct encrypted line between them. This is the hardest concept in the series to grasp from text alone, the image does real work here.

Anyone who knows my projects knows my bias: free and self-hosted over paid and managed, every time. I self-host Qdrant. I use Docker Compose over managed platforms. It is a consistent principle, not an aesthetic.

So when I needed remote access to my homelab, my first instinct was obvious: self-host WireGuard. It is the gold standard VPN protocol. Fast, modern, minimal.

I could not. And understanding exactly WHY taught me more about networking than the setup itself did.

Plain self-hosted WireGuard requires one side of the connection to be reachable at a known address. Normally that is the server: you forward a UDP port on your router, and clients out in the world dial in to your public IP and that port.

Remember the CGNAT post at the start of the build? I am behind CGNAT. I do not have a public IP. There is no address to forward a port on, because the "public-facing" side of my router is not actually public. It is another private address inside my ISP's shared pool.

Self-hosting the WireGuard server does not fail because of a configuration mistake I could fix with more effort. It fails because there is no reachable address to put in the configuration in the first place.

That distinction matters. It is not a skill gap. It is a missing prerequisite.

The only way around it is putting something with a real public IP in front of the connection, acting as a rendezvous point. Two options:

1. Rent a cheap VPS with a public IP and run Headscale on it (a self-hosted alternative to Tailscale's coordination server)
2. Use Tailscale's own free coordination service, which is exactly that rendezvous point, already running, at zero cost

Here is the part that let me make peace with option 2:

Tailscale IS WireGuard underneath. Same protocol, same modern cryptography. It is not a different, weaker alternative. It is WireGuard plus a coordination layer that solves the exact problem CGNAT creates: devices with no public address finding each other.

And after the coordination server introduces the devices, my actual traffic flows directly between them, end-to-end encrypted, peer to peer. Tailscale's servers broker the introduction and occasionally relay traffic through genuinely hostile networks. They are not sitting in the middle of my data.

So the honest reasoning: the one piece that cannot be self-hosted here is the public rendezvous point, because self-hosting it requires renting a machine with a public IP, which already defeats a chunk of the "everything on my own hardware" goal. Tailscale's free tier does the identical job at zero cost and zero additional hardware.

Principles are good. Understanding when a principle structurally cannot apply is better.

#wireguard #tailscale #networking #selfhosting #CGNAT
---

## Post 18: Tailscale in practice: what "device accepted" actually does under the hood

> **Image suggestion:** Screenshot of tailscale status output showing your three devices with their 100.x addresses (blur if you prefer). Three lines of terminal output that prove the whole concept works.

The setup itself is almost anticlimactic. On every device that should be part of the mesh:

```
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

I ran this on three machines: the Proxmox host, the VM (srv1), and my laptop.

Do not skip the laptop. A mesh containing only servers and no client is useless to you personally. You need at least one device you actually sit at inside the network, otherwise you have built a private club with no members.

Each tailscale up prints a URL. You open it, log in, and the device is "accepted."

Two things about that step that are worth actually understanding:

First, use the SAME account on every single device. Different accounts create separate, non-connected tailnets. Your devices will be online, healthy, and completely unable to see each other.

Second, here is what "accepted" really does under the hood, because it is not magic:

The coordination server now knows this device's cryptographic identity belongs to you. It assigns the device a permanent 100.x.x.x address that never changes, no matter what physical network the device is on. Home wifi, mobile hotspot, a cafe in another country: same address. Then it tells every other device in your tailnet how to reach the new one directly.

After that introduction, the devices talk peer-to-peer, encrypted, without needing the coordination server for the actual data.

Check the roster:

```
tailscale status
```

Three lines, one per device, each with its own 100.x address. I could now SSH into my VM from anywhere on the planet.

Two quality-of-life notes:

I turned on MagicDNS in the Tailscale admin console, which lets me use names instead of memorizing numbers. ssh srv1@srv1 instead of ssh srv1@100.72.72.58. (And after the username typo story, you know exactly why I prefer typing fewer characters that can be transposed.)

I also assumed the free tier's device limit was tiny, something like 3, based on nothing except how many machines I happened to have. It is actually 100 devices on the free personal plan. My three machines use a rounding error of that. Check the actual limits before you assume a paywall is coming.

Next up: the gremlins. Four networking problems that each looked like one thing and turned out to be another. The first one made my brand new server unable to resolve a single domain name.

#tailscale #networking #homelab #remoteaccess
---

## Post 19: Gremlin 1: my DNS queries were quietly going nowhere

> **Image suggestion:** Split terminal screenshot: ping 1.1.1.1 succeeding on top, curl failing to resolve a hostname below. The visual split IS the diagnostic lesson.

My fresh VM could not download anything:

```
curl: (6) Could not resolve host: tailscale.com
```

But watch this:

```
ping -c 2 1.1.1.1    →   worked perfectly
```

Stop and appreciate what that split result proves, because this is a debugging pattern you will use for the rest of your career.

Raw IP traffic works. Pinging a number succeeds. So the internet connection itself is fine: cables, router, ISP link, all healthy.

But turning a NAME into a number fails. That is one specific subsystem: DNS resolution.

The split instantly narrows the entire universe of possible problems down to one layer. Not "the internet is broken." Specifically: name resolution is broken.

During the Proxmox install I had set DNS to 1.1.1.1, Cloudflare's public resolver. A completely standard, normally excellent choice. Millions of setups use it.

The actual cause: on my network path, direct queries to public DNS resolvers were not getting through. Many consumer and mobile networks are set up so DNS flows through the provider's own resolvers, and direct queries to outside resolvers quietly go nowhere. No error. No notice. Queries to 1.1.1.1 just die.

The fix was pointing DNS at my own router, which forwards queries along the path that actually works:

```
echo "nameserver 10.0.0.1" > /etc/resolv.conf
```

Names resolved immediately.

The transferable rule, worth memorizing as a reflex:

If pinging a raw IP works but resolving a name does not, the problem is DNS. Not "the internet." Not the firewall. Not the cable. DNS.

The two-command test costs ten seconds:

```
ping -c 2 1.1.1.1        # tests raw connectivity
ping -c 2 google.com     # tests name resolution
```

First works, second fails: DNS problem. Both fail: connectivity problem. Both work: your problem is somewhere else entirely.

And a quieter lesson underneath: "best practice" configurations assume a network path that behaves like the textbook. Mine does not, and plenty of real-world connections do not either. Sometimes the textbook answer (public DNS resolver) loses to the boring answer (just use the router) because of realities between you and the internet that no tutorial can know about.

Keep that thought. This same network path has one more surprise coming in Gremlin 4.

#dns #networking #debugging #linux #homelab
---

## Post 20: Gremlin 2: the "invalid SSL certificate" that was actually a wrong clock

> **Image suggestion:** Screenshot of timedatectl showing the wrong date, or a graphic of a certificate validity window with a 'you are here' marker sitting before the start date.

Next gremlin. Trying to download something over HTTPS:

```
curl: (60) SSL certificate problem: certificate is not yet valid
```

"Certificate not valid" reads like a security problem. Broken cert, misconfigured server, maybe something suspicious happening. That is where your head goes.

The actual cause was almost silly: my VM's system clock was wrong. Stuck months in the past. The VM had inherited a stale clock at boot and never synced it.

Here is why a wrong clock produces a CERTIFICATE error, because the mechanism is worth understanding:

Every HTTPS certificate carries a validity window: a start date and an end date. When your machine checks a certificate, it compares that window against its own idea of "now."

My VM believed "now" was months ago, BEFORE the certificate's start date. So from the machine's point of view, the certificate legitimately was not valid yet. The certificate was fine. Reality was fine. My clock was lying about what day it was.

The check and the fixes:

```
timedatectl                    # check what the machine thinks the time is
timedatectl set-ntp true       # auto-correct via internet time servers
date -s "2026-08-03 21:30:00"  # manual fallback if NTP itself is blocked
```

(Note the fallback. After Gremlin 1, I trust nothing on my network path to "just work.")

The lesson, and it is a great one for your general debugging toolkit:

Certificate errors are not always about certificates. A wrong system clock produces "not yet valid" (clock in the past) or "expired" (clock in the future) errors on perfectly healthy certificates.

Checking the clock takes five seconds and rules out an entire category of confusing failures. It should be one of your first checks whenever TLS/SSL misbehaves in weird, sweeping ways, especially on machines that have been powered off, are freshly created, or are virtual.

VMs specifically are prone to this: they can boot with whatever stale time their virtual hardware hands them, and if NTP has no way through, the clock stays wrong forever until someone looks.

Two gremlins down. The next one was not the server's fault at all. It was my own laptop sabotaging me, and the clue was hiding in plain sight inside the error message.

#ssl #tls #debugging #linux #sysadmin
---

## Post 21: Gremlin 3: my own VPN was eating my traffic (the clue was in the error)

> **Image suggestion:** Terminal screenshot of the ping output with the reply coming from 10.8.0.1, that address circled. Caption idea: 'the answer was in the first line the whole time'.

This one had me confused until I actually READ the error instead of skimming it.

From my laptop, I tried to ping my Proxmox server on the local network:

```
ping -c 3 10.0.0.50
From 10.8.0.1 icmp_seq=1 Destination Host Unreachable
```

Destination Host Unreachable. Okay, server down? Wrong IP? Cable out?

No. Look at where the reply came FROM: 10.8.0.1.

That is not my router. My router is 10.0.0.1. So who is 10.8.0.1?

It is the gateway of my OTHER VPN. I run a separate, pre-existing WireGuard connection (wg0) on my laptop for unrelated work purposes. Its gateway lives at 10.8.0.1.

Which means: my ping to 10.0.0.50 never touched my home network at all. My laptop's VPN configuration was intercepting traffic destined for 10.0.0.x addresses and routing it into the VPN tunnel, where it found nothing.

The likely cause: that VPN's AllowedIPs rules are written very broadly, capturing more address ranges than intended, including my home LAN's range.

The quick fix for testing:

```
sudo wg-quick down wg0    # bring the VPN down
# ... do the local work ...
sudo wg-quick up wg0      # bring it back
```

With wg0 down, 10.0.0.50 was instantly reachable. Mystery closed.

One clarification that mattered later: once Tailscale was fully set up, both VPNs ran simultaneously with no conflict. The clash was specifically wg0 versus my plain LAN addresses (both fighting over 10.x space), not wg0 versus Tailscale, which lives in its own separate 100.x range. Address ranges that do not overlap do not fight.

The transferable lesson, and it is a sharp one:

When a device that should obviously be reachable returns "Destination Host Unreachable" from a completely UNEXPECTED IP, that unexpected IP is the whole answer. It is telling you exactly which piece of software wrongly intercepted your traffic.

Error messages are full of details we skim past. The source address of a failed ping feels like noise. It was the entire diagnosis, printed right there, in the first line of output.

Read your errors. All of them. Slowly.

#wireguard #vpn #networking #debugging #homelab
---

## Post 22: Gremlin 4: my tunnel would not connect, and the fix was one config line

> **Image suggestion:** Screenshot of the cloudflared QUIC error log lines, then the one-line protocol: http2 fix below it. Problem and fix in one image.

Last gremlin, and it brings us back to the same theme as Gremlin 1: my network path has opinions.

Setting up Cloudflare Tunnel (full post on it next), the cloudflared service got stuck in an endless loop:

```
ERR Failed to dial a quic connection error="failed to dial to edge with quic: timeout: no recent network activity"
INF Retrying connection in up to 2s
```

Connect, fail, retry. Forever. Meanwhile my regular internet browsing worked completely normally.

The key word in that error is quic.

QUIC is a modern transport protocol that cloudflared uses by default. It is fast, it is the future, and it runs over UDP instead of TCP.

Remember Gremlin 1, where DNS queries to public resolvers quietly went nowhere? Same energy here: on this connection, this kind of UDP traffic does not get through. Regular TCP browsing flows fine. Less common outbound UDP dies quietly, no error, no notice, just timeouts.

The fix is one line in the tunnel's config file, forcing a TCP-based protocol instead:

```
protocol: http2
```

After that change and reinstalling the service, the tunnel connected immediately and stayed happily in active (running) state.

Now zoom out, because Gremlins 1 and 4 together form a pattern worth keeping:

On this connection, completely normal traffic flows fine (web browsing over TCP, DNS through the default resolvers), while less common traffic patterns do not (direct queries to public DNS resolvers, UDP-based QUIC). This is normal on consumer and mobile-style connections worldwide, and if you are behind CGNAT, chances are decent your connection behaves the same way.

So here is the troubleshooting rule I extracted:

When something that "should just work" mysteriously times out or hangs, while everything else is fine, suspect the network path and try falling back to the most conventional, boring protocol available. Regular DNS via the router instead of a public resolver. HTTP/2 over TCP instead of QUIC over UDP.

The fancy option fails silently. The boring option gets through, because no network can break the boring option without breaking the internet for everyone.

Four gremlins down. Next post is the payoff: how a machine with no public IP, behind CGNAT, on a completely ordinary home connection, ended up serving a real website to the entire public internet.

#cloudflare #networking #debugging #homelab
---

## Post 23: My domain is live from my house, with no public IP. Here is how Cloudflare Tunnel works.

> **Image suggestion:** THE money shot: photo of your phone, on mobile data (signal indicator visible), displaying 22112002.xyz live. If you only invest effort in one image in this whole series, this is the one. Consider it for the series' pinned/cover post too.

This is the moment the whole build was pointing at.

The problem, one more time: Tailscale gets ME into my own network from anywhere. But a website needs the opposite: it needs the PUBLIC INTERNET to reach a specific service on my network. And I still have no public IP for anyone to connect to.

Cloudflare Tunnel solves the exact mirror image of Tailscale's problem, using the same core trick: outbound-only connections.

How it actually works, mechanically:

cloudflared, a small service running on my VM, opens an OUTBOUND connection to Cloudflare and holds it open persistently. When someone on the internet requests 22112002.xyz, Cloudflare's edge receives that request (they own real public IPs; I do not need to) and forwards it DOWN the already-open tunnel to my VM.

My side never accepts an inbound connection. From the network's perspective, my VM just made a normal outgoing connection, the same as visiting any website. CGNAT has no problem with that whatsoever.

The setup:

```
# Install
curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared

# Authorize against the domain (prints a URL; fine to open on your phone, the VM has no browser)
cloudflared tunnel login

# Create the tunnel, note the UUID it prints
cloudflared tunnel create homelab

# Point the domain at it. This auto-creates a CNAME record.
# No A/AAAA record needed, or even possible: I have no public IP to put in one.
cloudflared tunnel route dns homelab 22112002.xyz
```

The config file (including the protocol fix from Gremlin 4):

```
tunnel: <tunnel-uuid>
credentials-file: /home/srv1/.cloudflared/<tunnel-uuid>.json
protocol: http2
ingress:
  - hostname: 22112002.xyz
    service: http://localhost:8080
  - service: http_status:404
```

That last catch-all line is required. Without it, any hostname not explicitly listed gets rejected outright.

Then install it as a persistent service so it survives reboots:

```
sudo cloudflared service install
sudo systemctl status cloudflared
```

Now, the proof. I stood up a trivial test page:

```
mkdir ~/www && echo "<h1>22112002.xyz is live</h1>" > ~/www/index.html
cd ~/www && python3 -m http.server 8080
```

And here is the detail that matters: I tested it from my phone ON MOBILE DATA, deliberately NOT on my home wifi.

Why? Testing from home wifi only proves the site works on my own LAN, which was never in question. Testing from mobile data proves a genuine stranger, anywhere on the internet, on a completely different network, can load my page. That is the actual goal, so that is what the test has to demonstrate. Test the claim you actually care about.

The page loaded. A domain, resolving through public DNS, served from a repurposed PC in my house, through CGNAT, with zero port forwarding.

One error worth knowing before you hit it: if you ever see "Cloudflare Tunnel error / Error 1033," it means the tunnel and DNS routing are set up CORRECTLY, but nothing is currently listening on the local port your config points to (or the tunnel service is not running). The pipe is connected; nothing is plugged into your end of it. It is not a DNS problem and not a tunnel-setup problem, which is an important distinction when deciding what to actually fix.

The build works. But one test page on one port is not a hosting setup. Next posts: hosting MULTIPLE sites behind this single tunnel, the tools I rejected along the way (including a hard look at cPanel and nginx), and the best debugging story of the whole project.

#cloudflare #selfhosting #webhosting #homelab #CGNAT
---

## Post 24: One tunnel, many websites: why I put a reverse proxy in front of everything

> **Image suggestion:** Diagram in your style: Internet > Cloudflare > tunnel > ONE reverse proxy > branching to Site A / Site B / Site C. Reuse this exact diagram again in the finale so it becomes recognizable.

The homelab was live with one test page on port 8080. But the actual plan is bigger: my portfolio now, client sites later, maybe a blog subdomain, all under 22112002.xyz and possibly other domains entirely.

The naive approach would be to give the tunnel config one hostname-to-port line per site, and run a separate little webserver process on a separate port for each:

```
ingress:
  - hostname: 22112002.xyz
    service: http://localhost:8080
  - hostname: blog.22112002.xyz
    service: http://localhost:8081
  - hostname: clientsite.com
    service: http://localhost:8082
```

This technically works for a small number of sites. I rejected it anyway, and the reasons are worth spelling out:

1. Every new site means editing the tunnel config and restarting cloudflared, a service that is supposed to be my stable, always-on public entry point. I do not want to be restarting my ONLY path to the internet every time I add a page.

2. Every site needs its own dedicated process on its own port, tracked manually. Which port is free? Which did I already use? That is bookkeeping waiting to fail.

3. No shared behavior. If I ever want gzip, access logging, or consistent security headers, I would configure them separately, per site, per process.

4. It does not scale the way I actually want to scale. "Add a client's site" should be a small, boring, repeatable action, not surgery on core public-facing infrastructure.

The standard, correct pattern: put ONE reverse proxy in front of everything, and let the tunnel talk only to that.

```
Internet → Cloudflare → Tunnel → [ONE reverse proxy] → routes by hostname → Site A / Site B / Site C
```

Now the tunnel's job is permanently simple and never changes: forward everything to the reverse proxy, always on the same port. The reverse proxy is the only thing that gets touched when a site is added, and touching it does not go near the tunnel or the public-facing plumbing.

A reverse proxy, if the term is new: a program that sits between incoming requests and one or more backend services, deciding which backend handles each request, usually by looking at which hostname was asked for. The outside world only ever needs to know about one address.

Notice this is the same design idea as everything else in this project, one layer up: do not let each new piece require touching the fragile, hard-to-debug layer underneath it. Isolate change to the layer where mistakes are cheap and safe.

The next question was which reverse proxy. And that decision involved rejecting the most famous name in web hosting, plus the most famous name in web servers. Reasons in the next post.

#webhosting #reverseproxy #architecture #homelab
---

## Post 25: Why I said no to cPanel (and yes, I did the math)

> **Image suggestion:** Simple comparison graphic: cPanel column (license $15-45/mo, 1-2GB RAM, CentOS-family only) vs your stack column (free, ~50MB, Debian). Numbers in big type. Cost graphics travel far on LinkedIn.

When people think "hosting multiple websites," one name dominates: cPanel. It is what shared hosting providers use, so it is the default mental model most of us inherit for "managing websites on a server."

I looked at it properly instead of dismissing it, and here is exactly why it was wrong for this build. I think "why not X" is often more educational than "how to do Y," because it forces you to understand tradeoffs instead of memorizing one path.

First, a correction to my own assumption: cPanel is not a Linux thing that comes with servers. It is a paid, licensed commercial product that runs ON TOP of Linux.

The four disqualifiers, concretely:

1. Cost. Licensing runs somewhere in the range of $15 to 45+ per month depending on account limits, charged to whoever operates the server. Everything else in this build is free: Tailscale free tier, Cloudflare's free tunnel, free OS, free reverse proxy. A monthly license for a control panel would contradict the entire approach.

2. RAM. cPanel's own recommended minimum is around 1GB just for the panel, more realistically 2GB+, before any actual site traffic. I am budgeting 8GB (16 after an upgrade) across a hypervisor, VMs, and future services like n8n. Spending a quarter of my scarcest resource on a management dashboard, rather than on anything my sites actually need, is a bad trade.

3. OS lock-in. cPanel only installs on the CentOS/AlmaLinux/CloudLinux family. My VM is Debian. Using cPanel would mean rebuilding the VM on a different OS, or running a second VM just for the panel. More overhead, more moving parts, for a GUI wrapped around functionality a six-line text file gives me for free.

4. Problem mismatch, the biggest one. cPanel earns its cost when you are reselling hosting to many non-technical clients who each need isolated logins, email inboxes, and zero command-line contact. I am a developer hand-deploying a small number of sites for myself and maybe people I know. cPanel solves a problem I do not have, and charges rent for it.

One honest note for balance: "no GUI, forever" is not actually my constraint. "No paid license, no OS lock-in, minimal RAM" is. If I ever want a dashboard, free self-hosted options exist that pass those tests, CyberPanel being the closest match (open source, runs on Debian/Ubuntu). The point is knowing your REAL constraint, because it rules out cPanel specifically while leaving the door open to lighter tools later.

Next post: the more controversial rejection. I did not pick nginx, and I want to be honest about why, because the reason is about me, not about nginx.

#cpanel #webhosting #selfhosting #homelab
---

## Post 26: I did not pick nginx, and the reason is about me, not nginx

> **Image suggestion:** Side-by-side screenshot: the 4-line Caddyfile next to the 12-line nginx config, same site. No caption needed, the size difference is the argument.

This is the decision most likely to get me comments, so let me be properly honest about it.

nginx is a completely legitimate, extremely widely used choice. It is also, frankly, the more "expected" skill on a CV. I chose Caddy instead, and I want to lay out the real reasoning rather than pretend nginx is objectively bad.

What pushed me away from nginx for THIS project:

1. Verbose, easy-to-break config. Every site needs a server block, explicit listen and server_name directives, a location block with try_files logic just to serve static files correctly, and every line needs a trailing semicolon. Miss one, and the config can fail to load, sometimes with an error that does not point clearly at the mistake.

2. No built-in automatic HTTPS. Certificates are normally bolted on with Certbot plus a renewal job to remember. In my setup this barely matters (Cloudflare Tunnel handles public HTTPS), but it is one more moving part nginx does not handle natively.

3. Manual reload discipline. You are expected to run nginx -t before reloading, as a separate step you must remember. Skip it and a bad edit can take the whole server down on reload.

Now the honest part. Look back at this series. A mistyped username. Enter instead of Space on an installer screen. Pasting commands over a password prompt. A comment pasted as a command.

I know my own failure mode by now: small, avoidable syntax-level mistakes. So I made a deliberate choice to reduce the NUMBER OF PLACES I could make one, rather than optimize for the most commonly expected CV keyword.

Here is the size difference for the exact task I needed, serving one static site matched by domain name.

Caddy, the whole thing:

```
22112002.xyz {
    root * /var/www/22112002.xyz
    file_server
}
```

The nginx equivalent:

```
server {
    listen 80;
    server_name 22112002.xyz;
    root /var/www/22112002.xyz;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Same result. More lines, more punctuation, more places a small typo silently breaks something.

Why Caddy fits this setup beyond brevity:

- Adding a site is one more short block. No sites-available/sites-enabled symlink dance.
- caddy validate checks config syntax BEFORE applying it. (This turns out to matter enormously in the next post.)
- Low RAM footprint on hardware I am budgeting carefully.
- Automatic HTTPS is Caddy's famous feature, but in my case it is beside the point, since the tunnel handles public HTTPS. Caddy earns its place purely on how much less there is to write and get wrong.

Quick word on two other options I considered and skipped: Traefik shines when you have many Docker containers to auto-discover and route via labels. I am hosting static sites directly on a VM, not orchestrating containers, so its core strength does not apply. Apache is the most battle-tested option alive, but heavier than needed for pure static hosting, with no strong reason to reach for it here.

The tradeoff I am knowingly accepting: nginx experience appears in more job listings. Caddy is production-grade but less universally "expected." I prioritized getting this right without a debugging marathon over matching the most common industry keyword. A reasonable call for a personal homelab, and one I would revisit the day a job or client specifically needs nginx.

Know your tools. But know your own failure modes first.

#caddy #nginx #webserver #homelab #engineering
---

## Post 27: "Wait, can I even run a web server behind CGNAT?" (a misconception worth untangling)

> **Image suggestion:** Diagram of the full request path with a bright line showing where CGNAT applies (router inbound) and where it does not (localhost). A padlock on the router side, nothing on the loopback side.

Partway through setting up Caddy, I stopped and asked myself a question that felt alarming:

If I am behind CGNAT and cannot open ports, can I even USE a web server like Caddy or nginx at all?

Good instinct to double-check. But it was based on a misunderstanding worth untangling properly, because I suspect it trips up a lot of people.

CGNAT only affects INBOUND connections arriving from the public internet at my router. It has nothing to do with ports used purely LOCALLY, inside one machine, between two processes running on that same machine.

When Caddy binds to port 80 on my VM, that is happening on the VM's own loopback / local network stack. It is one local program (Caddy) listening for connections from another local program (cloudflared). Nothing about that "opens a port to the internet" in any meaningful sense.

The full request path:

```
Public internet
   → Cloudflare's edge (real public IPs, theirs, not mine)
   → down the already-open outbound tunnel connection
   → cloudflared, running on srv1
   → localhost:80
   → Caddy
```

Walk that chain and notice: nowhere does my router accept an inbound connection, forward a port, or need a public IP. localhost:80 never leaves the VM. It is purely how two programs on the same machine talk to each other.

The only thing in the entire chain that touches the public internet is the tunnel itself, and it works by dialing OUTWARD, from my VM to Cloudflare. An outbound connection, which CGNAT never restricts, because outbound is what your machine does all day every day.

The CGNAT wall only ever applied to the approach I abandoned at the very start of this project: forwarding a port on my ROUTER for the wider internet to connect to directly. A web server running locally was never subject to that restriction, no matter which one I chose.

The loopback address, if the term is new: 127.0.0.1 (IPv4) or ::1 (IPv6) is a special address a machine uses to talk to itself. Traffic to it never leaves the machine, so it is untouched by CGNAT, routers, or anything network-level at all.

File this under "questions that feel silly until you realize how many architectures they quietly explain." Local ports and public ports live in different worlds. Tunnels exist precisely to bridge those worlds in one direction only.

Next post is my favorite of the series: the site went down, my first theory was wrong, and three commands taught me more about debugging than the rest of the project combined.

#CGNAT #networking #webhosting #homelab
---

## Post 28: The best debugging lesson of the whole build: evidence over guessing

> **Image suggestion:** Terminal screenshot triptych: empty ss -tlnp output, the Caddy log showing port 443, then ss showing port 80 bound after the fix. Three screenshots, one debugging story. Would also work as a 3-slide carousel.

Read this one slowly. The specific bug does not matter. The method does.

The symptom: after installing Caddy and writing a Caddyfile, the tunnel started throwing:

```
ERR error="Unable to reach the origin service. The service may be down or it may not be
responding to traffic from cloudflared: dial tcp [::1]:80: connect: connection refused"
```

My first theory: I spotted [::1]:80 in the error. ::1 is the IPv6 form of localhost, equivalent to 127.0.0.1 in IPv4. So my assumption was: cloudflared is connecting over IPv6, Caddy is only listening on IPv4, they are mismatched. Fix: point the tunnel explicitly at 127.0.0.1 to force IPv4.

That was a REASONABLE theory given the evidence at that moment. The error literally contained an IPv6 address.

But reasonable is not the same as correct. And I was about to just apply the fix and hope, without verifying the theory against anything.

Instead, I stopped and gathered direct evidence. Three commands, each answering one narrow question:

Question 1: is anything listening on port 80 at all, on ANY protocol?

```
sudo ss -tlnp | grep :80
```

Completely empty. No output.

That one result invalidated my entire IPv4-vs-IPv6 theory in a single command. It was not a mismatch between two things listening on different protocols, because NOTHING was listening on port 80 whatsoever. The IPv6 detail was a red herring. The real problem was one level simpler than I had assumed.

Question 2: can I confirm this from the receiving side, bypassing Cloudflare entirely?

```
curl -v http://127.0.0.1:80
```
```
* connect to 127.0.0.1 port 80 ... failed: Connection refused
```

Independent confirmation. Even a purely local request, no tunnel, no DNS, no internet involved, got refused. Problem isolated to exactly one statement: something on this machine is not listening on port 80. Cloudflare, DNS, and everything beyond this box are now formally cleared.

Question 3: so what IS Caddy actually doing?

```
sudo systemctl status caddy --no-pager -l
```
```
"logger":"http","msg":"enabling HTTP/3 listener","addr":":443"
"msg":"server running","name":"srv0","protocols":["h1","h2","h3"]
```

There is the answer, in plain text. Caddy was alive and running the whole time, but listening on port 443, under a generic server name of srv0. Port 80 appears nowhere in the log.

Root cause: the Caddyfile did not match intent. The domain block was not specified in a way Caddy interpreted as "listen on port 80 for this exact address," and even with automatic HTTPS turned off, Caddy chose to bind a bare domain name to 443.

The fix: remove all ambiguity by stating the port explicitly.

```
{
    auto_https off
}

22112002.xyz:80 {
    root * /var/www/22112002.xyz
    file_server
}
```

Reload, re-check:

```
sudo systemctl reload caddy
sudo ss -tlnp | grep :80
```

This time a line appeared. Caddy bound to 80, cloudflared could reach it, site live.

Now the generalized lesson, the one that transfers to every system you will ever debug:

When a theory forms, do not stack a fix on top of it and hope. Narrow the problem with commands that produce FACTS:

- "Is anything listening on this port at all?" (ss -tlnp)
- "Can I reproduce the failure locally, with nothing else involved?" (curl at the local address)
- "What does the service's own log say it is doing?" (systemctl status, journalctl)

Each one confirms or eliminates a specific hypothesis. My IPv6 theory felt plausible from the error alone; a thirty-second ss check eliminated it instantly and pointed straight at the real cause. A port mismatch, not a protocol mismatch.

Prefer a command that produces a fact over a change that produces a hope.

That sentence is the most valuable thing I took from this entire build.

#debugging #linux #devops #engineering #caddy
---

## Post 29: The final architecture, the numbers, and what this build actually taught me

> **Image suggestion:** Two options, or both as a carousel: the final architecture diagram (reuse and extend the post 24 one, now with Tailscale drawn in) and a photo of the machine sitting in its corner, running, unremarkable. End on the humble hardware.

Series wrap-up. Here is what is running right now, end to end, on one repurposed PC in my house:

```
Internet
   ↓
Cloudflare edge (has real public IPs, I never needed one)
   ↓  relayed down an outbound-initiated tunnel
cloudflared, running on srv1
   ↓  forwards to one fixed local port
Caddy, listening on 22112002.xyz:80
   ↓  routes internally by requested hostname
/var/www/22112002.xyz/  →  my portfolio's files
```

Alongside that: Tailscale mesh across the Proxmox host, the VM, and my laptop, so I can SSH into either server from anywhere on earth. No public IP, no port forwarding, coexisting fine with my separate pre-existing VPN.

Adding a new site, a client domain or a blog subdomain, is now the same boring, repeatable recipe every time:

1. New folder under /var/www/
2. New short block in the Caddyfile, with its own explicit :80
3. One command: cloudflared tunnel route dns homelab <newdomain>
4. One new hostname entry in the tunnel's ingress, still pointing at the same Caddy port
5. sudo caddy validate --config /etc/caddy/Caddyfile BEFORE reloading
6. sudo systemctl reload caddy and restart cloudflared

None of those steps touch each other's territory. The tunnel never knows Caddy changed. Caddy never knows how traffic arrived. That separation of concerns was the entire point of the reverse proxy.

What is next, with reasoning:

- RAM upgrade, 8GB to 16GB. The original two-VM plan (one for hosting, one for automation) was too tight on 8GB once host overhead plus two full guest OSes were counted. 16GB makes two proper VMs comfortable.
- Keeping hosting and automation as SEPARATE servers, deliberately. A website strangers visit should not share a blast radius with internal automation tooling that has broader access to my accounts. Same isolation reasoning as everything else, one level up.
- AI agents, with a reality check. I want n8n running locally as the orchestrator, but running LLM INFERENCE locally needs a GPU I do not have. Correct architecture: the box coordinates workflows and calls out to a hosted model API for the actual reasoning. It does not need to be the thing doing the heavy computation.
- The honest weak point: everything runs on one spinning hard drive. If this ever becomes something other people rely on, an SSD matters more than any further RAM.

And the lessons I am actually keeping, condensed from 21 posts of mistakes:

1. Constraints shape architecture. CGNAT, one fact about my connection, dictated Tailscale, Cloudflare Tunnel, and half the design. Understand your constraints before your tools.
2. Read the screen. Enter is not always confirm. Comments are not commands. Usernames have an exact spelling.
3. Error messages report symptoms, not causes. "Permission denied" was a typo. "Certificate not valid" was a clock. "Unreachable" from a weird IP was my own VPN. The details you skim are the diagnosis.
4. When something fancy silently fails, fall back to something boring. Router DNS over public resolvers. HTTP/2 over QUIC. Boring gets through.
5. Isolate change to cheap layers. The tunnel stays simple forever; the Caddyfile absorbs all future change. Design so mistakes land where they are safe.
6. And above everything: prefer a command that produces a fact over a change that produces a hope.

Total spent on software and services for all of this: zero. One old PC, one cheap domain, and a long list of mistakes I am genuinely glad I made, because every one of them is now a reflex.

If you have an old machine gathering dust and an ordinary home connection, you have everything this build needed.

Thanks for following along. Questions about any part of the setup, drop them below. And if you are behind CGNAT and stuck, I have been exactly where you are.

#homelab #selfhosting #linux #devops #learninpublic #careergrowth
