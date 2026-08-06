# LinkedIn Series: Homelab Foundations (the concepts behind the build)
### 6 posts. These sit BEFORE the 22-post journey series, or run as their own mini-series.
### Full write-ups live in the repo. Mention "link in the comments" and drop the repo link there, since LinkedIn punishes links in the post body.

---

## Post F1: What is a hypervisor? (The question I should have answered before touching my server)

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

## Post F2: Type 1 vs Type 2 hypervisors, or why VirtualBox is not a server

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

## Post F3: How does a VM not know it is fake?

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

## Post F4: VMs vs containers is not a war. Here is when I use each.

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

## Post F5: Why Proxmox? (An honest comparison, including when NOT to pick it)

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

Next post wraps this mini-series with the practical Proxmox knowledge I actually use weekly, including the one confusion that costs people their data: snapshots are not backups.

#proxmox #vmware #homelab #selfhosting #opensource

---

## Post F6: Snapshots are not backups (and other Proxmox lessons I use every week)

Last post of the foundations series. These are the practical rules I actually operate by, and one of them protects your data from a mistake that catches almost everyone.

First, the rule of thumb for guests. Proxmox gives you two kinds: full KVM virtual machines and lightweight LXC containers. My rule: reach for a container when it is a simple Linux service. Reach for a VM when I need a different OS, hard isolation, or real hardware handed to the guest. And if a container has to be created privileged, I pause and reconsider, because unprivileged is the safe default. Guest root should never equal host root.

Second, the settings I never skip when creating a VM, because each earns its place. VirtIO for the disk controller and the network card, so the guest cooperates with the hypervisor instead of being lied to slowly. And the QEMU guest agent installed inside the VM, so Proxmox can shut it down cleanly and actually show me its IP address. I once stared at a blank IP field in the summary page for a while before learning that last one.

Third, and this is the big one. Snapshots and backups are different tools, and confusing them will eventually cost someone their data. Maybe you. So:

A snapshot is a point-in-time capture of a VM's state, stored on the SAME disk as the VM. It is instant, and it is perfect for the moment right before a risky change. Upgrade goes wrong, roll back in seconds.

But if that disk dies, the snapshot dies with it. A snapshot on a failing drive protects you from nothing.

A backup is a full, self-contained archive of the guest written SOMEWHERE ELSE. Another disk, a NAS, a Proxmox Backup Server. It survives the death of the host itself.

The practice that follows: snapshot before every risky change, scheduled backups for actual safety, and at least one copy living off the machine. And test a restore once in a while. An untested backup is a hope, not a backup.

This distinction is personal for me. My entire homelab currently lives on one spinning 500GB hard drive from another era. Snapshots make me brave day to day. Backups are what let me sleep.

Last thing, on networking, because it confused me early. Proxmox creates a virtual switch called vmbr0 and plugs your physical network port into it. Every VM gets a virtual cable into that same switch. The result is that your VMs appear on your home network as if they were separate physical machines, pulling IPs from your router like any laptop or phone would. Once I pictured it as a switch instead of something mystical, Proxmox networking stopped being scary.

That closes the foundations. Hypervisors, how the trick works, VMs versus containers, why Proxmox, and how to not lose your data. The full written notes are in my repo, link in the comments.

From here, the story gets messy: what actually happened when I built this thing for real, including my ISP fighting me and one keystroke that cost me hours. That series is next.

#proxmox #homelab #backups #sysadmin #selfhosting
