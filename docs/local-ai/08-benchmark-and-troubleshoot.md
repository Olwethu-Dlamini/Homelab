# Benchmarking and troubleshooting

Every speed in this series except Mistral's is an estimate. This page is how
to replace them with measurements, and what to check when something is slow or
broken.

## Measuring: the quick way

```bash
sudo docker exec -it ollama ollama run qwen3.5:4b --verbose --think=false "Say hello in five words."
```

After the answer, `prompt eval rate` is reading speed and `eval rate` is
writing speed. Run it twice; ignore the first run's numbers if the model had
to load.

## Measuring: the comparable way

To compare models fairly, send each one the same ~400-token prompt, once cold
(just loaded from disk) and once warm, and read Ollama's own timings from the
API. Save this as `~/bench.py` on `ai`:

```python
#!/usr/bin/env python3
"""Reading and writing speed per model, from Ollama's own timings.

    python3 bench.py qwen3.5:4b granite4.2:3b lfm3:1.7b
"""
import json
import sys
import urllib.error
import urllib.request

BASE = "http://localhost:11434"
TEXT = ("The homelab runs on one old desktop with an 8th-gen i5, 16GB of RAM and "
        "a spinning disk. Proxmox splits it into a VM for websites and a VM for "
        "automation, where n8n, Qdrant, WhatsApp and a local language model "
        "share 12GB. Without a graphics card, every word the model writes means "
        "reading all of its weights from RAM again. ") * 6
PROMPT = "Summarise this in two sentences:\n\n" + TEXT


def post(path, body):
    req = urllib.request.Request(BASE + path, json.dumps(body).encode(),
                                 {"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=900) as r:
        return json.load(r)


def chat(model, prompt):
    body = {"model": model, "stream": False, "think": False, "keep_alive": "5m",
            "messages": [{"role": "user", "content": prompt}],
            "options": {"temperature": 0}}
    try:
        return post("/api/chat", body)
    except urllib.error.HTTPError as e:
        if e.code != 400:
            raise
        del body["think"]          # model without a thinking switch
        return post("/api/chat", body)


def rate(count, ns):
    return f"{count / (ns / 1e9):.1f}" if count and ns else "-"


print(f"{'model':22} {'run':5} {'load s':>6} {'prompt':>6} {'read/s':>7} "
      f"{'answer':>6} {'write/s':>7}")
for model in sys.argv[1:]:
    post("/api/generate", {"model": model, "keep_alive": 0})   # unload: next run is cold
    for run in ("cold", "warm"):
        # A different first word per run, so the warm run can't reuse the
        # cold run's cached prompt.
        r = chat(model, f"Run {run}. {PROMPT}")
        print(f"{model:22} {run:5} {r.get('load_duration', 0) / 1e9:6.1f} "
              f"{r.get('prompt_eval_count', 0):6} "
              f"{rate(r.get('prompt_eval_count'), r.get('prompt_eval_duration')):>7} "
              f"{r.get('eval_count', 0):6} "
              f"{rate(r.get('eval_count'), r.get('eval_duration')):>7}")
    post("/api/generate", {"model": model, "keep_alive": 0})   # free the RAM again
```

Run it when no workflow is due (it holds the only Ollama slot):

```bash
python3 ~/bench.py qwen3.5:4b granite4.2:3b lfm3:1.7b
```

`read/s` and `write/s` on the warm row are the numbers to record. `load s` on
the cold row is the disk **only the first time a model loads after the VM
boots**. After that, Linux usually still has the file in its page cache, and
the "cold" load is much faster than the HDD. For a true disk number, empty the
cache first (harmless, it just refills):

```bash
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches >/dev/null
```

## Results

Fill in a row after each change ([chapter 3](03-the-ai-vm-in-proxmox.md)'s CPU
type and cores, or a new model), so you know what actually helped.

