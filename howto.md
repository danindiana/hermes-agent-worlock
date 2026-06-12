# How-To — operating Hermes Agent on worlock

A practical runbook for the bare-metal deployment at `/home/jeb/hermes-agent`.

## 1. Activate the environment

The repo uses a dedicated Python 3.12 venv at `./.venv`. Activate it with the `v` alias:

```bash
cd /home/jeb/hermes-agent
v
```

`v` is `source /home/jeb/bin/v.sh`. `v.sh` searches these locations **in order** and sources the
first it finds:

1. `./.venv/bin/activate`   ← this repo's venv (winner)
2. `./venv/bin/activate`
3. `../.venv/bin/activate`
4. `../venv/bin/activate`
5. `/home/jeb/programs/python_programs/venv/bin/activate`  ← shared fallback (Python 3.10, wrong for hermes)

On success it prints the active interpreter, e.g.
`(Active) Python 3.12.x @ ./.venv/bin/activate`.

> If you run `./hermes` **without** activating first, the shebang falls back to the system `python3`
> and you'll get `ModuleNotFoundError`. Either run `v` first, or call `.venv/bin/python ./hermes`.

## 2. Run the agent

```bash
./hermes               # default interactive chat
./hermes --help        # top-level usage and subcommands
./hermes chat          # interactive chat with the agent
./hermes model         # select default model and provider
./hermes setup         # interactive setup wizard
./hermes status        # status
./hermes sessions      # list/inspect past sessions
./hermes mcp           # MCP server management
```

Other subcommands include `gateway`, `cron`, `webhook`, `hooks`, `doctor`, `backup`, `import`,
`config`, `skills`, `plugins`, `memory`, `tools`, `dashboard`, `logs`. Run `./hermes <cmd> --help`
for any of them.

Global flags worth knowing: `--resume SESSION`, `--continue [NAME]`, `--worktree`, `--skills`,
`--yolo`, `--tui`, `--dev`.

## 3. Recreate the venv from scratch

If the venv is corrupted or deleted:

```bash
cd /home/jeb/hermes-agent
rm -rf .venv
python3.12 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -e .
```

Verify:

```bash
.venv/bin/python -c "import cli; print('cli imports OK')"
.venv/bin/python ./hermes --help
```

## 4. Install optional feature extras

`pyproject.toml` defines optional dependency groups. Install only what you need:

```bash
.venv/bin/python -m pip install -e '.[voice]'       # local STT/TTS (faster-whisper, sounddevice)
.venv/bin/python -m pip install -e '.[messaging]'   # Telegram, Discord, Slack, aiohttp, qrcode
.venv/bin/python -m pip install -e '.[cron]'        # croniter scheduler
.venv/bin/python -m pip install -e '.[slack]'       # slack-bolt + slack-sdk
.venv/bin/python -m pip install -e '.[modal]'       # Modal serverless backend
.venv/bin/python -m pip install -e '.[dev]'         # pytest, debugpy, mcp
```

(Quote the extra so zsh doesn't glob the brackets.)

## 5. Update dependencies

```bash
cd /home/jeb/hermes-agent
git pull origin main           # pull upstream changes (origin = NousResearch)
.venv/bin/python -m pip install -e . --upgrade
```

## 6. Common errors

| Error | Fix |
|-------|-----|
| `ModuleNotFoundError: No module named 'prompt_toolkit'` / `'fire'` | Activate venv (`v`) or use `.venv/bin/python ./hermes` |
| `requires a different Python: 3.10.x not in '>=3.11'` | Rebuild venv with `python3.12 -m venv .venv` |
| zsh: `no matches found: .[voice]` | Quote the extra: `'.[voice]'` |
| firecrawl `json` field `UserWarning` | Ignore — harmless |

## 7. Git remotes on this checkout

```
origin    https://github.com/NousResearch/hermes-agent.git      (upstream — pull updates)
private   https://github.com/danindiana/hermes-agent-worlock.git (your private fork — push here)
```

Push local/doc changes to `private`. Pull upstream code from `origin`.
