# Giving n8n a Public Doorbell, Without Handing Out the House Keys

Date: 21 October 2026

The 16GB upgrade ([part two](ram-upgrade-16gb.md)) was for exactly this: a second VM, `ai`, that runs my automation, n8n plus a local AI model, kept apart from `srv1`, which hosts my websites. Until today `ai` was private on purpose. You could only reach it over Tailscale.

Today that stopped being enough. I'm doing an n8n course, and its checker tests your work by sending a request to your webhook **from the course's own servers**. My webhook URL was `http://ai:5678/webhook/...`, and `ai` is a name that only exists inside my tailnet. The internet had never heard of it.

The fix took one DNS record, one tunnel line and one Caddy block on `srv1`, plus one setting on `ai`. It cost `ai` no extra memory. As usual, the mistakes and the reasoning are kept in.

---

## Part 0: The starting symptom

Sending the request from Postman's web app:

```
Cloud Agent Error: Couldn't resolve host. Make sure the domain is publicly
accessible or select a different agent.
```

n8n itself was fine. From my laptop, on the tailnet, `curl http://ai:5678/healthz` answered `{"status":"ok"}`. The clue is the words **Cloud Agent**. Postman's web app doesn't send from my machine: it sends from Postman's servers, out on the internet. Same for the course checker. Neither of them is on my tailnet, so `ai` doesn't resolve for them.

**The lesson:** "it works for me" and "it works for them" are different tests when *me* is on a private network. Ask where the request actually starts from.

