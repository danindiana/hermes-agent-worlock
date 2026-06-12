# How-To — Download & configure Carnice-9b for Ollama + Hermes

A worked record of pulling a raw GGUF off HuggingFace and wiring it into Ollama so Hermes can run it.
The same recipe applies to any single-file GGUF; Carnice-9b is the concrete example.

- **Source:** <https://huggingface.co/kai-os/Carnice-9b-GGUF> → `Carnice-9b-Q8_0.gguf`
- **What it is (from GGUF metadata):** Qwen3.5-architecture **9B**, quantization **Q8_0**, native
  context **262144 (256k)**. It is a **reasoning model** (Qwen3.5 thinking mode).
- **Ollama model created:** `carnice-9b:latest`

---

## 1. Download the GGUF

HuggingFace "blob" URLs are HTML pages; the actual file is the **`resolve`** URL. Large HF files now
go through the Xet CAS bridge via a signed, expiring redirect, and the transfer can drop mid-stream —
so always download with resume + retries:

```bash
mkdir -p ~/models && cd ~/models
curl -L -C - --retry 8 --retry-delay 3 --retry-all-errors \
  -o Carnice-9b-Q8_0.gguf \
  "https://huggingface.co/kai-os/Carnice-9b-GGUF/resolve/main/Carnice-9b-Q8_0.gguf"
```

- `-L` follow redirects · `-C -` resume a partial file · `--retry*` survive Xet drops.
- The first attempt here died at 34 MB (curl exit 18, partial file); the resumed pull completed.

**Verify the download** (size must match the server's `Content-Length`; magic must be `GGUF`):

```bash
curl -sIL "<resolve-url>" | grep -i content-length   # expected: 9527501600
stat -c %s Carnice-9b-Q8_0.gguf                        # must equal the above
head -c 4 Carnice-9b-Q8_0.gguf                         # must print: GGUF
```

## 2. Read the model's metadata first

Import once with a bare Modelfile to learn the architecture and native context (this dictates the
template and a safe `num_ctx`):

```bash
printf 'FROM ~/models/Carnice-9b-Q8_0.gguf\n' > /tmp/probe.Modelfile
ollama create carnice-9b:probe -f /tmp/probe.Modelfile
ollama show carnice-9b:probe | grep -iE "architecture|parameters|context length|quantization"
#   architecture  qwen35      parameters 9.0B
#   context length 262144     quantization Q8_0
ollama rm carnice-9b:probe
```

## 3. Create the real model (the two things that matter)

A raw GGUF often ships **no chat template**, so Ollama falls back to `TEMPLATE {{ .Prompt }}`
(passthrough). That makes a ChatML model leak `<|im_start|>` / `<|endoftext|>` tokens and never stop.

For **Qwen3.5** models, Ollama 0.22+ has a built-in renderer/parser — the same one the stock
`qwen3.5:4b` uses (check with `ollama show --modelfile qwen3.5:4b`). Use it instead of hand-writing a
Go template:

```Dockerfile
# ~/models/Carnice-9b.Modelfile
FROM /home/jeb/models/Carnice-9b-Q8_0.gguf
RENDERER qwen3.5          # native ChatML formatting
PARSER qwen3.5            # separates reasoning vs content, applies stop tokens
PARAMETER num_ctx 131072  # >= Hermes's 64,000 floor (see howto_ollama_integration.md)
PARAMETER temperature 1
PARAMETER top_k 20
PARAMETER top_p 0.95
```

```bash
ollama create carnice-9b:latest -f ~/models/Carnice-9b.Modelfile
```

**Why `num_ctx 131072`:** Hermes refuses any model below `MINIMUM_CONTEXT_LENGTH = 64_000`. The model
supports 256k natively; 128k clears the floor with margin and fits VRAM easily for a 9B. (Background:
[`howto_ollama_integration.md`](howto_ollama_integration.md) — "Uncapping a model's context window".)

## 4. Verify the way Hermes will actually call it

Test the **OpenAI-compatible `/v1`** endpoint, not just `/api/generate`:

```bash
curl -s http://localhost:11434/v1/chat/completions -H 'Content-Type: application/json' -d '{
  "model":"carnice-9b:latest",
  "messages":[{"role":"user","content":"Say hello in one short sentence."}],
  "max_tokens":600}' | python3 -c "import sys,json;d=json.load(sys.stdin);m=d['choices'][0]['message'];print('finish:',d['choices'][0]['finish_reason']);print('content:',repr(m.get('content','')));print('reasoning:',repr((m.get('reasoning_content') or m.get('reasoning') or '')[:120]))"
# finish: stop   content: 'Hello! How can I help you today?'   reasoning: 'Thinking Process: ...'
```

> **It's a reasoning model.** Chain-of-thought goes into `reasoning`; the final answer is in `content`.
> Give it a generous `max_tokens` — a small budget can be consumed entirely by thinking and return an
> empty `content` with `finish_reason: length`. Hermes handles reasoning content natively
> (stored in `assistant_msg["reasoning"]`).

Confirm Hermes accepts the context window:

```bash
v && python -c "
from agent.model_metadata import get_model_context_length, MINIMUM_CONTEXT_LENGTH
c=get_model_context_length('carnice-9b:latest', base_url='http://localhost:11434/v1', api_key='ollama')
print(c, 'passes:', c >= MINIMUM_CONTEXT_LENGTH)   # 131072 passes: True
"
```

## 5. Use it / clean up

- In chat: `/model` → select **carnice-9b:latest** (the picker live-lists Ollama models).
- **`ollama create` copies the GGUF into Ollama's own blob store**
  (`/usr/share/ollama/.ollama/models`), so the downloaded source file is now a duplicate. Once the
  model is verified, the source can be deleted to reclaim space:

  ```bash
  rm ~/models/Carnice-9b-Q8_0.gguf   # ~8.9 GB; the ollama model keeps working
  ```

## VRAM note (worlock-specific)

Carnice (~9.5 GB) coexists fine with smaller models, but **cannot** be co-resident with
`nemotron-3-nano-30b-small:64k` (~25 GB, which nearly fills the 26.5 GB combined VRAM). Loading
Carnice while nemotron is resident fails with `model failed to load … resource limitations`
(`exit status 2` in the Ollama log). The fix is just to unload the big model first — and the Hermes
`/model` picker already unloads the previous model on switch. See
[`bare_metal_configs.md`](bare_metal_configs.md) for the GPU/VRAM layout and `OLLAMA_SCHED_SPREAD`.

Related: [[howto_ollama_integration]] · [[bare_metal_configs]] · [[lessons_learned_ollama]]
