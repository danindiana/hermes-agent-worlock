# Lessons Learned — making Hermes use *all* local Ollama models

A post-mortem of the multi-session effort to get the `/model` picker on worlock to (a) list every
pulled Ollama model, (b) actually switch to the selected one, and (c) keep it switched. Three
distinct bugs hid behind one symptom ("it always runs `qwen3.5:4b`").

## The symptom

The `/model` picker showed only the single configured model, and even after it listed all models,
selecting one appeared to do nothing — prompts kept running the old default.

## Bug 1 — the picker never queried the live Ollama server

`list_authenticated_providers()` (`hermes_cli/model_switch.py`) built the model list **purely from
config**. For a user-defined / custom endpoint it only knew the one `model:` saved in
`custom_providers`, even though Ollama's `/v1/models` served ~30+. The code literally had:

```python
# Try to probe /v1/models if URL is set (but don't block on it)
# For now just show what we know from config
```

**Fix:** an opt-in `probe_live` flag that merges `fetch_api_models(api_key, base_url)` results into
the list. The interactive picker passes `probe_live=True`; tests stay config-only so they don't make
network calls.

**Lesson:** a comment that says "try to probe … for now just show config" is a TODO masquerading as
behaviour. Grep for the actual data source before assuming a list is dynamic.

## Bug 2 — the status bar didn't repaint after a switch

Once selection worked, the switch *succeeded* (the request really did carry the new model) but the
bottom status bar kept showing the old name. The picker called `_close_model_picker()` — which
invalidates/repaints — **before** `_apply_model_switch_result()` updated `self.model`, and nothing
invalidated afterward.

**Fix:** invalidate at the end of `_apply_model_switch_result()` (both success and error paths).

**Lesson:** order matters with deferred repaints. If you invalidate, *then* mutate the state the view
reads, the repaint captured the stale value. Invalidate after the mutation.

## Bug 3 (the real one) — per-turn re-resolution reverted the pick

This is why it "always defaulted to the same model." `_ensure_runtime_credentials()` runs **before
every turn** and calls `resolve_runtime_provider(requested=self.requested_provider)`. After a switch,
`requested_provider` was `"custom:local-ollama"`. Resolving that pool entry returns the entry's
**stored `model`** field, and these lines then overwrite the just-picked model:

```python
runtime_model = runtime.get("model")
if runtime_model and isinstance(runtime_model, str):
    self.model = runtime_model        # ← clobbers the picked model every turn
```

So: pick `qwen3:14b` → `self.model = qwen3:14b` → type a prompt → re-resolve → `self.model` reset to
the stored `qwen3.5:4b`. Verified directly:

```
resolve_runtime_provider(requested='custom:local-ollama') -> model='qwen3.5:4b'
resolve_runtime_provider(requested='custom', explicit_base_url=...) -> model=None
```

**Fix:** after a custom-endpoint switch, set `requested_provider` to the **bare `custom`** provider
(backed by the explicit base_url/api_key already set on the switch). Bare `custom` resolves to
`model=None`, so per-turn re-resolution no longer overrides the selection. Also update the matching
`custom_providers` entry's `model` so a restart resolves to the pick too.

**Lesson:** in-memory state that is re-derived each turn from config will silently undo any mid-session
mutation that doesn't also update its source. Find what the hot loop re-reads, not just where the
value is first set.

## How to verify a model switch *actually* took effect (not just the UI)

Don't trust the status bar alone. Confirm the value that hits the wire:

```python
from run_agent import AIAgent
a = AIAgent(model='qwen3.5:4b', api_key='ollama',
            base_url='http://localhost:11434/v1', provider='custom',
            api_mode='chat_completions', quiet_mode=True)
print(a._build_api_kwargs([{'role':'user','content':'hi'}])['model'])  # qwen3.5:4b
a.switch_model(new_model='qwen3:14b', new_provider='custom:local-ollama',
               api_key='ollama', base_url='http://localhost:11434/v1')
print(a._build_api_kwargs([{'role':'user','content':'hi'}])['model'])  # qwen3:14b
```

And on the server side:

```bash
curl -s http://localhost:11434/api/ps | python3 -c "import sys,json;print([m['name'] for m in json.load(sys.stdin)['models']])"
```

## Operational gotcha — edited source needs a full restart

A running `./hermes` does **not** hot-reload edited `.py` files. During this work the live session
(checked via `ps -o pid,lstart -p <pid>`) predated the edits, so the fixes "didn't work" until a full
quit-and-relaunch. Always compare the process start time against the file mtime before concluding a
fix failed.

## Takeaways in one line each

- "For now just show config" comments are unimplemented features — verify the real data source.
- Invalidate the view *after* mutating what it reads, not before.
- Anything re-resolved per-turn from config will revert mid-session changes — patch the hot path.
- Verify switches at the API-kwargs / `ollama ps` layer, not just the status bar.
- Source edits require a full restart; check process start time vs file mtime.

Related: [[lessons_learned]] (the venv/bootstrap post-mortem).
