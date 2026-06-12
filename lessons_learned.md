# Lessons Learned — getting `./hermes` running on worlock

A short post-mortem of the `ModuleNotFoundError: No module named 'prompt_toolkit'` debugging session,
written so the next person (or the next machine) doesn't relive it.

## The original failure

```
$ ./hermes
Traceback (most recent call last):
  File "/home/jeb/hermes-agent/./hermes", line 11, in <module>
    main()
  ...
  File "/home/jeb/hermes-agent/cli.py", line 43, in <module>
    from prompt_toolkit.history import FileHistory
ModuleNotFoundError: No module named 'prompt_toolkit'
```

It looks like "just pip install a module." It wasn't — there were three layered traps.

## Trap 1 — `#!/usr/bin/env python3` runs under *whatever* venv is active

`./hermes` starts with `#!/usr/bin/env python3`. That means the interpreter is chosen by `PATH`, not
by the repo. On worlock the active interpreter resolved to the **shared** venv at
`/home/jeb/programs/python_programs/venv`, which had none of hermes's dependencies.

**Lesson:** a shebang of `env python3` makes the *activation order* the real configuration. The repo
itself doesn't pin its interpreter, so "which Python" is ambient state.

## Trap 2 — the shared venv was Python 3.10; hermes needs ≥ 3.11

Installing the project into the shared venv to fix the missing deps failed outright:

```
ERROR: Package 'hermes-agent' requires a different Python: 3.10.12 not in '>=3.11'
```

`pyproject.toml` declares `requires-python = ">=3.11"`. So the shared venv was a dead end — you
cannot satisfy the requirement by adding packages to it.

**Lesson:** check `requires-python` before trying to retrofit an existing environment. A version
mismatch is not fixable with `pip install`.

## Trap 3 — installing one missing module just reveals the next

Manually `pip install prompt_toolkit` got past line 43, only to hit `import fire` a few lines later,
then more. Chasing imports one at a time is a treadmill.

**Lesson:** when a project ships a `pyproject.toml`/`requirements.txt`, install *the project*
(`pip install -e .`) and let the dependency resolver pull everything declared. Don't hand-install
transitive deps.

## The fix that actually worked

A dedicated, repo-local venv on a supported Python:

```bash
cd /home/jeb/hermes-agent
python3.12 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -e .
```

This works because:
- worlock has Python 3.11, 3.12, and 3.13 in `/usr/local/bin` (3.12 was chosen as a stable middle).
- `pip install -e .` resolved all of `pyproject.toml`'s base dependencies in one shot.
- The `v` alias prefers `./.venv` over the shared venv (see `v.sh` search order), so activation is
  now automatic from inside the repo.

## A harmless red herring

After the fix, importing `cli` prints:

```
UserWarning: Field name "json" in "MonitorPageDiff" shadows an attribute in parent "BaseModel"
```

This comes from the `firecrawl` library's Pydantic models, not from hermes. It is a warning, not an
error, and does not affect operation. Don't waste time on it.

## Takeaways in one line each

- A repo with `env python3` shebang has no opinion about its interpreter — *you* must.
- Read `requires-python` first; version mismatches aren't pip-fixable.
- Install the project, not its imports one by one.
- Prefer a repo-local `.venv` over a shared one to make activation deterministic.
- Not every warning is a problem — triage before chasing.
