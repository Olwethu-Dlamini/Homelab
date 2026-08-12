# Hosting a New Site on a Subdomain — Full Guide

Based on setting up `lms.22112002.xyz` on srv1 (Debian 13, Caddy, Cloudflare Tunnel, MariaDB, PHP-FPM).

---

## 1. The Big Picture

You're behind CGNAT (no public IP), so you can't just point DNS at your server's IP like normal hosting. The workaround is a **Cloudflare Tunnel**: `cloudflared` runs on your VM and holds an *outbound* connection to Cloudflare. Outbound works even with no public IP. Cloudflare then routes incoming traffic for your domain through that tunnel to your local services.

Chain for every request:

```
Browser → Cloudflare → Tunnel → Caddy (port 80) → PHP-FPM (socket) → MariaDB
```

Every piece in that chain has to be told about the new hostname separately. That's the core lesson: adding a subdomain isn't one step, it's four small steps across four different systems.

---

## 2. Redo This Yourself — Step by Step

### Step 1: DNS (Cloudflare side, via CLI)

```bash
cloudflared tunnel route dns <tunnel-id> newsub.yourdomain.xyz
```

This creates a CNAME pointing your subdomain at your tunnel. No dashboard work needed if you manage the tunnel via config file. Verify in Cloudflare dashboard → DNS if you want to see it.

### Step 2: Tunnel ingress rule

Edit `/etc/cloudflared/config.yml`. Add your hostname **above** the catch-all 404 line — order matters, rules are checked top-down:

```yaml
ingress:
  - hostname: yourdomain.xyz
    service: http://127.0.0.1:80
  - hostname: newsub.yourdomain.xyz
    service: http://127.0.0.1:80
  - service: http_status:404
```

Restart: `sudo systemctl restart cloudflared`. Check `systemctl status cloudflared` for `active (running)`.

### Step 3: Caddy site block

Caddy is what actually decides what to serve for each hostname. Add a new block to `/etc/caddy/Caddyfile`, matching your existing style (yours uses `auto_https off` + explicit `:80`):

```caddyfile
newsub.yourdomain.xyz:80 {
	root * /var/www/newsub.yourdomain.xyz
	php_fastcgi unix//run/php/php8.4-fpm.sock
	file_server
	encode gzip
}
```

Leave out `php_fastcgi` if it's a static site. Reload: `sudo systemctl reload caddy`.

### Step 4: PHP (if the site needs it)

```bash
sudo apt install php-fpm php-mysql
ls /run/php/          # confirm the exact socket filename — versions drift (8.3 vs 8.4)
```

Match that exact socket name in the Caddy block. This was the actual bug we hit — copy-pasted an old `8.3` socket path when the install gave `8.4`. Silent failure, no obvious error. Always verify, don't assume the version.

### Step 5: Database

```bash
sudo mariadb
```

```sql
CREATE DATABASE app_db;
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'a_real_password';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;
```

### Step 6: Wire the app to the database

If the app reads DB credentials from environment variables (common pattern, `getenv('DB_HOST')` etc.), set them at the **PHP-FPM pool level**, not in a `.env` file the web server won't see:

```bash
sudo nano /etc/php/8.4/fpm/pool.d/www.conf
```

Add at the bottom:
```
env[DB_HOST] = localhost
env[DB_PORT] = 3306
env[DB_NAME] = app_db
env[DB_USER] = app_user
env[DB_PASS] = a_real_password
```

Restart: `sudo systemctl restart php8.4-fpm`.

### Step 7: Import schema and check hardcoded URLs

```bash
mariadb -u app_user -p app_db < schema.sql
grep -rn "localhost" /var/www/newsub.yourdomain.xyz/ --include=*.php
```

Any app config with a hardcoded `APP_URL` or base path pointing at `localhost:8000` or similar needs updating to your real domain — otherwise redirects and generated links break in production even though the page itself loads fine.

### Step 8: Test at every layer, don't skip to the end

```bash
curl -I https://newsub.yourdomain.xyz          # check status code first
curl -v https://newsub.yourdomain.xyz/test.php # verbose, to see TLS + headers
```

If something's broken, check logs in this order — cheapest info first:
1. `curl -v` output (did the request even reach the server, what status came back)
2. `sudo journalctl -u caddy --no-pager -n 30` (did Caddy route it correctly)
3. PHP-FPM log — but only useful if `catch_workers_output = yes` is uncommented in `/etc/php/8.4/fpm/pool.d/www.conf`. By default PHP errors from workers are swallowed silently. This was the second real bug we hit — no errors anywhere until this was turned on.
4. Direct database login test, bypassing PHP entirely: `mariadb -u app_user -p app_db` — isolates "is this a PHP problem or a MySQL problem."

---

## 3. Databases — the parts worth actually understanding

**What a database user actually is:** a set of credentials plus a grant (a permission scope). `CREATE USER` makes the identity, `GRANT` decides what it's allowed to touch. Without a grant, a valid login can still do nothing.

