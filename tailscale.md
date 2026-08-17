# The Day My Proxmox Host Went Dark: A Tailscale Recovery Story

This covers one single session, start to finish: I tried to reach my Proxmox host over Tailscale and couldn't. What looked like one problem turned out to be three separate, stacked problems, each with its own lesson. I'm writing it the same way as the other guides — mistakes included, reasoning included — because the debugging process here is more valuable than the final fix.

---

## Part 0: The starting symptom

I tried to reach Proxmox's web UI using its Tailscale IP and got nothing. My first instinct was to assume Tailscale itself was broken somehow. It wasn't — but figuring out *what actually was* wrong took working through several layers, one at a time, using evidence rather than guesses at each step. That method — check first, fix second — is the actual thread running through this whole session.

---

## Part 1: Finding out what was actually wrong (not what I assumed was wrong)

Instead of immediately trying to "fix Tailscale," the first move was to just look at what Tailscale itself reported, from my laptop:

```bash
tailscale status
```

```
100.113.156.34  oll   ...  linux  -
100.100.200.25  pve   ...  linux  offline, last seen 13d ago
100.72.72.58    srv1  ...  linux  active; relay "jnb", tx 7488 rx 0
```

This one command told me something specific and important: **only `pve` was offline, and it had been offline for 13 days, not just since today.** That's a very different problem than "Tailscale broke just now." Something had been wrong with the Proxmox host's Tailscale connection for nearly two weeks, and I simply hadn't tried to reach it in that window to notice.

**The lesson here:** always check the actual status output before assuming which layer is broken. If I'd jumped straight to reinstalling Tailscale everywhere, I'd have wasted time on `oll` and `srv1`, which were both completely fine the whole time. The problem was isolated to exactly one machine, and the tool told me that for free, in one command.

---

## Part 2: SSH'ing in and finding three separate issues at once

Since Tailscale itself was down on `pve`, the only way in was the old reliable path — the plain LAN IP:
```bash
ssh root@10.0.0.50
```

From here, three checks, each answering a specific question:

### Check 1: Is the machine actually freshly booted, or has it been running the whole time?
```bash
uptime
```
```
17:25:01 up 10 min
```
Only **10 minutes** of uptime. This told me immediately that the box had just been turned on — meaning whatever was wrong with Tailscale wasn't a live crash I was catching mid-failure, it was something that needed to re-establish itself after however long the machine had actually been off.

### Check 2: What is the Tailscale service itself doing right now?
```bash
systemctl status tailscaled --no-pager -l
```
The key line:
```
Status: "Needs login: "
```
This was the actual, specific problem — not a crashed service, not a misconfiguration, just a session that had logged itself out and was waiting to be re-authorized. Useful to know precisely, because "needs login" has a very different fix than "service crashed" or "service never started."

### Check 3: Is the clock right? (Learned to check this reflexively after the certificate saga earlier in the project)
```bash
timedatectl
```
```
System clock synchronized: no
NTP service: inactive
```
Not the main problem this time, but a real one sitting alongside it, worth fixing regardless since a wrong clock has bitten this project before (the "SSL certificate not yet valid" issue from setting up Cloudflare Tunnel). Running `timedatectl set-ntp true` and re-checking a few minutes later showed it correctly synced on its own.

**The lesson:** don't stop investigating after finding the first plausible cause. The clock issue and the login issue were both real, independent problems that happened to surface at the same time. Fixing only one would have left the other lurking.

---

## Part 3: The bigger question — why was the host off for so long?

The "needs login" state made sense once I found out *why*: the host hadn't been continuously running. But I wanted to know whether this was a crash, a power outage, or something I did on purpose — because each of those has a completely different follow-up action.

```bash
last -x | head -20
```

The critical line:
```
shutdown system down  7.0.2-6-pve   Mon Apr 13 22:14 - 17:14  (125+19:00)
```

Reading this carefully: this is labeled **`shutdown`**, not `crash` — a handful of other lines further down from that same night *were* labeled crash, from earlier restarts during setup, but this particular long gap was a clean, deliberate shutdown command. And the duration — **125 days, 19 hours** — lines up exactly with the gap between an April 13th shutdown and today, August 17th, when I physically/remotely powered it back on.

**What this actually means:** nobody's server crashed, no power outage brought it down, no mystery fault occurred. At some point back in April, a proper shutdown was issued, and the machine simply sat off, exactly as told, until today.

