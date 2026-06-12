# How-To — Ollama integration with Hermes on worlock

How the local Ollama server is wired into Hermes, how to pick/switch models, and how to verify it.

## The setup

Hermes talks to Ollama through Ollama's **OpenAI-compatible** API. The relevant `config.yaml`
(`~/.hermes/config.yaml`) block:

```yaml
model:
  default: qwen3.5:4b                       # whichever model is the current default
  provider: custom                          # bare "custom" = endpoint defined by base_url
  base_url: http://localhost:11434/v1       # Ollama's OpenAI-compatible endpoint (/v1 required)
  api_key: ollama                           # Ollama ignores auth; any non-empty string works
custom_providers:
- name: Local Ollama
  base_url: http://localhost:11434/v1
  api_key: ollama
  model: qwen3.5:4b                          # kept in sync with model.default by the picker
```

Two URLs matter:
- **`/v1`** (`http://localhost:11434/v1`) — OpenAI-compatible chat/completions + `/v1/models`. This
  is what Hermes uses for generation and for listing models in the picker.
- **native root** (`http://localhost:11434`) — Ollama's own API (`/api/tags`, `/api/ps`,
  `/api/generate`). Hermes uses this only to unload a model (free VRAM) on switch.

## Picking / switching models in chat

```
/model            # opens the interactive picker (Provider → Model)
```

Select **Local Ollama**, then a model. On selection Hermes now:

1. Switches the live agent to that model (the next prompt uses it).
2. **Persists** it to `config.yaml` by default (survives restart) — both `model.default` and the
   `custom_providers` entry's `model`.
3. **Unloads** the previously-loaded model from Ollama VRAM (best-effort).
4. Prints `✓ Model switched` + `▶ Now answering with: <model>` and refreshes the status bar.

Switch by name without the menu:

```
/model qwen3:14b               # session only
/model qwen3:14b --global      # also persist as default
```

## How the picker lists every Ollama model

`list_authenticated_providers(..., probe_live=True)` calls `fetch_api_models(api_key, base_url)`
against `http://localhost:11434/v1/models` and merges the live models with the config ones (config
default stays first). Without `probe_live` (tests, non-interactive callers) it returns config-only,
so no network calls happen unexpectedly.

> The picker caps at `max_models=50`. With more than ~50 pulled models you'd see a subset; bump the
> `max_models=50` arg in `cli.py`'s `_handle_model_switch` if needed.

## Verify it actually switched

UI can lie; confirm at two layers.

**What Hermes will send** (model that goes into the API request):

```bash
v   # activate the repo venv
python -c "
from run_agent import AIAgent
a = AIAgent(model='qwen3.5:4b', api_key='ollama', base_url='http://localhost:11434/v1',
            provider='custom', api_mode='chat_completions', quiet_mode=True)
a.switch_model(new_model='qwen3:14b', new_provider='custom:local-ollama',
               api_key='ollama', base_url='http://localhost:11434/v1')
print(a._build_api_kwargs([{'role':'user','content':'hi'}])['model'])   # -> qwen3:14b
"
```

**What the server has loaded:**

```bash
curl -s http://localhost:11434/api/ps | python3 -c \
  "import sys,json;print([m['name'] for m in json.load(sys.stdin)['models']])"
```

After sending a prompt with the new model selected, the new model should appear here and the old one
should drop (Hermes unloads it; otherwise it ages out via keep-alive).

## Uncapping a model's context window (`num_ctx` shim)

Some Ollama models pin a small context window in their **Modelfile**, e.g.
`nemotron-3-nano-30b-small:latest` ships with:

```
PARAMETER num_ctx 8192
```

8192 tokens fails outright. Hermes enforces a **hard minimum of 64,000 tokens**
(`MINIMUM_CONTEXT_LENGTH = 64_000` in `agent/model_metadata.py`); below it, agent init aborts with:

```
Failed to initialize agent: Model <name> has a context window of 8,192 tokens,
which is below the minimum 64,000 required by Hermes Agent.
```

The model's *architecture* supports far more (this one reports 1,048,576), but Ollama only **serves**
what `num_ctx` allows, and:

