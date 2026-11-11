# A chat window: Open WebUI

n8n uses the models in the background. For asking questions yourself, a chat
page is nicer than `ollama run` over SSH. [Open WebUI](https://docs.openwebui.com/)
is a ChatGPT-style web app that talks to Ollama. This page adds it to the stack
in the smallest form, reachable only over Tailscale.

It's optional. If you only use the models from n8n, skip this chapter and save
the memory.

## What it costs

Open WebUI's **slim** image is about 200MB to download, against 1.6GB for the
standard one. It leaves out its own embedding and speech models, which this
VM couldn't afford to run anyway. Its docs measured **about 300MB of RAM at
idle**.

| | Before | After |
|---|---|---|
| Sum of container caps (Ollama at 6GB) | ~10GB | ~10.7GB |
| Sum of container caps (Ollama at 8GB) | ~12GB | ~12.7GB, more than the VM's 12GB |

Caps aren't usage: today everything except a loaded model uses under 2GB. But
if you run a 9B model with the 8GB cap, watch `free -m` and swap after adding
this, or stop Open WebUI while big jobs run.

What slim **can't** do on its own: chat with uploaded documents (it would need
PostgreSQL with pgvector), voice, or local web search. Document Q&A on this box
belongs in n8n with Qdrant ([chapter 7](07-n8n-and-rag.md)). Slim is a chat
window, and that's all it needs to be here.

## Adding it to the stack

Like Ollama's settings, this goes in the DDOService repo's
`stack/docker-compose.yml`, then a redeploy. Port 3000 is already WAHA's, so
Open WebUI uses 8080, bound to the VM only:

```yaml
  open-webui:
    image: ghcr.io/open-webui/open-webui:slim
    container_name: open-webui
    # Tailscale Serve puts HTTPS in front; nothing else reaches it.
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      # Fixed key, or every recreate logs everyone out.
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY:?set WEBUI_SECRET_KEY in .env}
    volumes:
      - open_webui_data:/app/backend/data
    cpus: 1
    mem_limit: 768m
    depends_on:
      - ollama
    restart: unless-stopped
```

and under `volumes:` at the bottom:

```yaml
  open_webui_data:
```

The secret key goes in `/opt/automation/.env`:

```bash
echo "WEBUI_SECRET_KEY=$(openssl rand -hex 32)" | sudo tee -a /opt/automation/.env >/dev/null
```

(Or add `WEBUI_SECRET_KEY` to the list of secrets `deploy.sh` generates, next
to `QDRANT_API_KEY`, so a fresh install gets one automatically.)

Redeploy and check it answers:

```bash
sudo bash deploy.sh
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/health    # 200
```

## Reaching it: Tailscale only

Open WebUI has logins, but it's still a door into a machine that holds n8n's
credentials. It stays **off the public internet**: no Cloudflare Tunnel entry,
no Caddy block on srv1. Same rule as n8n's editor in the
[public webhooks write-up](../../n8n-public-webhooks.md).

`tailscale serve` gives it an HTTPS address that only exists inside the
tailnet:

```bash
# inside ai
sudo tailscale serve --bg --https=8443 http://127.0.0.1:8080
sudo tailscale serve status
```

Then open `https://ai.<your-tailnet>.ts.net:8443` from the laptop or the phone
(with the Tailscale app on). `--bg` makes it survive reboots.

If `serve` complains about certificates, turn on **MagicDNS** and **HTTPS
Certificates** in the Tailscale admin console under *DNS*, then try again.

## First login

1. The **first account you create becomes the admin**. Create yours straight
   away.
2. Then close the door: *Admin Panel → Settings → General → Enable New Sign
   Ups* → off. Open WebUI stores this setting in its own database, so the
   admin panel, not an environment variable, is the place to change it.
3. Under *Admin Panel → Settings → Connections*, the Ollama connection should
   show `http://ollama:11434`. The models you pulled appear in the model
   picker.

## Using it well on a CPU

- **Pick the model per chat.** Granite 4.2 3B or LFM3 for quick questions,
  Qwen3.5 4B for harder ones ([chapter 4](04-the-best-models.md)).
- **Keep chats short.** Every message resends the whole conversation, so the
  reading phase grows with each turn. Start a new chat for a new topic.
- **Thinking costs minutes.** If the model shows a long "thinking" block for a
  simple question, turn thinking off in the chat's advanced parameters if your
  version offers it, or use a model that answers directly.
- **Images work** with Qwen3.5 (drag a photo in), but an image becomes hundreds
  of prompt tokens. Expect tens of seconds before the answer starts.
- **The first message after a quiet spell is slow**: the model was unloaded
  after `OLLAMA_KEEP_ALIVE` and must load from the HDD
  ([chapter 5](05-ollama-on-the-stack.md#keep-alive-memory-vs-waiting)).

## Removing it

```bash
sudo tailscale serve --https=8443 off
sudo docker rm -f open-webui
```

Then delete the `open-webui` service and volume from the compose file in the
DDOService repo, so the next deploy doesn't bring it back. Its chats live in a
volume until you delete that too (`sudo docker volume ls` shows the exact
name, usually `automation_open_webui_data`).
