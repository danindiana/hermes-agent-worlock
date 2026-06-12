# Hermes Agent — worlock deployment ☤

> **This is a private fork** of [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent),
> deployed bare-metal on the host **`worlock`**. The original upstream README is preserved verbatim at
> [`UPSTREAM_README.md`](UPSTREAM_README.md). This README documents *this machine's* install, the venv
> fix that got `./hermes` running, and how to operate it here.

---

## What is Hermes Agent?

Hermes Agent is Nous Research's self-improving AI assistant with tool-calling capabilities. It
creates skills from experience, improves them during use, searches its own past conversations, and
builds a persistent model of the user across sessions. It runs locally, in Docker, over SSH, or on
serverless backends, and can be reached from a terminal TUI or messaging platforms (Telegram,
Discord, Slack, WhatsApp, Signal). See [`UPSTREAM_README.md`](UPSTREAM_README.md) for the full
feature tour and the canonical install path.

The Python package is `hermes-agent` (version 0.10.0), launched via the `./hermes` wrapper script.

---

## TL;DR — running it on worlock

```bash
cd /home/jeb/hermes-agent
v            # activates ./.venv (Python 3.12) via the `v` shell alias
./hermes     # launches the agent
```

If `./hermes` crashes with `ModuleNotFoundError: No module named 'prompt_toolkit'`, the local venv
is not active. See [Troubleshooting](#troubleshooting) below.

---

## The deployment story (why a local venv exists)

Out of the box, `./hermes` uses `#!/usr/bin/env python3`, so it runs under **whatever Python is first
on `PATH`**. On worlock that defaulted to the *shared* venv at
`/home/jeb/programs/python_programs/venv`, which:

1. Did **not** have hermes's dependencies installed (`prompt_toolkit`, `fire`, …), and
2. Is **Python 3.10**, while hermes requires **Python ≥ 3.11** — so it could not even
   `pip install -e .` the project.

The fix was a dedicated, repo-local virtual environment built on Python 3.12:

```bash
cd /home/jeb/hermes-agent
python3.12 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -e .       # pulls all deps from pyproject.toml
```

The `v` alias (`source /home/jeb/bin/v.sh`) searches `./.venv` **before** the shared venv, so once
this `.venv` exists, `v` activates the correct interpreter automatically.

`.venv/` is gitignored and never committed.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `ModuleNotFoundError: No module named 'prompt_toolkit'` (or `fire`, etc.) | `./hermes` ran under a venv without hermes deps | Run `v` from the repo dir first, or call `.venv/bin/python ./hermes` |
| `ERROR: Package 'hermes-agent' requires a different Python: 3.10.x not in '>=3.11'` | venv built on Python 3.10 | Rebuild the venv with `python3.12 -m venv .venv` |
| `UserWarning: Field name "json" ... shadows an attribute in parent "BaseModel"` (firecrawl) | Harmless upstream library warning | Ignore — not an error |

If unsure which interpreter is active: `which python3 && python3 --version`. Inside the repo's venv
it should report `…/hermes-agent/.venv/bin/python3` and `Python 3.12.x`.

---

## Companion documentation

| Doc | What's in it |
|-----|--------------|
| [`howto.md`](howto.md) | Operational runbook — activate venv, run subcommands, recreate the venv, update deps |
| [`lessons_learned.md`](lessons_learned.md) | The debugging narrative and the traps that bit us |
| [`future_directions.md`](future_directions.md) | Hardening ideas — pin Python, bootstrap script, extras, upstream sync |
| [`bare_metal_system.md`](bare_metal_system.md) | Host hardware/OS/network overview for worlock |
| [`bare_metal_system_config.md`](bare_metal_system_config.md) | Concrete config: paths, Python versions, `v.sh`, services |
| [`UPSTREAM_README.md`](UPSTREAM_README.md) | The original NousResearch README, unmodified |

---

## License

MIT, inherited from upstream. See [`LICENSE`](LICENSE).