- The OpenAI-compatible `/v1` endpoint has **no per-request `num_ctx`**, so Hermes can't raise it.
- `OLLAMA_CONTEXT_LENGTH` (env) does **not** override a Modelfile `PARAMETER num_ctx`.

**Shim = a derivative model with `num_ctx` ≥ 64,000.** It must clear Hermes's 64k floor, so 32k is
*not* enough — go straight to 64k:

```bash
printf 'FROM nemotron-3-nano-30b-small:latest\nPARAMETER num_ctx 65536\n' > /tmp/nemo.Modelfile
ollama create nemotron-3-nano-30b-small:64k -f /tmp/nemo.Modelfile
```

Then point Hermes at the variant (and the custom_providers entry, so it sticks) — or just select it in
the `/model` picker, where it appears automatically (live-listed):

```bash
v && python -c "
from agent.model_metadata import get_model_context_length, MINIMUM_CONTEXT_LENGTH
ctx = get_model_context_length('nemotron-3-nano-30b-small:64k',
      base_url='http://localhost:11434/v1', api_key='ollama')
print(ctx, 'passes:', ctx >= MINIMUM_CONTEXT_LENGTH)   # -> 65536 passes: True
"
```

No further Hermes config is needed — it reads `num_ctx` from the new model and detects 65536.

**Mind the VRAM.** Ollama allocates the KV cache for the *full* `num_ctx` at load time, so bigger
windows cost VRAM up front. On worlock (≈26.5 GB across both GPUs, `SCHED_SPREAD=1`) for this ~24 GB
model:

| Variant | Hermes gate (≥64k) | Total VRAM at load | GPU 0 (5080) headroom | Verdict |
|---------|--------------------|--------------------|-----------------------|---------|
| `:latest` (8k) | ✗ rejected | ~14.5 GB | lots | unusable — below the 64k floor |
| `:32k` | ✗ rejected | ~24.8 GB | ~0.8 GB | still below the 64k floor — don't bother |
| `:64k` | ✓ passes | ~25.0 GB | ~0.7 GB | **the working option — GPU 0 is tight but loads** |

`:64k` is the smallest window that satisfies Hermes here, which is fortunate because this 24 GB model
leaves little VRAM room — going higher than 64k risks OOM during generation. `OLLAMA_KV_CACHE_TYPE=q8_0`
(already set, see `bare_metal_configs.md`) keeps the KV cache compact. If `:64k` OOMs under real load,
the model is simply too big for this GPU pair at agent context — use a smaller/lower-quant model.

To remove a variant: `ollama rm nemotron-3-nano-30b-small:64k`.

## Managing Ollama directly

```bash
ollama list                                         # installed models
curl -s http://localhost:11434/api/tags             # same, via API
curl -s http://localhost:11434/api/ps               # currently loaded (in VRAM)
# unload a model now (what Hermes does on switch):
curl -s http://localhost:11434/api/generate -d '{"model":"qwen3.5:4b","keep_alive":0}'
```

Ollama on worlock (see `bare_metal_configs.md`): both GPUs, `OLLAMA_MAX_LOADED_MODELS=2`,
`OLLAMA_NUM_PARALLEL=1`, `OLLAMA_KEEP_ALIVE=60m`.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Picker lists only 1 model | running old build, or `probe_live` not set | restart hermes; ensure Ollama is up at `:11434` |
| Selected model "doesn't take" | running session predates the fix | **fully quit and relaunch** — source edits don't hot-reload |
| Still defaults to old model after restart | `custom_providers` entry `model` out of sync | pick again (now syncs both), or edit `config.yaml` |
| `ollama ps` still shows old model | keep-alive (60m) | normal; Hermes also unloads on switch |
| 400 "model must be non-empty" | base_url missing `/v1` | use `http://localhost:11434/v1` |

## Restart reminder

A running `./hermes` will not pick up edited source or some config changes. After changing code or
deep config, **quit completely and relaunch** (`v && ./hermes`). Confirm with
`ps -o pid,lstart -p <pid>` that the process started after your change.

Related: [[howto]] (general operations), [[lessons_learned_ollama]] (why these bugs existed).
