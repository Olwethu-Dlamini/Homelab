# Hosting Multiple Sites on My Homelab: Caddy, the Roads Not Taken, and a Real Debugging Story

This picks up right where the last guide left off — Proxmox, Tailscale, and Cloudflare Tunnel were already working, with a single test page live on `22112002.xyz`. This document covers the next problem: **how do I host more than one website behind a single server, and a single tunnel, without it turning into a mess as I add more sites over time?**

Same rule as last time: I'm keeping the wrong turns in, because the wrong turn plus the reasoning for correcting it is worth more than a clean answer with no context.

---

## Part 1: The actual problem — one tunnel, many websites

Up to this point, my Cloudflare Tunnel pointed straight at one thing: a simple test page running on port 8080. That's fine for one site. But I want multiple — my portfolio now, client sites later, possibly a blog subdomain, all under `22112002.xyz` and its subdomains, maybe other domains entirely later.

**The naive approach** would be: give the tunnel's config one `hostname → port` line per site, and run a separate little webserver process on a separate port for each one.
```yaml
ingress:
  - hostname: 22112002.xyz
    service: http://localhost:8080
  - hostname: blog.22112002.xyz
    service: http://localhost:8081
  - hostname: clientsite.com
    service: http://localhost:8082
```

**Why I didn't go this route, even though it technically works for a small number of sites:**

- Every new site means touching the tunnel config and restarting `cloudflared` — a service that's supposed to be my stable, always-on public entry point. I don't want to be restarting my *only* path to the internet every time I add a page.
- Every site needs its own dedicated running process bound to its own port, which I'd have to track manually and remember not to collide (which port is free? which one did I already use?).
- No shared behavior across sites — if I ever want gzip compression, access logging, or basic security headers applied consistently, I'd have to configure it separately, per site, per process.
- It doesn't scale in the way I actually want to scale — "add a client's site" should be a small, boring, repeatable action, not something that touches my core public-facing infrastructure each time.

**The standard, correct pattern instead: put one reverse proxy in front of everything, and let the tunnel talk to *only* that.**
```
Internet → Cloudflare → Tunnel → [ONE reverse proxy] → routes internally by hostname → Site A / Site B / Site C
```
Now the tunnel's job is permanently simple and never changes: forward every request to the reverse proxy, always on the same port. The reverse proxy is the only thing that ever needs updating when a new site is added, and updating it doesn't touch the tunnel or the public-facing plumbing at all.

This is the same idea, one layer removed, as everything else in this project: don't let each new piece require touching the fragile, hard-to-debug layer underneath it. Isolate change to the layer where it's cheap and safe to make mistakes.

---

## Part 2: The tools I *didn't* pick, and exactly why

I want to document this properly because "why not X" questions are usually more educational than "how do I do Y" questions — they force you to actually understand the tradeoffs instead of memorizing one path.

### Why not cPanel?

cPanel came up because it's the name most associated with "hosting multiple websites" in general internet culture — lots of shared hosting providers use it, so it's the default mental model a lot of people have for "managing websites on a server."

But cPanel is a **paid, licensed product** that happens to run on top of Linux — it is not itself a Linux distribution, and it's not free. Concretely, here's why it was wrong for this project specifically:

- **Cost**: licensing runs somewhere in the $15–45+/month range depending on account limits, charged to whoever operates the server. This directly contradicts the free/self-hosted approach I've been using for everything else in this build (Tailscale free tier, Cloudflare's free tunnel, free OS, free reverse proxy).
- **RAM**: cPanel's own recommended minimum is around 1GB just for the panel itself, more realistically 2GB+, before any actual site traffic. On hardware where I was already budgeting carefully between the host, one VM, and future services like n8n, that's a real chunk of a scarce resource spent on a management dashboard rather than anything my actual sites need.
- **OS lock-in**: cPanel only installs on CentOS/AlmaLinux/CloudLinux family systems. My VM is Debian. Using cPanel would have meant either rebuilding my VM on a different OS, or standing up a second VM just to host the panel — more RAM overhead, more moving parts, for a GUI wrapped around functionality I can get for free with a six-line text file.
- **Mismatch with what I'm actually doing**: cPanel earns its cost when you're reselling hosting to many non-technical clients who each need isolated logins, email inboxes, and no command-line involvement whatsoever. I'm a developer, hand-deploying a small number of sites for myself and possibly a few people I know personally. The complexity cPanel adds is solving a problem I don't have.

