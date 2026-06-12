# Future Directions

Hardening and quality-of-life ideas for the worlock hermes-agent deployment. None are required for
day-to-day use; they reduce the chance of the original "wrong Python / missing deps" class of failure
recurring.

## 1. Pin the Python version explicitly

The root cause of the original breakage was ambient interpreter selection. Make it explicit:

- Add a `.python-version` file (`3.12`) so `pyenv`/tooling and humans agree on the interpreter.
- Optionally change the `./hermes` shebang on this fork to point at the repo venv directly
  (`#!/home/jeb/hermes-agent/.venv/bin/python`) so it no longer depends on `PATH`/activation.
  Trade-off: hard-codes a path, so it's a fork-local change, not upstreamable.

## 2. Add a bootstrap script / Makefile

A one-command setup removes the chance of skipping a step:

```make
# Makefile
.PHONY: setup run
setup:
	python3.12 -m venv .venv
	.venv/bin/python -m pip install --upgrade pip
	.venv/bin/python -m pip install -e .
run:
	.venv/bin/python ./hermes
```

Or a `scripts/bootstrap-worlock.sh` that also checks `python3.12` exists and errors clearly if not.

## 3. Decide which optional extras this host needs

worlock has dual GPUs and lots of RAM — it's a good candidate for the heavier extras:

- `[voice]` — local STT via faster-whisper can use the GPU.
- `[messaging]` / `[slack]` — if you want to reach the agent from Telegram/Discord/Slack.
- `[cron]` — for scheduled automations.

Install deliberately (see `howto.md` §4) rather than `.[all]`, to keep the surface small.

## 4. Keep the fork synced with upstream

```bash
git fetch origin
git merge origin/main        # or rebase your doc commits on top
git push private main
```

Consider a periodic `git fetch origin` (the Ollama delegation toolkit or a cron job could do this)
and a note when upstream bumps `requires-python` again — that would invalidate the venv.

## 5. Reproducibility

- Capture a known-good lockfile: `.venv/bin/python -m pip freeze > requirements.lock.txt` and commit
  it, so the exact dependency set can be reproduced even if upstream version ranges drift.
- Document GPU/driver assumptions in `bare_metal_system.md` (already done) so a rebuild on different
  hardware knows what changed.

## 6. CI / smoke test

A tiny GitHub Action (on the private repo) that runs `pip install -e .` + `python -c "import cli"` on
Python 3.11/3.12/3.13 would catch a broken dependency set before it reaches the host.

## 7. Observability

worlock already runs Netdata (port 19999) and the Ollama delegation toolkit. If hermes is run as a
long-lived gateway process, wire its logs (`./hermes logs`) into the existing monitoring rather than
leaving them in a terminal.