(The quick workaround, if you only need to test: switch Postman to the **Desktop Agent**, which sends from your own machine. That doesn't help the course checker, though.)

---

## Part 1: Choosing how to go public

I already know the CGNAT story from the [first guide](homelab-guide.md): my ISP gives me no public IP, so "forward a port on the router" is off the table. Three options were left:

| Option | What it gives | Why I didn't / did pick it |
|---|---|---|
| **Tailscale Funnel** | A free public HTTPS URL, using the Tailscale I already run | Only on a `*.ts.net` name. I want my tools on my own domain |
| **A new `cloudflared` container on `ai`** | My own domain, a second tunnel | Works, but it's one more service on the VM where I'm budgeting every MB |
| **Reuse srv1's tunnel and Caddy** | My own domain, nothing new running | ✅ srv1 already has a Cloudflare Tunnel and Caddy serving `22112002.xyz`, the LMS and my mail. Both VMs sit on the same LAN |

The third one is the pattern from [the multi-site Caddy guide](multi-site-hosting-caddy.md), one step further: the tunnel always forwards to Caddy, and Caddy decides what goes where. Now "where" can also be **another machine on my LAN**, not only something running on srv1 itself.

### The decision that matters more than the tool: what exactly goes public

n8n is two things on one port:

- the **editor**: the screen where I build workflows, log in, and keep credentials (my bot tokens, mail settings, API keys)
- the **webhooks**: URLs under `/webhook/...` that outside things call to trigger a workflow

Only the second needs to be public. If the editor is on the internet, a weak password or one n8n security bug exposes everything. So the plan: **publish only `/webhook/*` and `/webhook-test/*`, and answer 404 for every other path.** This is the same trick my mail block already uses to hide Stalwart's `/admin`.

```
Course checker / Postman (internet)
        │  https://n8n.22112002.xyz/webhook/...
        ▼
Cloudflare edge          ← the HTTPS padlock lives here
        │  the tunnel srv1 already keeps open (outbound, so CGNAT doesn't matter)
        ▼
srv1: cloudflared → Caddy :80   ← the bouncer
        │  /webhook/* or /webhook-test/*  → let through
        │  anything else (/, /signin, /rest...) → 404, stops here
        ▼
ai: n8n  http://10.0.0.31:5678   (plain http, inside my home network)
```

---

## Part 2: The four changes

Same lesson as the [subdomain guide](subdomain-hosting-guide.md): a new hostname isn't one step, it's one small step in each system on the path. This time there were four.

### Step 0: Back up before touching anything live

That tunnel carries my site, the LMS and my mail. If I break its config, all of them go down.

```bash
ts=$(date +%Y%m%d-%H%M%S)
sudo cp -a /etc/cloudflared/config.yml /etc/cloudflared/config.yml.bak-$ts
sudo cp -a /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak-$ts
```

### Step 1: DNS (Cloudflare, from srv1)

```bash
T=$(sudo awk '/^tunnel:/{print $2}' /etc/cloudflared/config.yml)
cloudflared tunnel route dns "$T" n8n.22112002.xyz
```
```
INF Added CNAME n8n.22112002.xyz which will route to this tunnel
```

The `awk` just reads my tunnel's ID out of the config file, so I don't have to copy it by hand.

### Step 2: Tunnel ingress (srv1)

In `/etc/cloudflared/config.yml`, one more hostname, **above** the 404 catch-all, because rules are read top to bottom:

```yaml
  - hostname: n8n.22112002.xyz
    service: http://127.0.0.1:80
  - service: http_status:404
```

It points at Caddy like every other hostname. The tunnel doesn't know or care that n8n is on a different machine.

### Step 3: The Caddy block (srv1): this is the bouncer

Appended to `/etc/caddy/Caddyfile`:

```caddyfile
# n8n on the ai VM (10.0.0.31). Only webhooks are public; the editor, login
# and REST API stay on Tailscale (http://ai:5678). Every webhook checks a
# secret header in n8n itself.
n8n.22112002.xyz:80 {
	@hooks path /webhook/* /webhook-test/*
	handle @hooks {
		reverse_proxy 10.0.0.31:5678
	}
	handle {
		respond 404
	}
}
```

How to read it:

- `@hooks path ...` defines a **named matcher** called `hooks`: "requests whose path starts with `/webhook/` or `/webhook-test/`".
- `handle @hooks { ... }`: those requests get forwarded to n8n on `ai`.
- `handle { respond 404 }`: everything else gets a 404.
- `handle` blocks are mutually exclusive and tried in order, so a request is either forwarded or refused, never both.

### Step 4: Validate, *then* apply

The multi-site guide taught me to validate before reloading, so I validated both files:

```bash
cloudflared tunnel --config /etc/cloudflared/config.yml ingress validate
# OK
cloudflared tunnel --config /etc/cloudflared/config.yml ingress rule https://n8n.22112002.xyz/webhook/x
# Matched rule #7  hostname: n8n.22112002.xyz
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
# Valid configuration
```

`ingress rule <url>` is a nice one I hadn't used before: it tells you **which rule a URL would hit**, without sending any traffic.

Then:

```bash
sudo systemctl reload caddy        # reload: re-reads config, no downtime
sudo systemctl restart cloudflared # restart: the tunnel drops and reconnects
```

Right after the restart my main site answered **530** on the first check. That's Cloudflare saying "I can't reach your origin", which I know from the [RAM upgrade outage](ram-upgrade-outage.md). Two seconds later it was 200 again: the tunnel had re-registered its four connections. **A restart of `cloudflared` is a short outage for every site on the tunnel**, so I do it once, after validating, not while experimenting.

### Step 5: Tell n8n its public address (on `ai`)

n8n prints a URL on every webhook node, built from a setting called `WEBHOOK_URL`. Mine said `http://ai:5678/`. I changed my stack's compose file so it reads:

```yaml
- WEBHOOK_URL=${N8N_WEBHOOK_URL:-http://${N8N_HOST:-ai}:5678/}
- N8N_PROXY_HOPS=1
```

and put `N8N_WEBHOOK_URL=https://n8n.22112002.xyz/` in the server's `.env`. Now the webhook node shows `https://n8n.22112002.xyz/webhook/...`, ready to copy into the course.

Two things that confused me at first:

- **This doesn't make n8n speak HTTPS.** n8n still listens on plain http. `WEBHOOK_URL` only changes the address it *shows*. The HTTPS is Cloudflare's, the same as for my portfolio site.
- **`N8N_PROXY_HOPS=1`** tells n8n that public requests come through one proxy (Caddy), so it should read the caller's real address from the `X-Forwarded-For` header Caddy adds.

The full setup for the automation VM, including this change, lives in my [DDOService](https://github.com/Olwethu-Dlamini/DDOService) repo: `stack/`, plus `docs/PUBLIC-WEBHOOKS.md`.

---

## Part 3: Testing from the outside

The point is what *strangers* can reach, so the test has to come from outside the tailnet, by hostname:

```
GET  /                    404   ← Caddy: editor hidden
GET  /signin              404
GET  /rest/settings       404   ← n8n's internal API hidden
GET  /api/v1/workflows    404   ← n8n's public API hidden
GET  /healthz             404
POST /webhook/work-email  403   ← n8n: reachable, but no secret header
POST /webhook/telegram    403
POST /webhook/whatsapp    403
```

That **403** matters as much as the 404s. A webhook is now a URL anyone on the internet can hit. What stops a stranger is the **Header Auth** on each webhook node: the request must carry one specific header with one specific value, or n8n refuses it. All my webhooks already had it. One with *Authentication: None* would be open to the world.

I turned this into a script (`stack/public-webhook-check.sh` in DDOService), so after any change I can prove the editor is still hidden and the webhooks still refuse a call without their secret.

---

## Part 4: Then the course still failed, twice

Getting the request *to* n8n was only half of it. After that, every error came from inside my workflow.

### Mistake 1: a long command pasted into SSH lost its URL

```
oll@ai:~$ wget -qO deploy.sh
wget: missing URL
```

The terminal broke my paste and the URL never arrived. This bit me before when linking WhatsApp. The fix that works: download the file first (`wget` = "web get", it saves a URL to a file), then run a short command. Or copy files from the laptop with `scp file oll@ai:~/`.

### Mistake 2: `No Respond to Webhook node found in the workflow` (500)

My Webhook node was set to **Respond: Using 'Respond to Webhook' node**. That means "a node later in the workflow will send the reply". I hadn't added that node yet. n8n checks for it **before** it checks the secret header, so this 500 told me nothing about whether my header was right.

**Fix:** I added a *Respond to Webhook* node to each branch of my *If* node: `RespondSuccess` answers 200 with `{"status": "accepted", ...}`, and `RespondValidationError` answers 400 when a required field is missing. (Setting the Webhook to *Respond: Immediately* also works, if you don't need to choose the reply.)

### Mistake 3: Postman had no "Header Auth" option

n8n's *Header Auth* isn't a special Postman auth type. It just means "send this one header". Two ways in Postman:

- **Headers** tab → Key = the credential's *Name* from n8n, Value = its *Value*
- or **Authorization** tab → Type *API Key* → same name and value → *Add to: Header*

### Mistake 4: `Wrong type: '0' is a number but was expecting a string`

The course's checker sent:

```json
{"order_id": "VALIDURL-TEST", "customer_id": "VALIDURL-TEST", "total": 0}
```

My *If* node checked every field with **String → is not empty**, and strict type checking was on. `total` is the **number** `0`, not the text `"0"`, so n8n stopped the workflow with an error instead of comparing. No *Respond* node ran, so the course got a 500 again.

I only found this by **opening the failed run in n8n's Executions list**. It shows the exact body the checker sent and the exact error. The course's own message only said "Status Code: 500".

**Fix:** I changed the `total` condition to **Number → is not empty**. The type now matches what arrives, and a total of 0 still passes. (Turning on *Convert types where required* would also have worked.)

After that the course passed. ✅

---

## Part 5: What I actually learned

1. **Ask where the request comes from.** "Cloud Agent" meant Postman's servers, not my laptop. A private name like `ai` only resolves inside the tailnet.
2. **Publish the doorbell, not the house.** Only `/webhook/*` is public; the editor, login and API answer 404. The reverse proxy is the right place for that rule.
3. **Reuse the path you already have.** srv1's tunnel and Caddy took one more line each. A second tunnel on `ai` would have cost memory and added one more thing to break.
4. **Validate before you restart anything that carries other people's traffic.** `cloudflared ... ingress validate`, `ingress rule <url>`, `caddy validate`. Then restart once.
5. **`WEBHOOK_URL` changes the URL n8n shows, not how it listens.** HTTPS comes from Cloudflare.
6. **Every public webhook needs a secret.** 404 hides the editor; 403 is what protects the webhooks.
7. **A public URL only gets the request to the door.** When it still fails, open the execution and read what came in.
8. **Know your types.** `0` and `"0"` are different things, and strict validation will stop on the difference.

---

## Part 6: Quick reference

```bash
# --- on srv1 ---
# back up
ts=$(date +%Y%m%d-%H%M%S)
sudo cp -a /etc/cloudflared/config.yml /etc/cloudflared/config.yml.bak-$ts
sudo cp -a /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak-$ts

# DNS record for the new hostname
cloudflared tunnel route dns "$(sudo awk '/^tunnel:/{print $2}' /etc/cloudflared/config.yml)" n8n.22112002.xyz

# edit: tunnel ingress line above the 404, and the Caddy block (Part 2)

# validate, then apply
cloudflared tunnel --config /etc/cloudflared/config.yml ingress validate
cloudflared tunnel --config /etc/cloudflared/config.yml ingress rule https://n8n.22112002.xyz/webhook/x
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl reload caddy
sudo systemctl restart cloudflared
sudo journalctl -u cloudflared -n 20 --no-pager   # wait for 4x "Registered tunnel connection"

# --- on ai ---
# N8N_WEBHOOK_URL=https://n8n.22112002.xyz/ in /opt/automation/.env, then redeploy

# --- from anywhere ---
curl -s -o /dev/null -w '%{http_code}\n' https://n8n.22112002.xyz/                    # 404
curl -s -o /dev/null -w '%{http_code}\n' -X POST https://n8n.22112002.xyz/webhook/x   # 404 or 403
```

**Undo:** delete the `n8n.22112002.xyz` Caddy block and reload Caddy. Every public request then gets 404 straight away. For a full undo, also remove the tunnel line, restart `cloudflared` and delete the DNS record in the Cloudflare dashboard.

**Measured on 21 October:** a public webhook call from my laptop took a median of **0.153s**, against **0.061s** over Tailscale (10 calls each, through Cloudflare's Johannesburg edge). On srv1, Caddy used 64MB and cloudflared 24MB, both already running for the websites. On `ai`: nothing new.

---

## Glossary additions from today

- **Webhook**: a URL that one system calls to tell another that something happened, like a doorbell for software.
- **Test vs production webhook**: n8n gives every webhook two URLs. `/webhook-test/...` only works for about two minutes after you click *Listen for test event* in the editor. `/webhook/...` works whenever the workflow is active.
- **Header Auth**: the request must carry one named header (for example `X-Api-Key`) with one exact value. Wrong or missing → 403.
- **Named matcher (Caddy)**: `@name path /a/* /b/*`, a reusable condition you can attach to `handle`, `respond` or `reverse_proxy`.
- **`WEBHOOK_URL`**: n8n's setting for the public base address it prints on webhook nodes.
- **`X-Forwarded-For`**: a header a proxy adds with the original caller's IP address. Without it, n8n would think every request came from srv1.
- **HTTP 403 vs 404**: 404 = "nothing here" (Caddy hiding the editor, or n8n with no such webhook). 403 = "something is here, but you're not allowed" (n8n refusing a missing secret).
- **Strict type validation (n8n If node)**: compares values only when they're the declared type. A number checked as a string is an error, not a "false".