If I ever *do* want a GUI-style dashboard instead of editing text files, there are free, self-hosted alternatives that don't have cPanel's cost or OS restriction — CyberPanel (open-source, similar feel, runs on Debian/Ubuntu, uses OpenLiteSpeed under the hood) being the closest match. I'm noting this because "no GUI at all, forever" isn't actually the constraint — "no paid license, no OS lock-in, minimal RAM" is the real constraint, and it happens to rule out cPanel specifically while leaving room for lighter free options later if I ever want them.

### Why not nginx?

This one's more personal preference than a hard technical disqualifier — nginx is a completely legitimate, extremely widely used choice, and arguably the more "expected" skill on a CV compared to what I did pick. I said upfront I didn't want to work with it, so it's worth being honest about *why*, rather than pretending it's objectively bad:

- **Verbose, easy-to-break config syntax.** Every site needs a `server { }` block, explicit `listen` and `server_name` directives, a `location` block with `try_files` logic just to serve static files correctly, and every line needs a trailing semicolon — miss one, and the whole config can fail to load, sometimes with an error message that doesn't point clearly at the actual mistake.
- **No built-in automatic HTTPS.** Nginx doesn't manage TLS certificates on its own; that's normally bolted on separately with Certbot, plus a renewal cron job to remember. In my specific setup this barely matters, since Cloudflare Tunnel already handles the public-facing HTTPS side of things — but it's still one more moving part nginx doesn't handle natively that some other tools do.
- **Manual reload discipline.** You're expected to run `nginx -t` to test a config *before* reloading, as a separate step you have to remember — skip it, and a bad edit can silently take down the whole server on reload instead of refusing to apply.

None of this makes nginx bad — it makes it a tool optimized for people who want maximum low-level control and are comfortable reading extensive documentation to get it. Given how many small, avoidable mistakes I'd already made elsewhere in this project (a mistyped username, hitting Enter instead of Space on an installer screen, forgetting a package existed), I made a deliberate choice to reduce the *number of places I could make a syntax mistake* for this particular piece, rather than optimize for "most commonly expected on a CV."

### Why not Traefik or Apache?

Briefly, since these came up as alternatives worth naming even though I didn't seriously consider them:

- **Traefik** shines when you have many Docker containers that need to be automatically discovered and routed to, using labels on each container instead of a central config file. That's a fantastic fit for a more container-heavy, microservices-style setup — but I'm hosting a handful of static sites directly on a VM, not orchestrating a fleet of containers, so Traefik's core strength doesn't apply to my situation, and its YAML/label-based config is a steeper learning curve for no corresponding benefit right now.
- **Apache (httpd)** is the oldest, most battle-tested option here, with an enormous ecosystem and support for per-folder `.htaccess` config. It's heavier than necessary for pure static hosting and has an older-feeling config style compared to more modern options — there's no strong reason to reach for it over something lighter unless a specific project already depends on Apache-specific features.

---

## Part 3: Why Caddy, explained properly

Caddy is a web server — same category as nginx and Apache — built around two specific design goals: **sensible defaults**, and **automatic HTTPS out of the box**. Its config file is called a `Caddyfile`, written to read closer to plain English than nginx's syntax does.

Here's the size difference for the *exact* task I needed — serve one static site, matched by domain name:

**Caddy — the whole thing:**
```caddyfile
22112002.xyz {
    root * /var/www/22112002.xyz
    file_server
}
```

