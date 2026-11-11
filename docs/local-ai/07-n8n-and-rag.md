# n8n and RAG with local models

n8n is the main user of the models on `ai`. This page covers the two ways to
call Ollama from a workflow, how to keep CPU-speed AI out of the way of things
people wait for, and how to build RAG with Qdrant once the chatbot moves here.

## The credential is already there

`n8n-credentials.sh` created a shared credential called **`Ollama (local)`**
with Base URL `http://ollama:11434`. Every Ollama node uses it. Nothing about
it changes when you switch models: the model is chosen per node.

## Two ways to call a model

### 1. The AI nodes: Ollama Chat Model, Embeddings Ollama

n8n's AI nodes (Basic LLM Chain, AI Agent, Information Extractor, Text
Classifier, vector stores) take a **model sub-node**. For local models that's
**Ollama Chat Model** (chat) or **Embeddings Ollama** (vectors).

- Good for: agents, chains, anything that uses n8n's built-in AI features.
- In the sub-node, set **Model** to the tag you pulled (`qwen3.5:4b`,
  `granite4.2:3b`) and **Sampling Temperature** to 0 for repeatable
  extraction and classification.

### 2. An HTTP Request node straight to the API

For one-shot jobs (classify this, extract that), calling `/api/chat` directly
gives full control: thinking, JSON schema, context size and keep-alive.

| Field | Value |
|---|---|
| Method | `POST` |
| URL | `http://ollama:11434/api/chat` |
| Body Content Type | JSON |
| Options → Timeout | `300000` (5 minutes; CPU answers are slow) |

Body (Using JSON):

```json
{
  "model": "granite4.2:3b",
  "stream": false,
  "think": false,
  "messages": [
    {"role": "system", "content": "Classify the customer's message. Answer only with the JSON."},
    {"role": "user", "content": "{{ $json.text }}"}
  ],
  "format": {
    "type": "object",
    "properties": {
      "category": {"type": "string", "enum": ["payment", "question", "other"]}
    },
    "required": ["category"]
  },
  "options": {"temperature": 0}
}
```

The answer is in `message.content` as a JSON string; parse it with an *Edit
Fields* node (`{{ JSON.parse($json.message.content).category }}`). The `enum`
means the model *can't* answer anything but those three words.

If the message text can contain quotes or new lines, build the body in a *Code*
node or with `{{ JSON.stringify($json.text) }}` (without the surrounding
quotes) so the JSON stays valid.

## Thinking in n8n

n8n's documentation for the Ollama Chat Model node lists no thinking switch.
Three ways to keep thinking from costing minutes:

1. **Use the HTTP Request node** with `"think": false` (above). Fully
   controlled.
2. **Choose a model that doesn't think by default.** Check with the `/api/show`
   command in [chapter 5](05-ollama-on-the-stack.md#thinking-on-or-off).
3. **If you see a long reasoning block in the output**, the model is thinking:
   switch to option 1 or 2 for that step.

## Rules for workflows that use a local model

These follow from CPU speed ([chapter 1](01-can-a-cpu-run-ai.md)) and one shared
Ollama:

1. **One small job per call.** "Classify", "extract these four fields",
   "summarise in two sentences". Not "read this thread and handle it".
2. **Fixed output.** A JSON schema in `format`, temperature 0, and the parsing
   in n8n, not in your head.
3. **Short prompts.** Strip HTML, quoted replies and signatures before the text
   reaches the model. Every 1,000 prompt tokens is tens of seconds of reading.
4. **Never on the path a human waits on.** The design notes already apply this
   to the payment bot (rules, not AI). Customers get an instant rule-based
   reply; AI can enrich it later.
5. **Expect a queue.** Ollama answers one request at a time. Two AI workflows
   that fire together run one after the other. Schedule heavy batches for the
   night.
6. **Raise timeouts.** Anything calling Ollama needs minutes, not n8n's
   default seconds.

## RAG: answering from your own documents

RAG means: turn documents into vectors once, store them in Qdrant, and for each
question fetch the few most relevant pieces and hand only those to the model.

```
Documents ──► split into chunks ──► Embeddings Ollama ──► Qdrant
                                    (qwen3-embedding:0.6b)

Question ──► Embeddings Ollama ──► Qdrant: top 3 chunks
                                        │
                                        ▼
                         Ollama Chat Model (qwen3.5:4b) ──► answer
```

### Settings that make it work on this box

| Setting | Value | Why |
|---|---|---|
| Embedding model | `qwen3-embedding:0.6b` | Small and good ([chapter 4](04-the-best-models.md#qwen3-embedding-06b-for-search-and-rag)) |
| Qdrant collection vector size | **1024**, Cosine | The size Qwen3 Embedding 0.6B outputs. Must match, or inserts fail |
| `OLLAMA_MAX_LOADED_MODELS` | **2** | Chat and embedding models both stay loaded. With 1, every question unloads the chat model to embed, then reloads it (Mistral's reload took 48s) |
| `OLLAMA_MEM` | 6g is enough | ~4GB chat + ~1GB embedding + caches |
| Chunk size | ~500 tokens, small overlap | Three chunks plus the question stay around 1,500 tokens |
| Top K | 3 | Each extra chunk is more reading time |
| `OLLAMA_CONTEXT_LENGTH` | 8192 if prompts exceed 4K | Otherwise the prompt is silently cut |

In n8n: a **Qdrant Vector Store** node (credential `Qdrant (local)`,
`http://qdrant:6333`) with **Embeddings Ollama** attached, once in an
"insert documents" workflow and once in the "answer questions" workflow. Use
the same embedding model in both.

### What to expect

With Mistral, the design notes estimated **1.5 to 3 minutes** per RAG answer.
With the chapter 4 models and the scaling from
[chapter 2](02-sizing-a-model.md#estimating-speed-before-you-download), about
1,500 prompt tokens and a 150-token answer:

| Chat model | Reading | Writing | Total (est., model loaded) |
|---|---|---|---|
| Mistral 7B (old) | ~94s | ~50s | ~2.4 min |
| Qwen3.5 4B, thinking off | ~52s | ~38s | ~1.5 min |
| Granite 4.2 3B, thinking off | ~39s | ~25s | ~1.1 min |

Estimates, not measurements. Still too slow for a live chat, fine for an
answer that arrives as a message a minute later.

## When to call a hosted model instead

The original plan, n8n orchestrating and a hosted API (such as OpenRouter)
thinking, still has a place. Use it when:

- a person is waiting and an answer must come in seconds;
- the input is long (whole documents, long threads);
- the job needs many model calls (multi-step agents) or code;
- quality matters more than keeping the text in the house.

Keep the local model when the text is private, the job is small and
repeatable, or the workflow runs unattended and can wait.

A useful hybrid: the local model does the private first pass (classify, pull
out the needed fields, drop personal details), and only that reduced text goes
to a hosted model.
