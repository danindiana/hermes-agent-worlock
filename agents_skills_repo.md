# `agents_skills_repo.md` — Cold-Start Onboarding for Agents & Frontier Models

> **You are reading the first file you should read.** If you are an AI agent, a frontier model, or a
> human dropped into this repository with zero prior context, this document gets you productive fast
> and keeps you from re-discovering the traps that have already been paid for. It is intentionally
> verbose. Skim the **Quick-Reference Card** at the very bottom if you need answers in ten seconds;
> read top-to-bottom if you intend to change anything.

---

## Table of contents

1. [What this repository actually is](#1-what-this-repository-actually-is)
2. [60-second start](#2-60-second-start)
3. [Ground rules for agents working here](#3-ground-rules-for-agents-working-here)
4. [The host: `worlock`](#4-the-host-worlock)
5. [Repository map](#5-repository-map)
6. [Build, install, run](#6-build-install-run)
7. [Configuration model](#7-configuration-model)
8. [Ollama integration — the heart of this fork](#8-ollama-integration--the-heart-of-this-fork)
9. [The skills system](#9-the-skills-system)
10. [How to verify changes (the two-layer rule)](#10-how-to-verify-changes-the-two-layer-rule)
11. [Gotchas already paid for](#11-gotchas-already-paid-for)
12. [Git & collaboration](#12-git--collaboration)
13. [Companion documentation index](#13-companion-documentation-index)
14. [Quick-Reference Card](#14-quick-reference-card)

---

## 1. What this repository actually is

This is **`danindiana/hermes-agent-worlock`** — a **private fork** of
[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent), deployed bare-metal on a
workstation named **`worlock`**.

- **Upstream Hermes Agent** is Nous Research's self-improving, tool-calling AI assistant (Python
  package `hermes-agent`, currently version **0.10.0**). It runs locally, in Docker, over SSH, or on
  serverless backends, talks to many model providers, has a TUI and messaging gateways, and curates
  its own memory and skills. The canonical upstream description lives in
  [`UPSTREAM_README.md`](UPSTREAM_README.md).
- **This fork** adds a **deployment + local-inference layer**: a working Python-3.12 bootstrap, the
  local **Ollama** integration (run everything on worlock's two GPUs, no cloud spend), and a reworked
  `/model` picker. None of the fork's value is in the agent's "intelligence" — it's in *making it run
  reliably on this specific machine against local models*.

**Entry point chain:** the `./hermes` launcher (a `#!/usr/bin/env python3` wrapper) →
`hermes_cli/main.py` (`main()`, argument parsing, all `hermes <subcommand>` commands) → for chat,
`cli.py` (the `HermesCLI`/chat orchestrator and `prompt_toolkit` TUI) → `run_agent.py`
(`AIAgent`, the conversation loop that actually calls the model).

**Two audiences, two docs:**
- This file = **orientation** (the fork, the host, how to run/verify, where everything is).
- [`AGENTS.md`](AGENTS.md) = **upstream code-internals dev guide** (617 lines: module responsibilities,
  toolsets, prompt building, session DB, etc.). When you need to modify agent *internals*, read that.
  This file points you at the right places; `AGENTS.md` goes deep.

---

## 2. 60-second start

```bash
cd /home/jeb/hermes-agent
v            # activate the project venv (Python 3.12) — see the golden rule below
./hermes     # launch the interactive agent
```

`v` is a shell alias (`source /home/jeb/bin/v.sh`) that activates the first virtualenv it finds,
preferring this repo's `./.venv`. After activation it prints the live interpreter, which should read
`…/hermes-agent/.venv/bin/python` and `Python 3.12.x`.

**The golden rule:** *Always activate `./.venv` before running anything Python in this repo.* The
`./hermes` shebang is `env python3`, so it runs under whatever Python is first on `PATH`. Without the
venv it falls back to the system Python 3.10 (which lacks the dependencies and is below the required
3.11) and dies with `ModuleNotFoundError: No module named 'prompt_toolkit'`. If you ever see that
error, you forgot `v`. Alternatively, call the interpreter explicitly: `.venv/bin/python ./hermes`.

---

## 3. Ground rules for agents working here

These are not style preferences — each one corresponds to a real failure mode (see
[Gotchas](#11-gotchas-already-paid-for)).

1. **Activate the venv first.** Every `python`, `pytest`, or `./hermes` invocation assumes `./.venv`
   is active. When in doubt: `which python3 && python3 --version`.
2. **This is a fork — push to `private`, not `origin`.** `origin` is upstream NousResearch (pull-only
   for you). Your changes go to `private` (`danindiana/hermes-agent-worlock`). See
   [Git & collaboration](#12-git--collaboration).
3. **Source edits do NOT hot-reload.** A running `./hermes` keeps the old code in memory. After editing
   any `.py`, **fully quit and relaunch**. Before concluding "my fix didn't work," compare the running
   process start time to your edit time: `ps -o pid,lstart,etime -p $(pgrep -f './hermes')` vs
   `stat -c %y <file>`. More than one debugging hour has been lost to this.
4. **The UI can lie — verify at the data layer.** A status bar or banner can show stale state while the
   underlying value is correct (or vice-versa). Confirm behavior at the API-kwargs and server layers
   (see [the two-layer rule](#10-how-to-verify-changes-the-two-layer-rule)).
5. **Never commit `.venv/`, secrets, or `~/.hermes/`.** `.venv/` is gitignored; the live config and
   credentials live outside the repo in `~/.hermes/`.
6. **Commit trailer convention.** End commit messages with the co-author trailer this project uses
   (see [Git](#12-git--collaboration)). Branch off `main` for anything non-trivial.
7. **`jeb` has passwordless sudo** (`NOPASSWD: ALL`). System changes (e.g. editing the Ollama systemd
   override) work without prompting — which means be deliberate; back up files before editing
   (`sudo cp file file.bak.$(date +%Y%m%d)`).

---

## 4. The host: `worlock`

| | |
|---|---|
| Hostname / user | `worlock` / `jeb` (passwordless sudo) |
| OS / kernel / shell | Ubuntu 22.04.5 LTS / Linux 6.8.12 / zsh |
| CPU / RAM | AMD Ryzen 9 5950X (16c/32t) / 125 GiB |
| GPUs (driver 580.159.03) | RTX 5080 (~16 GB) + RTX 3080 (~10 GB) ≈ 26.5 GB VRAM combined |
| LAN | enp9s0 — 192.168.1.85/24 (DHCP; interface name/IP drift, see CLAUDE.md) |
| Repo / venv | `/home/jeb/hermes-agent` / `./.venv` (Python 3.12) |
| Hermes config/state | `~/.hermes/` (config.yaml, sessions, caches, skills, memories) |
| Ollama | `127.0.0.1:11434`, localhost-only — `/v1` (OpenAI-compatible) + native root |
| Available Pythons | `/usr/local/bin/python3.{11,12,13}`; system `python3` is 3.10 (too old) |

Deeper host detail lives in [`bare_metal_system.md`](bare_metal_system.md) (hardware/OS/network),
[`bare_metal_system_config.md`](bare_metal_system_config.md) (paths, Python versions, `v.sh`,
services), and [`bare_metal_configs.md`](bare_metal_configs.md) (Ollama + Hermes config values). The
machine also runs other local-AI tooling (OpenWebUI on :3000, an Ollama delegation toolkit) that
shares the same Ollama server — relevant because `OLLAMA_MAX_LOADED_MODELS=2` lets Hermes and a
delegate model coexist.

---

## 5. Repository map

Top-level directories (annotated; counts are approximate and will drift):

```
hermes-agent/
├── hermes                     # launcher script (env python3) — `./hermes`
├── cli.py                     # HermesCLI — interactive chat orchestrator + prompt_toolkit TUI
├── run_agent.py               # AIAgent — the conversation loop; _build_api_kwargs(), switch_model()
├── model_tools.py             # tool orchestration, discover_builtin_tools(), handle_function_call()
├── hermes_constants.py        # global constants
├── pyproject.toml             # package metadata, deps, optional extras, requires-python >=3.11
│
├── agent/                     # agent internals (prompt build, compression, caching, model metadata)
│   ├── model_metadata.py      #   context-length resolution; MINIMUM_CONTEXT_LENGTH = 64_000
│   ├── context_compressor.py  #   auto context compression
│   ├── prompt_builder.py      #   system-prompt assembly
│   └── models_dev.py          #   models.dev registry integration
├── hermes_cli/                # all `hermes` subcommands + setup + the /model picker
│   ├── main.py                #   entry point — every `hermes <subcommand>`
│   ├── model_switch.py        #   /model picker engine: list_authenticated_providers(), switch_model()
│   ├── config.py              #   DEFAULT_CONFIG, load/save, custom_providers compatibility
│   ├── runtime_provider.py    #   resolve_runtime_provider() — per-turn credential/model resolution
│   ├── models.py              #   model catalog, fetch_api_models(), provider model lists
│   └── setup.py               #   interactive setup wizard
├── gateway/                   # messaging gateway (Telegram/Discord/Slack/WhatsApp/Signal)
├── tools/                     # built-in tool implementations
├── skills/                    # bundled skills (25 categories) — see §9
├── tests/                     # pytest suite (~54 entries)
├── diagrams/                  # architecture/network/howto diagrams (PNG + SVG)
├── acp_adapter/ acp_registry/ # ACP (agent-client protocol) integration
├── plugins/ environments/     # plugin + environment backends
└── docker/ nix/ packaging/    # deployment scaffolding
```

**Files you will most often edit in this fork** (the local-inference surface):
- `cli.py` — the chat TUI, the `/model` picker UI and selection handling, status bar.
- `hermes_cli/model_switch.py` — provider/model listing and switching logic.
- `hermes_cli/runtime_provider.py` — what gets resolved before every turn (a common source of
  "my switch reverted" bugs).
- `agent/model_metadata.py` — context-length detection and the 64k minimum gate.
- `hermes_cli/main.py` — subcommand behavior, custom-endpoint setup.

For anything touching prompt building, toolsets, the session DB (`hermes_state.py` / SQLite + FTS5),
prompt caching, or trajectory generation, **read [`AGENTS.md`](AGENTS.md) first** — it documents those
subsystems in detail and this file deliberately does not repeat them.

---

## 6. Build, install, run

The repo is installed **editable** into a dedicated Python 3.12 venv. Why 3.12: `pyproject.toml`
declares `requires-python = ">=3.11"`, and the only system-wide interpreter (3.10) cannot satisfy it —
so a project-local venv on a newer Python is mandatory, not optional.

**Recreate the environment from scratch:**

```bash
cd /home/jeb/hermes-agent
rm -rf .venv
python3.12 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -e .          # pulls all base deps from pyproject.toml
```

**Sanity check:**

```bash
.venv/bin/python -c "import cli; print('cli imports OK')"
.venv/bin/python ./hermes --help
```

(A couple of harmless `firecrawl` `UserWarning: Field name "json" … shadows an attribute` lines are
library noise, not errors.)

**Optional feature extras** (quote the bracket so zsh doesn't glob):

```bash
.venv/bin/python -m pip install -e '.[voice]'      # local STT/TTS
.venv/bin/python -m pip install -e '.[messaging]'  # Telegram/Discord/Slack/...
.venv/bin/python -m pip install -e '.[cron]'       # scheduler
.venv/bin/python -m pip install -e '.[dev]'        # pytest, debugpy, mcp
```

**Common subcommands:** `./hermes` (chat), `./hermes model`, `./hermes setup`, `./hermes status`,
`./hermes sessions`, `./hermes mcp`, `./hermes skills`, `./hermes tools`, `./hermes config`,
`./hermes doctor`. Add `--help` to any. The everyday operator runbook is [`howto.md`](howto.md).

---

## 7. Configuration model

Live config (NOT in the repo): **`~/.hermes/config.yaml`**. The relevant block for local Ollama:

```yaml
model:
  default: nemotron-3-nano-30b-small:64k     # current default model
  provider: custom                           # bare "custom" => endpoint defined by base_url
  base_url: http://localhost:11434/v1        # Ollama OpenAI-compatible endpoint (/v1 required)
  api_key: ollama                            # any non-empty string; Ollama ignores auth
custom_providers:
- name: Local Ollama
  base_url: http://localhost:11434/v1
  api_key: ollama
  model: nemotron-3-nano-30b-small:64k       # MUST match model.default (see invariant below)
```

**The critical invariant:** `model.default` and the matching `custom_providers[].model` must agree.
`resolve_runtime_provider()` reads the `custom_providers` entry's `model` on every turn and at launch;
if it disagrees with `model.default`, the stored value wins and silently overrides your selection. The
`/model` picker now keeps both in sync automatically — but if you hand-edit config, keep them aligned.

**Other state in `~/.hermes/`:** `context_length_cache.yaml` (per-model context lengths discovered by
probing — can hold a stale/too-small value that overrides live detection; delete an entry to force a
re-probe), `sessions/` + `state.db` (SQLite session store with FTS5 search), `skills/`, `memories/`,
`auth.json` (credentials — never commit).

Full config reference: [`bare_metal_configs.md`](bare_metal_configs.md).

---

## 8. Ollama integration — the heart of this fork

Hermes reaches Ollama through Ollama's **OpenAI-compatible** API. **Two URLs matter:**

| URL | API | Used for |
|-----|-----|----------|
| `http://localhost:11434/v1` | OpenAI-compatible | generation + `/v1/models` (picker model list) |
| `http://localhost:11434` (root) | Ollama native | `/api/tags`, `/api/ps`, `/api/generate` (unload) |

### Picking / switching models

```
/model            # in-chat: opens the picker (Provider → Model), live-lists all Ollama models
/model <name>             # switch by name, session only
/model <name> --global    # switch and persist as default
```

What the reworked picker does on selection (all fork-specific behavior):
1. Switches the **live agent** to that model (next prompt uses it).
2. **Persists** to `config.yaml` by default (both `model.default` and the `custom_providers` entry).
3. **Unloads** the previously-loaded model from VRAM (best-effort `keep_alive:0` to the native API).
4. Echoes `✓ Model switched` + `▶ Now answering with: <model>` and refreshes the status bar.

The picker lists **all** pulled models because `list_authenticated_providers(..., probe_live=True)`
merges `fetch_api_models()` results from `/v1/models` with the config models. Without `probe_live`
(tests, non-interactive callers) it stays config-only, so no surprise network calls.

### Multi-GPU model splitting

`OLLAMA_SCHED_SPREAD=1` (set in the systemd override) forces Ollama to spread every model across both
GPUs instead of fitting it on one. This unlocks the combined ~26.5 GB VRAM for larger models, at the
cost of per-token PCIe traffic (slower for models that would fit on one GPU). Details + the
trade-off table: [`bare_metal_configs.md`](bare_metal_configs.md).

### The `num_ctx` context shim (and the 64k floor)

Two **independent** context caps exist; a model must satisfy both:
- **Server-side (`num_ctx`):** what Ollama actually *serves*. Some models pin a small value in their
  Modelfile (e.g. `nemotron-3-nano-30b-small:latest` → `PARAMETER num_ctx 8192`). The `/v1` endpoint
  has no per-request `num_ctx`, and `OLLAMA_CONTEXT_LENGTH` does **not** override a Modelfile param.
- **Client-side (`MINIMUM_CONTEXT_LENGTH = 64_000`, `agent/model_metadata.py`):** Hermes refuses to
  initialize any model whose context is below 64,000 tokens.

**The fix is a derivative model with `num_ctx ≥ 64000`:**

```bash
printf 'FROM nemotron-3-nano-30b-small:latest\nPARAMETER num_ctx 65536\n' > /tmp/nemo.Modelfile
ollama create nemotron-3-nano-30b-small:64k -f /tmp/nemo.Modelfile
```

Hermes then detects 65536 and starts; the variant appears in the picker automatically. **Do not** fake
it with a `model.context_length` config override — that passes the client gate but causes silent
truncation, because Ollama still only serves the Modelfile's window. Mind VRAM: Ollama allocates the
KV cache for the full `num_ctx` at load time, so big windows on big models can OOM. Full rationale +
VRAM-fit table: [`howto_ollama_integration.md`](howto_ollama_integration.md).

---

## 9. The skills system

`skills/` holds Hermes's bundled **skills** — self-contained capability packages (instructions +
optional scripts/templates) the agent can invoke. There are ~25 categories, e.g. `software-development`,
`research`, `devops`, `mlops`, `data-science`, `diagramming`, `email`, `github`, `red-teaming`,
`autonomous-ai-agents`, `media`, `social-media`, `smart-home`, `note-taking`. Skills follow the
[agentskills.io](https://agentskills.io) open standard.

Key facts for an onboarding agent:
- **Enable/disable per platform:** `hermes skills` (CLI) or the `/skills` slash command (search,
  browse, install from the Skills Hub). Backed by `hermes_cli/skills_config.py` and
  `hermes_cli/skills_hub.py`.
- **Self-improving loop:** Hermes can author new skills after complex tasks and refine existing ones
  during use; the active skill prompt is snapshotted in `~/.hermes/.skills_prompt_snapshot.json`.
- **The skills prompt is large.** It (plus tool schemas + system prompt) is *why* a model needs a
  generous context window — and why the 8192-token cap on a raw model makes it unusable as an agent
  (see §8). When debugging context errors, remember the skills prompt is a major token consumer.

For authoring/curating skills internals, see upstream docs and [`AGENTS.md`](AGENTS.md).

---

## 10. How to verify changes (the two-layer rule)

A recurring trap this session: the TUI showed one thing while the agent did another. **Never trust the
UI alone.** Confirm at the layer that matters.

**Layer 1 — what Hermes will send (the model in the outgoing request):**

```bash
v
python -c "
from run_agent import AIAgent
a = AIAgent(model='qwen3.5:4b', api_key='ollama', base_url='http://localhost:11434/v1',
            provider='custom', api_mode='chat_completions', quiet_mode=True)
print(a._build_api_kwargs([{'role':'user','content':'hi'}])['model'])   # before
a.switch_model(new_model='qwen3:14b', new_provider='custom:local-ollama',
               api_key='ollama', base_url='http://localhost:11434/v1')
print(a._build_api_kwargs([{'role':'user','content':'hi'}])['model'])   # after -> qwen3:14b
"
```

**Layer 2 — what the server actually has loaded:**

```bash
curl -s http://localhost:11434/api/ps | python3 -c \
  "import sys,json;print([m['name'] for m in json.load(sys.stdin)['models']])"
```

**Context-gate check** (does a model clear the 64k floor?):

```bash
python -c "
from agent.model_metadata import get_model_context_length, MINIMUM_CONTEXT_LENGTH
c=get_model_context_length('nemotron-3-nano-30b-small:64k',
   base_url='http://localhost:11434/v1', api_key='ollama')
print(c,'passes:',c>=MINIMUM_CONTEXT_LENGTH)
"
```

**Tests** (pytest lives in the venv; the project sets xdist `-n` in `addopts` which can error if
`pytest-xdist` is absent — override it):

```bash
.venv/bin/python -m pip install -q pytest          # if needed
.venv/bin/python -m pytest tests/hermes_cli/test_user_providers_model_switch.py -q -o addopts=""
```

---

## 11. Gotchas already paid for

Each is documented in full in [`lessons_learned.md`](lessons_learned.md) or
[`lessons_learned_ollama.md`](lessons_learned_ollama.md). One line each here so you don't repeat them:

- **`env python3` shebang = ambient interpreter.** "Which Python" is set by `PATH`, not the repo. Use
  `v`.  → [`lessons_learned.md`](lessons_learned.md)
- **`requires-python >=3.11` is not pip-fixable.** A 3.10 venv can't be retrofitted; rebuild on 3.12.
- **Install the project, not its imports.** `pip install -e .` resolves all deps; chasing
  `ModuleNotFoundError`s one at a time is a treadmill.
- **Invalidate the view *after* mutating what it reads.** The status bar stayed stale because the
  repaint fired before the model updated.  → [`lessons_learned_ollama.md`](lessons_learned_ollama.md)
- **Per-turn re-resolution can clobber a mid-session change.** `_ensure_runtime_credentials()` re-reads
  the provider every turn; resolving `custom:<slug>` returned the stored model and reverted the pick.
  Fix: request bare `custom` after a custom-endpoint switch.
- **Two independent context caps.** Server-served `num_ctx` vs client `MINIMUM_CONTEXT_LENGTH = 64_000`;
  only the server side affects truncation. A config override that passes the gate but exceeds `num_ctx`
  silently truncates.
- **Source edits need a full restart.** No hot reload; check `ps -o pid,lstart` vs file mtime.

---

## 12. Git & collaboration

```
origin    https://github.com/NousResearch/hermes-agent.git        # UPSTREAM — pull updates only
private   https://github.com/danindiana/hermes-agent-worlock.git   # THIS FORK — push here
```

- **Pull upstream:** `git fetch origin && git merge origin/main` (or rebase your doc/code commits).
- **Push your work:** `git push private main`. Never `git push origin` — that targets upstream.
- **Commit message trailer** (this repo's convention):

  ```
  Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
  ```

- Branch off `main` for non-trivial work. The markdown doc set in the repo root is the **source of
  truth** for this deployment — when you change behavior, update the relevant doc in the same commit.
- GitHub account on this host is **`danindiana`** (`gh` CLI authenticated, HTTPS + stored token).

---

## 13. Companion documentation index

| Doc | Purpose |
|-----|---------|
| [`README.md`](README.md) | Fork overview, architecture/network diagrams, quick start |
| **`agents_skills_repo.md`** | **This file — cold-start onboarding for agents** |
| [`AGENTS.md`](AGENTS.md) | Upstream dev guide — deep code internals (toolsets, prompts, session DB) |
| [`howto.md`](howto.md) | General operator runbook (venv, subcommands, extras, updates) |
| [`howto_ollama_integration.md`](howto_ollama_integration.md) | Ollama wiring: pick/switch, verify, `num_ctx` shim, VRAM table |
| [`lessons_learned.md`](lessons_learned.md) | venv/bootstrap post-mortem |
| [`lessons_learned_ollama.md`](lessons_learned_ollama.md) | The four `/model` + context bugs and their fixes |
| [`bare_metal_system.md`](bare_metal_system.md) | Host hardware/OS/network overview |
| [`bare_metal_system_config.md`](bare_metal_system_config.md) | Paths, Python versions, `v.sh`, services |
| [`bare_metal_configs.md`](bare_metal_configs.md) | Ollama service + Hermes model config values |
| [`future_directions.md`](future_directions.md) | Hardening ideas (pin Python, bootstrap script, CI) |
| [`UPSTREAM_README.md`](UPSTREAM_README.md) | The original NousResearch README, unmodified |

---

## 14. Quick-Reference Card

```
START           cd /home/jeb/hermes-agent && v && ./hermes
VENV            ./.venv  (Python 3.12)   | activate: v   | rebuild: python3.12 -m venv .venv && pip install -e .
INTERPRETER?    which python3 && python3 --version   (want …/.venv/bin/python, 3.12.x)
CONFIG          ~/.hermes/config.yaml    | caches/sessions: ~/.hermes/
MODEL SWITCH    /model   (in chat)       | persists + unloads old + refreshes status bar
SERVED MODELS   curl -s localhost:11434/api/tags     | loaded: curl -s localhost:11434/api/ps
OLLAMA          127.0.0.1:11434  | /v1 = OpenAI-compat (gen + model list) | root = native (unload)
MULTI-GPU       OLLAMA_SCHED_SPREAD=1 (systemd override) splits a model across both GPUs
CONTEXT FLOOR   Hermes minimum = 64,000 tokens (agent/model_metadata.py)
NUM_CTX SHIM    ollama create <m>:64k -f Modelfile  (FROM <m> / PARAMETER num_ctx 65536)
VERIFY MODEL    AIAgent._build_api_kwargs([...])['model']     (don't trust the UI)
TESTS           .venv/bin/python -m pytest <path> -q -o addopts=""
EDIT≠RELOAD     fully restart ./hermes after any .py edit (check ps -o pid,lstart vs file mtime)
GIT             pull: origin (upstream)  | push: private (danindiana/hermes-agent-worlock)
SUDO            jeb has NOPASSWD: ALL    | back up before editing system files
DEEP INTERNALS  read AGENTS.md
```

> **If you change behavior, update the matching doc in the same commit.** The docs in this repo root
> are the operational source of truth for the worlock deployment; keeping them current is part of the
> definition of done.