**The nginx equivalent, doing the same job:**
```nginx
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

**Why this fits my actual setup, concretely:**
- Adding a second site is just one more short block — no separate `sites-available`/`sites-enabled` symlink dance that nginx traditionally uses on Debian.
- `sudo systemctl reload caddy` re-reads config live, and — this turned out to matter a lot in my actual debugging session below — Caddy can **validate config syntax before applying it**, via `caddy validate --config ...`. A broken edit gets caught before it's live, rather than after.
- Low RAM footprint, appropriate for hardware I'm already budgeting carefully.
- Automatic HTTPS is Caddy's most famous feature, but in my setup it's almost beside the point — Cloudflare Tunnel already handles public-facing HTTPS. The real reason Caddy earns its place here is purely how much less there is to write, and get wrong, for straightforward multi-site static hosting.

**The honest tradeoff I'm accepting:** nginx experience is more commonly asked for in job listings, and shows up more often on other developers' CVs. Caddy is an excellent, production-grade tool, but it's less universally "expected knowledge" industry-wide. I made this trade deliberately, prioritizing "get this right without a debugging marathon" over "match the most common industry tool" — a reasonable call for a personal homelab, worth revisiting if a specific job or client situation ever calls for nginx specifically.

---

## Part 4: The CGNAT question I asked myself again — and why it didn't apply here

Partway through setting up Caddy, I paused and asked: *wait, can I even use nginx/Caddy at all if I'm behind CGNAT and can't open ports?*

This was a good instinct to double-check, but it turned out to be based on a misunderstanding I needed to untangle properly.

**CGNAT only affects *inbound* connections arriving from the public internet at my router.** It has nothing to do with ports used purely *locally*, inside one machine, between two processes that are both running on that same machine.

Caddy binds to a port (80, in this case) on the VM's own **loopback / local network stack** — that's just one local program listening for connections from another local program (`cloudflared`, in this case). Nothing about that "opens a port to the internet" in any meaningful sense. The only thing in this entire chain that ever touches the actual public internet is the Cloudflare Tunnel itself, and it works specifically by **dialing outward**, from my VM to Cloudflare — an outbound connection, which CGNAT never restricts. CGNAT only blocks the reverse: the outside world trying to dial *in*.

So the full request path looks like this:
```
Public internet → Cloudflare's edge (real public IPs) → outbound tunnel connection (already open) → cloudflared, running on srv1 → localhost:80 → Caddy
```
Nowhere in that chain does my router need to accept an inbound connection, forward a port, or have a public IP of its own. `localhost:80` never leaves the VM — it's purely how two programs on the same machine talk to each other. The CGNAT wall only applies to the specific approach I abandoned back at the very start of this whole project: forwarding a port on my *router* for the wider internet to connect to directly. Caddy (or nginx, or anything else) running locally was never subject to that restriction in the first place, regardless of which reverse proxy tool I chose.

---

## Part 5: The live debugging story — evidence over guessing

This is the part worth reading slowly, because the actual lesson isn't the specific fix — it's the *method* I used to get there, after nearly going down a wrong path first.

### The symptom
After installing Caddy and writing a Caddyfile, the tunnel started throwing:
```
ERR error="Unable to reach the origin service. The service may be down or it may not be responding to traffic from cloudflared: dial tcp [::1]:80: connect: connection refused"
```

### My first theory — and why it was wrong, but reasonable
I noticed `[::1]:80` in the error — `::1` is the IPv6 form of "localhost" (equivalent to `127.0.0.1` in IPv4). My first assumption was: *cloudflared is trying to connect over IPv6, and Caddy is only listening on IPv4, so they're mismatched — point the tunnel explicitly at `127.0.0.1` instead of `localhost` to force IPv4.*

This was a **reasonable theory to form from the evidence available at that moment** — the error literally contained an IPv6 address. But reasonable isn't the same as correct, and I hadn't actually verified the theory against real evidence yet — I was about to just apply a fix and hope.

### Stopping to actually check, instead of guessing further
Before applying more changes, the right move was to get **direct evidence** about what was actually happening, rather than layering another guess on top of an unconfirmed one. Three commands, each answering a specific, narrow question:

**1. Is anything even listening on port 80, at all, on any protocol?**
```bash
sudo ss -tlnp | grep :80
```
This came back **completely empty.** No output at all. That one result invalidated my entire IPv4-vs-IPv6 theory in a single command — it wasn't a mismatch between two things both listening on different protocols, because **nothing was listening on port 80 whatsoever.** The IPv6 detail in the original error was a red herring; the real problem was one level simpler than I'd assumed.

**2. Can I confirm this from the "receiving" side too, bypassing Cloudflare entirely?**
```bash
curl -v http://127.0.0.1:80
```
```
* connect to 127.0.0.1 port 80 from 127.0.0.1 port 50584 failed: Connection refused
```
This confirmed it independently — even a purely local request, with no tunnel, no DNS, no internet involved at all, got refused. This isolated the problem to *just* "something on this one machine isn't listening on port 80" — nothing to do with Cloudflare, DNS, or networking beyond the machine itself.

**3. So what IS Caddy actually doing?**
```bash
sudo systemctl status caddy --no-pager -l
```
The logs held the actual answer, right there in plain text:
```json
"logger":"http","msg":"enabling HTTP/3 listener","addr":":443"
"logger":"http.log","msg":"server running","name":"srv0","protocols":["h1","h2","h3"]
```
Caddy was alive and running the whole time — but it was listening on **port 443**, and under a server name of `"srv0"`, not `22112002.xyz`. Nowhere in that log did port 80 appear at all.

### The actual root cause
The Caddyfile, at that point, didn't match what we intended — either through an edit that got lost, or a version where the domain block wasn't specified in a way Caddy interpreted as "listen on port 80 for this exact address." Caddy, even with automatic HTTPS explicitly turned off, was still choosing to bind to 443 for a bare domain name in this situation, rather than 80.

### The actual fix — remove all ambiguity by stating the port explicitly
```caddyfile
{
	auto_https off
}