**The lesson, and the actual takeaway I want to remember:** when I intentionally shut this box down for any reason, *everything* running on it goes down with it — Proxmox, the `srv1` VM, Caddy, the Cloudflare Tunnel, my live portfolio site — and stays down silently until I manually power it back on. There's no safety net here; a homelab server isn't like a phone that wakes itself up. If I ever shut this down deliberately again (moving it, doing hardware work, whatever the reason), I need to actively remember to turn it back on — nothing will remind me, and nothing outside will show any sign of trouble except "the site's been down for months and nobody noticed."

This also retroactively explains something from earlier in the project: I'd been advised to check the BIOS setting "Restore on AC Power Loss → Power On" in case of an unexpected outage. That advice doesn't actually apply to *this* incident specifically, since this was a deliberate shutdown, not a power loss — but it's still worth having set regardless, for the *next* time it might genuinely be an outage rather than something I did on purpose.

---

## Part 4: Getting logged back in — and a string of small, honest mistakes

This part took longer than it should have, and every step of the delay taught something.

### Mistake: running bare `tailscale up` after changing a setting
```bash
tailscale set --ssh
tailscale up
```
```
Error: changing settings via 'tailscale up' requires mentioning all
non-default flags. ... use the command below ...
	tailscale up --ssh
```
I'd turned on Tailscale's own SSH feature (`--ssh`) as a separate setting, and then tried to bring the connection up without mentioning it again. Tailscale's `up` command intentionally refuses to silently drop settings you've already configured — it wants you to explicitly restate them so you can't accidentally lose a setting by omission. Not a bug, a safety behavior. The fix was simply including the flag: `tailscale up --ssh`.

