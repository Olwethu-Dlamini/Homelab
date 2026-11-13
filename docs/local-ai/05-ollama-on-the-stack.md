# Ollama on the stack

Ollama already runs on `ai` as one container in the `/opt/automation` Compose
stack. This page covers the settings worth changing, how to switch the default
model away from Mistral, how to turn thinking off, and how to call the API.

## How it's wired today

From the stack's `docker-compose.yml` (in the DDOService repo, `stack/`):

```yaml
ollama:
  image: ollama/ollama:latest
  container_name: ollama
  # Ollama has no auth, so it is only published on the VM itself.
  ports:
    - "127.0.0.1:11434:11434"
  volumes:
    - ollama_models:/root/.ollama
  environment:
    - OLLAMA_KEEP_ALIVE=${OLLAMA_KEEP_ALIVE:-2m}
    - OLLAMA_MAX_LOADED_MODELS=1
    - OLLAMA_NUM_PARALLEL=1
    # Since 13 Nov 2026: stops parallel model reads thrashing the HDD (chapter 3).
    - GOMAXPROCS=${OLLAMA_GOMAXPROCS-1}
  cpus: ${OLLAMA_CPUS:-3}
  mem_limit: ${OLLAMA_MEM:-6g}
```

Three things are already right and should stay:

- **Port bound to `127.0.0.1`.** Ollama has no login. Anyone who can reach port
  11434 can use and delete your models. Other containers reach it as
  `http://ollama:11434` on the Compose network; nothing outside the VM can.
- **One request at a time** (`OLLAMA_NUM_PARALLEL=1`). Parallel requests
  share the same cores and memory bus, so each one gets slower, and each
  reserves its own context memory.
- **A hard memory cap.** If a model is too big, Ollama fails to load it instead
  of pushing the whole VM into swap.

## Settings to add

`deploy.sh` downloads the compose file from the DDOService repo on every run,
so these changes go **in that repo's `stack/docker-compose.yml`**, then a
redeploy. Proposed `environment:` block:

```yaml
  environment:
    - OLLAMA_KEEP_ALIVE=${OLLAMA_KEEP_ALIVE:-2m}
    - OLLAMA_MAX_LOADED_MODELS=${OLLAMA_MAX_LOADED_MODELS:-1}
    - OLLAMA_NUM_PARALLEL=1
    # Default context for requests that don't set num_ctx (Ollama's own default is 4096).
    - OLLAMA_CONTEXT_LENGTH=${OLLAMA_CONTEXT_LENGTH:-4096}
    # Halve the KV cache's memory; needs flash attention, so force it on.
    - OLLAMA_FLASH_ATTENTION=1
    - OLLAMA_KV_CACHE_TYPE=q8_0
    # Local only: no Ollama cloud models or web search.
    - OLLAMA_NO_CLOUD=1
```

What each one buys:

