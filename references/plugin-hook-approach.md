# Hermes Plugin Hook Approach — Deterministic Sound Playback

## Rationale

Relying on LLM attention to call `play_sound` after every final response has a
~90% failure rate (verified empirically). Per **R5 (确定性逻辑不用LLM)**,
deterministic actions must use hard code, not model attention.

The solution: a Hermes plugin that hooks into the response pipeline at the
framework level, bypassing the LLM entirely.

## Architecture

```
LLM produces final text (no tool_calls)
  → agent loop detects completion
    → transform_llm_output hook fires (in Hermes kernel)
      → plugin plays .wav via ffplay (Popen, fire-and-forget)
        → text delivered to user + sound plays simultaneously
```

**Zero LLM involvement.** The hook fires unconditionally — the LLM never
decides whether to play the sound.

## Plugin Files

> **Path note:** ``$HERMES_HOME`` resolves to ``get_hermes_home()``.
> - Hermes Desktop (Windows): ``%LOCALAPPDATA%\\hermes``
> - CLI / Linux / macOS: ``~/.hermes``
>
> Place plugin files under ``$HERMES_HOME/plugins/construction-complete/``.

### `$HERMES_HOME/plugins/construction-complete/plugin.yaml`

```yaml
name: construction-complete
version: 1.0.0
description: Auto-play "Construction Complete" sound after every final agent response
author: UB
provides_hooks:
  - transform_llm_output
```

### `$HERMES_HOME/plugins/construction-complete/__init__.py`

```python
import os, subprocess, sys

_SOUND_PATH = os.path.expanduser(
    r"~\.claude\hooks\peon-ping\packs
a2_eva_commander\sounds\construction-complete.wav"
)

# Suppress console window on Windows (ffplay.exe is a console-subsystem app)
_CREATE_NO_WINDOW = 0x08000000 if sys.platform == "win32" else 0
_HIDE_FLAGS = {"creationflags": _CREATE_NO_WINDOW} if _CREATE_NO_WINDOW else {}

_FFPLAY = "ffplay"  # resolve at import time


def _play() -> None:
    if not os.path.isfile(_SOUND_PATH):
        return
    try:
        subprocess.Popen(
            [_FFPLAY, "-nodisp", "-autoexit", _SOUND_PATH],
            stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
            **_HIDE_FLAGS,
        )
    except Exception:
        pass


def _on_transform(**kwargs) -> str:
    """transform_llm_output hook — fires once per turn, after tool loop completes.
    The response text is passed as ``response_text`` keyword argument.
    """
    _play()
    return kwargs.get("response_text", "")  # return unchanged


def register(ctx):
    ctx.register_hook("transform_llm_output", _on_transform)
```

> **⚠️ Critical — Callback signature:**
> The `transform_llm_output` hook is invoked with keyword arguments only:
> `response_text`, `session_id`, `model`, `platform`. The parameter is named
> **`response_text`**, NOT `text`. Using `def _on_transform(text: str, **kwargs)`
> will raise `TypeError: missing 1 required positional argument: 'text'`.
> Always use `**kwargs` and access `kwargs.get("response_text", "")`.

### Config

```yaml
plugins:
  enabled:
    - construction-complete
```

## Available Hooks Reference

From `hermes_cli/plugins.py`:

| Hook | Fires when | Returns |
|------|-----------|---------|
| `pre_tool_call` | Before any tool executes | `{"action": "block", ...}` to veto |
| `post_tool_call` | After any tool returns | ignored |
| `transform_llm_output` | After tool loop, before final response delivered | modified text (or unchanged) |
| `pre_llm_call` | Once per turn, before LLM call | `{"context": str}` to inject |
| `post_llm_call` | Once per turn, after LLM call | ignored |
| `on_session_start` | New session created | ignored |
| `on_session_end` | Session ends | ignored |

*This is the sole sound path — the old MCP play_sound approach has been removed.
See the `peon-ping` skill's v2→v3 migration notes for rationale.*