### Mistake: pasting explanatory text into the terminal along with the real command
At one point a stray line appeared in my terminal:
```
bash: If: command not found
bash: syntax error near unexpected token `('
```
This happened because a chunk of plain-English explanation (starting with the word "If") got copied into the terminal alongside an actual command. The terminal tried to execute the English sentence as a shell command, predictably failing. Harmless, but a good reminder: when copying multi-line blocks that mix explanation and commands, only the actual code should go into the terminal.

### The real blocker: a stuck terminal after already authorizing in the browser
I clicked "Authorize" in the browser, but the terminal running `tailscale up --ssh` never returned to a normal prompt. This was genuinely confusing — the dashboard even showed `pve` with an "SSH" tag applied, suggesting *something* had registered, but the terminal sat frozen.

**Instead of waiting indefinitely or restarting things randomly, I opened a second, separate SSH session to the same host** — this is a genuinely useful technique: use a second window to inspect the live state of a machine without disturbing whatever the first, stuck session is doing.

From that second session:
```bash
sudo journalctl -u tailscaled -n 30 --no-pager -l
```

This is where the *actual* root cause finally surfaced, in plain text in the logs:
```
control: RegisterReq: got response; nodeKeyExpired=false, machineAuthorized=false; authURL=true
...
health(warnable=login-state): error: You are logged out. The last login error was: register request: http 410: auth path not found
```

**`http 410: auth path not found`** — this is a server-side rejection. The specific authorization URL being generated was being refused immediately by Tailscale's own control servers. This wasn't a stuck terminal at all — it was a terminal correctly waiting on a login flow that was *doomed to fail every single time*, because something about this node's registration state was already broken server-side, most likely a leftover, half-deleted, or conflicting registration for the same machine from an earlier attempt (I had tried to delete and recreate `pve`'s Tailscale identity earlier in the session, and the deletion and recreation likely raced each other, or the delete didn't fully propagate before a new registration attempt began).

**The lesson:** "it's stuck" and "it's failing repeatedly but instantly" can look identical from a first glance at a hanging terminal. The only way to tell them apart is to check the logs of the actual service doing the work, from a second vantage point, rather than continuing to stare at (or restart) the one frozen window.

### The actual fix: wipe local state and start completely fresh
```bash
sudo systemctl stop tailscaled
sudo rm -f /var/lib/tailscale/tailscaled.state
sudo systemctl start tailscaled
```
Deleting the local state file forces Tailscale to forget any half-completed identity and generate an entirely new node key from scratch, rather than trying to resume a broken registration. Combined with confirming the old `pve` entry was fully gone from the Tailscale admin console (not just "deletion clicked," but actually absent from the machine list), this cleared the conflict.
```bash
sudo tailscale up --ssh
```
This produced a **brand new** login URL — and using that fresh URL, rather than any earlier one still sitting open in a browser tab, is what finally worked. Old, previously-generated auth URLs don't stay valid indefinitely; reusing a stale one (even if it looks identical) can silently fail without any obvious error.

---

## Part 5: One more small trap right at the finish line

Once `pve` reconnected — under a **new** Tailscale IP, since it re-registered as a fresh node (`100.107.46.32`, different from the old `100.100.200.25`) — I tried:
```bash
ssh pve@100.107.46.32
```
```
tailscale: tailnet policy does not permit you to SSH as user "pve"
```

This looks alarming (a policy rejection!) but is actually the same class of mistake as the earlier `svr1`/`srv1` typo: **`pve` is the machine's name, not a username that exists on it.** The real account is `root`. Correct command:
```bash
ssh root@100.107.46.32
```

Worth noting the mechanism behind the error message too, since it's new: because Tailscale's own SSH feature (`--ssh`) is enabled on this node, login attempts now pass through **Tailscale's own SSH layer and ACL policy**, in addition to normal SSH — so an invalid username gets rejected with a "tailnet policy" message rather than a plain "no such user" error. Same root mistake as before, just a different, less obvious error message because a different system was doing the rejecting.

---

## Part 6: What I actually learned from this whole session

1. **Always check status before assuming a cause.** `tailscale status` immediately narrowed the problem to one machine instead of three, in one command.
2. **A machine that's "just been rebooted" (short uptime) has a different problem than one that's been running the whole time.** Check `uptime` early — it changes which fixes even make sense to try.
3. **`last -x` distinguishes a clean shutdown from a crash.** A long gap with no "crash" label means someone told it to turn off — worth knowing the difference before assuming a hardware fault.
4. **A stuck-looking terminal and a repeatedly-failing terminal can look identical.** The only way to tell them apart is checking the service's own logs from a second, independent vantage point — not restarting things blindly, not waiting indefinitely.
5. **Old auth/login URLs can go stale.** Always use the most recently generated one, and don't assume an old tab still works.
6. **A machine's name and a username on that machine are two different things, always.** This has now bitten me twice (`svr1` vs `srv1`, and `pve` vs `root`) — worth internalizing as a permanent checklist item before typing any `ssh user@host` command: *am I sure "user" is actually an account, not just the machine's label?*
7. **Deliberately shutting this box down means everything on it goes dark until I manually turn it back on.** No automatic recovery, no warning, nothing — a fact worth remembering every single time I power it off on purpose.

---

## Part 7: Quick-reference — the actual commands that fixed it, in order

```bash
# 1. Diagnose from the laptop first
tailscale status

# 2. Get onto the host via LAN (since Tailscale itself is what's down)
ssh root@10.0.0.50

# 3. Establish the real state of the host
uptime
systemctl status tailscaled --no-pager -l
timedatectl
last -x | head -20

# 4. Fix the clock if needed
sudo timedatectl set-ntp true

# 5. If tailscale up hangs or fails repeatedly, check the real logs
#    from a SECOND ssh session, not the stuck one
sudo journalctl -u tailscaled -n 30 --no-pager -l

# 6. If logs show a registration conflict (http 410 or similar),
#    wipe local state and re-register clean
sudo systemctl stop tailscaled
sudo rm -f /var/lib/tailscale/tailscaled.state
sudo systemctl start tailscaled
sudo tailscale up --ssh

# 7. Confirm from the laptop
tailscale status

# 8. Connect using the correct username (root, not the machine's name)
ssh root@<new-tailscale-ip>
```

---

## Glossary additions from today

- **`tailscaled`** — the background service (daemon) that actually maintains a machine's Tailscale connection; `tailscale` (no "d") is the command-line tool used to talk to it.
- **Node key** — the cryptographic identity Tailscale generates for a specific device; deleting the local state file forces a brand new one to be generated, effectively making the device "new" to the tailnet again.
- **`http 410`** — an HTTP status code meaning "this resource used to exist but is now gone, permanently." Seeing this from a login/registration attempt is a strong signal of a stale or conflicting server-side record, not a client-side problem.
- **Clean shutdown vs crash** — visible in `last -x` output: a line labeled `shutdown` means the OS was told to power off and did so in an orderly way; a line labeled `crash` means the system stopped unexpectedly without a graceful shutdown sequence.
- **Tailscale SSH (`--ssh`)** — an optional feature where Tailscale itself brokers SSH access using tailnet identity and policy, layered on top of (and enforced before) the normal system SSH login — meaning login failures can now come from tailnet policy, not just the OS's own user database.