| Date | Model | CPU type | `OLLAMA_CPUS` | Cold load | Read tok/s | Write tok/s |
|---|---|---|---|---|---|---|
| 2026-10-07 | mistral (7B Q4_K_M) | ? | 3 | 48.1s | ~16 | ~3 |
| 2026-11-13 | mistral (7B Q4_K_M) | host (AVX2 seen) | ? | **213.3s**; 40.9s on the next load | ~17 | 3.1 |
| 2026-11-13 (later) | mistral (7B Q4_K_M), `GOMAXPROCS=1`, Ollama 0.40.16 | host (AVX2 seen) | ? | **63.5s**, page cache emptied first | ~18 | 3.1 |
| 2026-11-13 (bench.py) | mistral (7B Q4_K_M), `GOMAXPROCS=1`, Ollama 0.40.16 | host (AVX2 seen) | ? | 52.5s, part of the file still cached | **18.8** | **2.8** |
| | qwen3.5:4b | | | | | |
| | granite4.2:3b | | | | | |
| | lfm3:1.7b | | | | | |
| | qwen3.5:9b (8GB cap) | | | | | |

The 7 October row is from the stack's design notes, measured with one short
prompt, so its reading speed is rough. The 13 November row is one API request
(16 prompt tokens, 27 answer tokens) after the VM had been up 43 days: the cold
load took four times October's, see
[chapter 3](03-the-ai-vm-in-proxmox.md#4-the-disk-why-the-first-answer-is-slow).
The load right after it, with part of the file still cached, took 40.9s.
The "later" row is the same request after redeploying with `GOMAXPROCS=1` and
emptying the page cache: the cold load fell from 213.3s to 63.5s (about 70MB/s)
and writing speed stayed at 3.1 tok/s. The redeploy also upgraded Ollama from
0.40.7 to 0.40.16, so part of the gain may come from that.
The "bench.py" row is the comparable run from this page (570 prompt tokens,
about 110 answer tokens), the first real numbers for the table: warm, Mistral
reads at 18.8 tok/s and writes at 2.8 tok/s, so a request like this takes
about 70s once loaded. Its cold load of 52.5s came about 15 minutes after the
previous run with ~1.9GB still in the page cache, so it is not a disk number;
63.5s above is.

**n8n research agent (2026-11-14).** One question to the `Local research agent
(Ollama + SearXNG)` workflow (Chat Trigger, AI Agent, Ollama Chat Model with
`mistral`, SearXNG tool), sent to its chat webhook: "What is the latest stable
Proxmox VE version? Search the web." It took **201.5s** end to end, with Mistral
not loaded beforehand, so a cold load is included. The answer was 9.2, released
21 May 2026, with the proxmox.com download link: a fact newer than Mistral's
training, so it came from the search. The run has no per-node breakdown
because this n8n does not save successful production runs.

## Troubleshooting

| Symptom | Likely cause | Check / fix |
|---|---|---|
| Writing under ~2 tok/s on a 4B model | VM lacks AVX2 | `grep -o -w avx2 /proc/cpuinfo` inside `ai`; CPU type `host` ([ch. 3](03-the-ai-vm-in-proxmox.md#1-cpu-type-does-the-vm-see-avx2)) |
| Slow, and `ollama ps` shows the right model | Ollama capped at 3 CPUs; swap in use; another request ahead in the queue | `OLLAMA_CPUS` in `.env`; `free -m`; `sudo docker logs ollama --since 10m` |
| Long pause before any answer, then normal speed | Model loading from the HDD, or a long thinking phase | `load_duration` in the API response; `"think": false` ([ch. 5](05-ollama-on-the-stack.md#thinking-on-or-off)) |
| Cold load of minutes for a few GB (well under the HDD's ~90MB/s) | Ollama's parallel readers make the HDD seek; or another VM on the disk; or a failing disk | `GOMAXPROCS=1` for Ollama first ([ch. 3](03-the-ai-vm-in-proxmox.md#4-the-disk-why-the-first-answer-is-slow)), then `iostat -x 2` and `smartctl -a` on `pve` |
| Model fails to load, error mentions memory | Model + context bigger than `OLLAMA_MEM` | A smaller model, a smaller context, or `OLLAMA_MEM=8g` ([ch. 2](02-sizing-a-model.md#checklist-will-it-fit)) |
| Ollama container restarts during a request | Killed at its memory cap | `sudo docker inspect ollama --format '{{.State.OOMKilled}}'` → `true` means the same fix as above |
| `pull` fails asking for a newer Ollama | New model, old image | `sudo bash deploy.sh` pulls the latest `ollama/ollama` image |
| n8n: `ECONNREFUSED` to Ollama | Base URL says `localhost`, which inside n8n's container is n8n itself | Base URL `http://ollama:11434` (the `Ollama (local)` credential) |
| n8n node times out | CPU answers take minutes | Raise the node's timeout ([ch. 7](07-n8n-and-rag.md#2-an-http-request-node-straight-to-the-api)) |
| Answer ignores the end of a long prompt | Prompt longer than the context (default 4,096 tokens), so it was cut | Shorter prompt, or `num_ctx` / `OLLAMA_CONTEXT_LENGTH` 8192 |
| Output isn't valid JSON | Free-form answer, thinking text, high temperature | `format` with a JSON schema, `"think": false`, temperature 0 |
| Every RAG question takes an extra minute | Chat and embedding models swapping | `OLLAMA_MAX_LOADED_MODELS=2` ([ch. 7](07-n8n-and-rag.md#settings-that-make-it-work-on-this-box)) |
| Disk filling up | Old models still stored | `sudo docker exec ollama ollama list`, then `ollama rm <tag>` |

## Quick reference

```bash
# --- on pve ---
qm config 101 | grep -E '^(cores|cpu|memory|balloon):'
qm set 101 --cpu host && qm shutdown 101 && qm start 101

# --- inside ai: the VM ---
grep -o -w -E 'avx2|fma|f16c' /proc/cpuinfo | sort -u
free -m; cat /proc/pressure/memory
sudo nano /opt/automation/.env            # OLLAMA_MODEL, OLLAMA_CPUS, OLLAMA_MEM, OLLAMA_KEEP_ALIVE
cd /opt/automation && sudo bash deploy.sh # apply

# --- inside ai: models ---
sudo docker exec ollama ollama list               # downloaded
sudo docker exec ollama ollama ps                 # loaded now, size, CPU/GPU
sudo docker exec ollama ollama pull qwen3.5:4b
sudo docker exec ollama ollama stop qwen3.5:4b    # unload now
sudo docker exec ollama ollama rm mistral         # delete from disk
sudo docker exec -it ollama ollama run qwen3.5:4b --verbose --think=false
python3 ~/bench.py qwen3.5:4b granite4.2:3b

# --- inside ai: logs and limits ---
sudo docker logs ollama --since 10m
sudo docker stats --no-stream ollama
sudo docker inspect ollama --format '{{.State.OOMKilled}}'
```

## Glossary

- **Token**: the unit a model reads and writes, about ¾ of an English word.
- **Prefill / reading**: processing the prompt. Limited by compute.
- **Decode / writing**: producing the answer one token at a time. Limited by
  memory bandwidth.
- **Quantization**: storing weights in fewer bits (Q4_K_M ≈ 4.8 bits) to save
  RAM and speed up writing.
- **Context window**: how many tokens the model sees at once, prompt and answer
  together.
- **KV cache**: the memory that holds the context; grows with every token.
- **MoE (mixture of experts)**: a model that uses only part of its weights per
  token; needs the RAM of its total size, writes at the speed of its active
  size.
- **Thinking**: hidden reasoning a model writes before its answer; better
  answers, more waiting.
- **Embedding**: a list of numbers representing the meaning of a text, used for
  search and RAG.
- **AVX2 / FMA**: CPU instructions that do several multiplications at once; the
  fast path for CPU inference.
- **Keep-alive**: how long Ollama keeps a model in RAM after its last request.