22112002.xyz:80 {
	root * /var/www/22112002.xyz
	file_server
}
```
Adding `:80` directly onto the site's address line removes any possible ambiguity about which port Caddy should bind to for that block. After reloading:
```bash
sudo systemctl reload caddy
sudo ss -tlnp | grep :80
```
This time, a line actually appeared — Caddy was finally bound to port 80, `cloudflared` could reach it, and the site went live.

### The lesson, generalized
When a fix based on a first theory doesn't resolve things (or before even applying one), the correct move is to **narrow the problem with direct, verifiable commands** rather than stacking a second guess on an unconfirmed first one:
- "Is anything listening on this port at all?" (`ss -tlnp`)
- "Can I reproduce the failure locally, with no other systems involved?" (`curl` directly at the local address)
- "What does the service's own log say it's actually doing?" (`systemctl status` / `journalctl`)

Each of these either **confirms or eliminates** a specific hypothesis. My IPv6 theory felt plausible from the error message alone, but a thirty-second `ss` check eliminated it instantly and pointed straight at the real cause — a port mismatch, not a protocol mismatch. This is the actual debugging skill worth carrying forward: **prefer a command that produces a fact over a change that produces a hope.**

---

## Part 6: The architecture, once it actually worked

```
Internet
   ↓
Cloudflare edge (has real public IPs — I don't need one)
   ↓  (relayed down an outbound-initiated tunnel connection)
cloudflared, running on srv1
   ↓  (forwards to a fixed local address/port)
Caddy, listening on 22112002.xyz:80
   ↓  (routes internally, based on which hostname was requested)
/var/www/22112002.xyz/  →  my portfolio's files
```

Adding another site later — a client's domain, or a subdomain like `blog.22112002.xyz` — follows the exact same repeatable pattern every time:
1. A new folder under `/var/www/`
2. A new short block in the Caddyfile, with its own explicit `:80`
3. One `cloudflared tunnel route dns homelab <newdomain>` command (creates the DNS record automatically — no A/AAAA record possible or needed, since I have no public IP)
4. One new `hostname:` entry in the tunnel's ingress list, all still pointing at the same Caddy port
5. `sudo caddy validate --config /etc/caddy/Caddyfile` before reloading, to catch mistakes before they go live
6. `sudo systemctl reload caddy` and `sudo systemctl restart cloudflared`

None of these steps touch each other's territory — the tunnel never needs to know anything changed inside Caddy, and Caddy never needs to know anything about how traffic reached it in the first place. That separation of concerns is the entire point of putting a reverse proxy in front, instead of wiring the tunnel directly at each site individually.

---

## Part 7: Glossary additions for this phase

- **Reverse proxy** — a program that sits between incoming requests and one or more backend services, deciding which backend should handle each request (commonly by looking at the requested hostname), so the outside world only ever needs to know about one address.
- **Caddyfile** — Caddy's configuration file format, designed to be short and readable; each domain gets its own `{ }` block describing how to handle requests for it.
- **`ss -tlnp`** — a command that lists which programs are currently listening for network connections, and on which ports; the fastest way to answer "is anything actually running on port X?" with certainty instead of assumption.
- **Loopback address (`127.0.0.1` / `::1`)** — a special address a machine uses to talk to itself; `127.0.0.1` is the IPv4 version, `::1` is the IPv6 version. Traffic to either never leaves the machine, so it's unaffected by CGNAT or any router-level restriction.
- **Config validation** — checking a configuration file's syntax and structure *before* applying it, so a mistake gets caught with a clear error message instead of silently breaking a running service.
- **Evidence-based debugging** — forming a hypothesis from available symptoms, then testing that specific hypothesis with a targeted command before acting on it, rather than applying a fix and hoping it addresses the real cause.