| Setting | Why |
|---|---|
| `OLLAMA_MAX_LOADED_MODELS` from `.env` | Set it to `2` when RAG starts, so the chat model and the embedding model can both stay loaded instead of swapping on every question ([chapter 7](07-n8n-and-rag.md)) |
| `OLLAMA_CONTEXT_LENGTH` | Makes the context explicit. Raise to `8192` for RAG prompts, after checking memory ([chapter 2](02-sizing-a-model.md#context-and-the-kv-cache)) |
| `OLLAMA_FLASH_ATTENTION=1` + `OLLAMA_KV_CACHE_TYPE=q8_0` | KV cache at about half the memory, with very little quality loss, per Ollama's FAQ |
| `OLLAMA_NO_CLOUD=1` | Guarantees every prompt stays on the box |

All of these come from [Ollama's FAQ](https://docs.ollama.com/faq).

## Switching the default model

The stack's `.env` holds the model `deploy.sh` pulls, plus Ollama's limits.
On `ai`:

```bash
sudo nano /opt/automation/.env
```

```ini
OLLAMA_MODEL=qwen3.5:4b
OLLAMA_KEEP_ALIVE=2m
OLLAMA_CPUS=4          # chapter 3, step 1
OLLAMA_MEM=6g          # 8g only for 9B models (chapter 3)
```

Save, then redeploy (it keeps the `.env` and pulls the new model):

```bash
sudo bash deploy.sh
```

Then pull the others you want from [chapter 4](04-the-best-models.md):

```bash
sudo docker exec ollama ollama pull granite4.2:3b
sudo docker exec ollama ollama pull qwen3-embedding:0.6b
sudo docker exec ollama ollama list
```

**n8n doesn't follow `OLLAMA_MODEL` by itself.** Each Ollama node in a
workflow names its model. Open every workflow that uses `Ollama (local)` and
change `mistral` to the new tag. Only after the last one is switched:

```bash
sudo docker exec ollama ollama rm mistral    # frees 4.4GB of disk
```

## A first test

```bash
sudo docker exec -it ollama ollama run qwen3.5:4b --verbose --think=false \
  "Reply with one word, payment, question or other: 'Hi, I paid R350 this morning'"
```

`--verbose` prints timings after the answer. The two lines that matter:

```
prompt eval rate:     NN tokens/s     ← reading speed
eval rate:            NN tokens/s     ← writing speed
```

Run it twice. The first run includes loading from disk; the second shows the
real speed. Write both numbers down: [chapter 8](08-benchmark-and-troubleshoot.md)
has a table for them.

While it runs, in a second terminal:

```bash
sudo docker exec ollama ollama ps
```

`PROCESSOR` should say `100% CPU` and `SIZE` is the model's real memory use,
context included.

## Thinking on or off

Qwen3.5, Granite 4.2, LFM3 and others can think before answering
([chapter 2](02-sizing-a-model.md#thinking-models)). Each model declares what
it supports. Ask Ollama:

```bash
curl -s http://localhost:11434/api/show -d '{"model":"qwen3.5:4b"}' \
  | python3 -c 'import json,sys; print(json.load(sys.stdin).get("thinking"))'
```

`default` is what happens when you say nothing; `values` lists what you may
ask for (`true`/`false`, or levels like `low`/`medium`/`high`).

To switch it off:

| Where | How |
|---|---|
| `ollama run` | `--think=false`, or type `/set nothink` inside a session |
| Native API (`/api/chat`, `/api/generate`) | `"think": false` in the request |
| n8n | See [chapter 7](07-n8n-and-rag.md#thinking-in-n8n) |

When thinking is on, the API returns the reasoning in a separate `thinking`
field, so it never pollutes the answer you parse.

## Keep-alive: memory vs waiting

`OLLAMA_KEEP_ALIVE=2m` unloads a model two idle minutes after its last request.
That keeps ~4GB free most of the day, at the price of a load (seconds to tens
of seconds on the HDD) on the next request.

- **Leave it at 2m** while models are only used now and then.
- **Preload before a batch** so the first item doesn't pay the load:

  ```bash
  curl -s http://localhost:11434/api/generate -d '{"model":"qwen3.5:4b","keep_alive":"30m"}'
  ```

- **Unload now**: `sudo docker exec ollama ollama stop qwen3.5:4b`.

A `keep_alive` in a request overrides the server default for that model.

## The API, for scripts and HTTP Request nodes

Inside the VM the API is at `http://localhost:11434`; from other containers
(n8n, Open WebUI) it's `http://ollama:11434`.

### Chat, with a fixed JSON answer

`format` takes a JSON schema, and the model is forced to match it. This is the
most reliable way to get data out of a small model:

```bash
curl -s http://localhost:11434/api/chat -d '{
  "model": "granite4.2:3b",
  "stream": false,
  "think": false,
  "messages": [{"role": "user",
    "content": "Extract name and amount: Thandi paid R350 for DStv on Friday"}],
  "format": {
    "type": "object",
    "properties": {"name": {"type": "string"}, "amount": {"type": "number"}},
    "required": ["name", "amount"]
  },
  "options": {"temperature": 0}
}'
```

The response's `message.content` is the JSON. It also reports
`prompt_eval_count`, `prompt_eval_duration`, `eval_count` and `eval_duration`
(nanoseconds), which is all you need to compute speeds.

### OpenAI-compatible endpoint

Tools written for OpenAI's API work by changing the base URL to
`http://ollama:11434/v1` (any API key string; Ollama ignores it). Use the
native `/api/chat` when you need Ollama-specific options like `think` or
`keep_alive`.
