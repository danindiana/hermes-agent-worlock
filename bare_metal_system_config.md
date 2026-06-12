# Bare-Metal System Config — `worlock`

Concrete configuration values relevant to running and rebuilding hermes-agent on this host.
Companion to [`bare_metal_system.md`](bare_metal_system.md) (the hardware/OS overview).

## Python interpreters available

worlock has several Pythons under `/usr/local/bin` plus the system one:

| Command | Version | Use |
|---------|---------|-----|
| `python3.11` | 3.11.0 | meets hermes `>=3.11` |
| `python3.12` | 3.12.9 | **chosen for hermes `.venv`** |
| `python3.13` | 3.13.7 | available |
| system `python3` | 3.10.x | too old for hermes |

## Virtual environments

| Path | Python | Purpose |
|------|--------|---------|
| `/home/jeb/hermes-agent/.venv` | 3.12 | **hermes-agent** (this repo) — created for the fix |
| `/home/jeb/programs/python_programs/venv` | 3.10 | shared fallback venv — **not** suitable for hermes |

`.venv/` is gitignored (`.gitignore` lines for `/venv/` and `.venv/`), so it is never committed.

## The `v` venv-activation helper

`v` is a shell alias: `v='source /home/jeb/bin/v.sh'`. `v.sh` cd's into an optional project arg, then
sources the first venv it finds from this ordered list:

```
./.venv/bin/activate          # repo-local (hermes wins here)
./venv/bin/activate
../.venv/bin/activate
../venv/bin/activate
/home/jeb/programs/python_programs/venv/bin/activate   # shared fallback (3.10)
```

Because `./.venv` is first, running `v` inside `/home/jeb/hermes-agent` activates the correct 3.12
environment automatically.

## Exact commands used to fix `./hermes`

```bash
cd /home/jeb/hermes-agent
python3.12 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -e .          # installs hermes-agent 0.10.0 + all base deps
# verify
.venv/bin/python -c "import cli; print('cli imports OK')"
.venv/bin/python ./hermes --help
```

## hermes-agent package facts (from `pyproject.toml`)

- Name/version: `hermes-agent` 0.10.0
- `requires-python = ">=3.11"`
- Base deps include: `openai`, `anthropic`, `python-dotenv`, `fire`, `httpx[socks]`, `rich`,
  `tenacity`, `pyyaml`, `requests`, `jinja2`, `pydantic`, `prompt_toolkit`, `exa-py`, `firecrawl-py`,
  `parallel-web`, `fal-client`, `edge-tts`, `PyJWT[crypto]`.
- Optional extras: `modal`, `daytona`, `dev`, `messaging`, `cron`, `slack`, `matrix`, `cli`,
  `tts-premium`, `voice`, `pty`, `honcho`.

## Host service config pointers (see `CLAUDE.md` for full detail)

| Area | Location / value |
|------|------------------|
| Ollama systemd override | `/etc/systemd/system/ollama.service.d/override.conf` (`CUDA_VISIBLE_DEVICES=0,1`, `OLLAMA_MAX_LOADED_MODELS=2`, `OLLAMA_NUM_PARALLEL=1`, `OLLAMA_KEEP_ALIVE=60m`) |
| SSH | port 22222, pubkey-only, UFW LAN (`enp9s0`) + VPN (`wgpia0`) only |
| UFW | 3000 / 19999 / 11435 LAN-only (192.168.1.0/24) + `wgpia0` deny; 443 allow-in |
| Docker compose | `open-webui` and `self-hosted-ai-starter-kit` under `~/programs/...` |
| Claude session logs | `~/.claude/projects/-home-jeb-Documents-claude-creations/` (JSONL) |
| GitHub | account `danindiana`, `gh` CLI authenticated, HTTPS + stored token |

## Git remotes for this checkout

```
origin    https://github.com/NousResearch/hermes-agent.git        # upstream
private   https://github.com/danindiana/hermes-agent-worlock.git   # private fork (push target)
```