**`localhost` vs `127.0.0.1` — this bit us today.** They look interchangeable but MySQL/MariaDB treats them as different connection paths:
- `localhost` → connects via a local Unix socket file
- `127.0.0.1` → connects via actual TCP, even though it's still "local"

A user created as `'app_user'@'localhost'` will NOT be able to log in if your app config defaults to `127.0.0.1` — you'll get an access-denied error that looks identical to a wrong password, even though the password's fine. If you ever see access-denied errors and you're sure the password is right, check this first.

**Root with no password isn't automatically unsafe** — Debian's MariaDB uses `unix_socket` auth for root by default, meaning "you're root on this box" is the credential, not a password. That's fine because there's no separate secret to leak. But your app's own database user is different: it authenticates over a real password stored in a config file, readable by anything that compromises the app (SQL injection, leaked config, bad backup). Never leave that one blank, even on localhost-only setups.

**Schema files (`schema.sql`)** are just SQL `CREATE TABLE` statements bundled into one file, usually with `CREATE TABLE IF NOT EXISTS` and foreign keys between them. Order can matter — a table with a foreign key to another table must be created after that table exists, so if an import throws an error, check whether it's a real syntax problem or just an ordering issue.

**Basic mental model for designing your own schema**, if you're building from scratch next time:
- One table per "thing" (users, courses, enrollments)
- Every table gets a primary key (usually an auto-incrementing `id`)
- Relationships between tables use foreign keys (e.g. `enrollments.user_id` references `users.id`)
- Start with 3-4 tables max, get it working, add complexity only when you actually need it

---

## 4. Advantages of This Setup (native Caddy + PHP-FPM + MariaDB, no Docker)

- **Fewer moving parts to debug.** Everything runs as a normal systemd service. When something breaks, `systemctl status` and `journalctl` tell you directly — no container networking layer to also suspect.
- **Lower resource overhead** on modest hardware — no container runtime tax, useful on a homelab box that isn't a dedicated server.
- **You already understand this stack** — no new tooling to learn on top of the actual task.
- **Direct file access** — editing a config or a PHP file is just editing a file, no exec-into-container step.

## 5. Disadvantages

- **No isolation.** Every site on this box shares the same PHP-FPM pool config, same MariaDB instance, same OS-level dependencies. A bad `apt upgrade` or a misconfigured pool file can affect every site at once, not just one.
- **Version drift risk.** Today's bug (wrong PHP socket version hardcoded) happens because there's no per-app "sealed" environment — each site depends on whatever's globally installed on srv1 at that moment.
- **Harder to move.** If you ever want to migrate this to another box or scale it, there's no single portable unit to copy — you'd be recreating Caddy blocks, PHP pool settings, and DB dumps by hand.
- **One misbehaving app can take others down** — e.g. a PHP-FPM pool crash-looping affects every PHP site on the server, not just the LMS.

## 6. Docker Compose Alternative (the files sitting in your repo, unused)

Since the LMS repo already ships a `Dockerfile` and `docker-compose.yml`, it's built to be portable. The tradeoff going that route instead would have been:

**Advantages:** isolated dependencies per app (its own PHP version, its own extensions), easy to tear down and rebuild cleanly, one `docker-compose.yml` describes the whole stack (app + DB) so it's reproducible on any machine, easier to eventually move off this box.

**Disadvantages:** another layer to learn/debug (container networking, volumes, port mapping), slightly more RAM overhead, and you'd still need Caddy on the host to reverse-proxy into the container — so it doesn't remove Caddy from the picture, it just moves PHP and MySQL inside containers.

**Recommendation:** for one-off internal tools like this LMS, native is fine and what you already know. If you start running many different apps with conflicting PHP/dependency version needs on the same box, that's the point where Docker earns its complexity — each app gets its own contained environment instead of everything fighting over one global PHP install.

## 7. Other Recommendations Going Forward

- **Delete test/debug files immediately after use** — `phpinfo()` and DB test scripts left in a public web root leak server internals and credentials. We deleted both today; make it a reflex, not an afterthought.
- **Turn on `catch_workers_output = yes`** in every PHP-FPM pool by default from now on — it costs nothing and saves you from silent failures like the one we debugged today.
- **Never hardcode `localhost:PORT` or similar dev URLs in app configs** — use environment variables or config that adapts per environment (dev vs production), so you don't have to hunt down and fix redirect bugs after deploy.
- **Set up a backup routine for MariaDB** before this LMS holds real data — a simple cron job with `mysqldump` to a separate disk/location is enough for a homelab, and needs to exist before something breaks, not after.
- **Consider a staging subdomain** (`staging.22112002.xyz` or similar) for testing config changes to Caddy/PHP before they hit a subdomain people actually use — cheap insurance against exactly the kind of trial-and-error we just did live on the real hostname.
