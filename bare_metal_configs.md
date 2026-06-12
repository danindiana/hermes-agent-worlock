# Bare-Metal Configs — Ollama + Hermes on `worlock`

Concrete configuration values for the local LLM stack that Hermes uses on worlock. Captured
2026-06-12. Companion to [`bare_metal_system.md`](bare_metal_system.md) (hardware/OS) and
[`bare_metal_system_config.md`](bare_metal_system_config.md) (Python/venv/services).

## Ollama service

- **Version:** ollama 0.22.1
- **Systemd override:** `/etc/systemd/system/ollama.service.d/override.conf`
- **Bind:** localhost-only (`OLLAMA_HOST=127.0.0.1:11434`)
- **Installed models:** 44 (see `ollama list`)

```ini
[Service]
Environment="OLLAMA_HOST=127.0.0.1:11434"     # localhost only — not exposed on LAN
Environment="CUDA_VISIBLE_DEVICES=0,1"        # both GPUs: RTX 5080 + RTX 3080 (~26.5 GB VRAM)
Environment="OLLAMA_MAIN_GPU=0"
Environment="OLLAMA_GPU_OVERHEAD=0"
Environment="OLLAMA_NUM_PARALLEL=1"           # requests queue; no concurrent generations
Environment="OLLAMA_MAX_LOADED_MODELS=2"      # up to two models resident at once
Environment="OLLAMA_KEEP_ALIVE=60m"           # warm window after last use
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"       # quantized KV cache to save VRAM
```

After editing: `sudo systemctl daemon-reload && sudo systemctl restart ollama`
(jeb has `NOPASSWD: ALL`, so no password prompt).

### Endpoints

| URL | API | Hermes uses it for |
|-----|-----|--------------------|
| `http://localhost:11434/v1` | OpenAI-compatible | generation + `/v1/models` (picker list) |
| `http://localhost:11434` | Ollama native | `/api/ps`, `/api/tags`, `/api/generate` (unload) |

## Hermes model config

File: `~/.hermes/config.yaml`

```yaml
model:
  default: <current-model>                    # synced to the picker selection
  provider: custom                            # bare "custom" => endpoint = base_url
  base_url: http://localhost:11434/v1
  api_key: ollama                             # any non-empty string; Ollama ignores it
custom_providers:
- name: Local Ollama
  base_url: http://localhost:11434/v1
  api_key: ollama
  model: <current-model>                      # picker keeps this == model.default
```

Key invariant (learned the hard way — see [[lessons_learned_ollama]]): `model.default` and the
`custom_providers` entry's `model` must agree, because `resolve_runtime_provider()` reads the latter
on every turn / launch. The `/model` picker now keeps them in sync automatically.

## VRAM behaviour on switch

- `OLLAMA_MAX_LOADED_MODELS=2` lets Ollama hold two models; `OLLAMA_KEEP_ALIVE=60m` keeps them warm.
- On a model switch the Hermes picker fires a best-effort unload of the *previous* model
  (`POST /api/generate {"model": <old>, "keep_alive": 0}`) so VRAM frees immediately rather than
  waiting out the keep-alive.

## Quick commands

```bash
ollama list                                   # 44 installed models
curl -s localhost:11434/api/ps   | python3 -c "import sys,json;print([m['name'] for m in json.load(sys.stdin)['models']])"   # loaded
curl -s localhost:11434/api/tags | python3 -c "import sys,json;print(len(json.load(sys.stdin)['models']))"                   # count
systemctl status ollama --no-pager | head     # service health
nvtop                                          # live GPU/VRAM
```

## Related local AI tooling (not Hermes, same host)

- **Ollama Delegation Toolkit** — `~/Documents/claude_creations/.../ollama-delegate`
  (repo: `github.com/danindiana/ollama-delegate`). Uses the same Ollama server as a fast local
  reasoning/planning layer. `OLLAMA_MAX_LOADED_MODELS=2` lets Hermes and a delegate model coexist.
- **OpenWebUI** (Docker, port 3000) and **self-hosted-ai-starter-kit** (GPU profile) — see
  `bare_metal_system.md`.

Related: [[bare_metal_system_config]], [[howto_ollama_integration]], [[lessons_learned_ollama]].
