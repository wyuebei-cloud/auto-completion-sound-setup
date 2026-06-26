---
name: auto-completion-sound-setup
description: "Complete guide to setting up automatic Hermes completion sound (plugin hook + sound packs + deps). Portable — share this with anyone who wants the same setup."
version: 1.0.0
---

# Auto Completion Sound Setup

## What This Covers

Reproduce the entire Hermes automatic completion sound system on a fresh
machine: plugin hook + sound files + dependencies. Two parallel paths:

| Path | What | Use case |
|------|------|----------|
| **Plugin (primary)** | `transform_llm_output` hook → ffplay → .wav | Every final response, automatically, ~100% reliable |
| **Peon-ping packs** | `openpeon.json` sound packs | Source of the .wav files — shared resource used by both paths |

## Prerequisites

- Hermes Desktop (Windows) or Hermes CLI
- `ffplay` (from FFmpeg) — sound playback engine

### Installing ffplay (if missing)

**Windows (WinGet):**
```bash
winget install Gyan.FFmpeg
```

**macOS:**
```bash
brew install ffmpeg
```

**Linux:**
```bash
sudo apt install ffmpeg
```

Verify:
```bash
ffplay -version
```

---

## Step 1 — Get the Sound Packs

The sound packs live at `~/.claude/hooks/peon-ping/packs/` and are formatted
as CESP (Command Effect Sound Pack, `openpeon.json` manifest). Each pack has:

```
<name>/
├── openpeon.json          # Manifest with categories, files, sha256
├── .checksums
└── sounds/
    ├── construction-complete.wav
    └── ... (other .wav/.mp3)
```

### Option A: Clone from peon-ping repo (recommended)

The peon-ping project distributes a default pack set. Follow its install
instructions, or manually copy the packs directory.

### Option B: Copy from an existing install

Zip and transfer the entire `packs/` directory:
```bash
# On source machine
cd ~/.claude/hooks/peon-ping
zip -r peon-ping-packs.zip packs/
```

Then on the target machine, unzip to the same path:
```bash
mkdir -p ~/.claude/hooks/peon-ping
cd ~/.claude/hooks/peon-ping
unzip /path/to/peon-ping-packs.zip
```

### Option C: Use any .wav file (no packs needed)

If you just want one sound, you don't need the full pack structure.
Place a single .wav anywhere on disk and point the plugin to it.

---

## Step 2 — Install the Plugin

### Files to create

**`$HERMES_HOME/plugins/construction-complete/plugin.yaml`**

> ⚠️ `$HERMES_HOME` varies by install:
> - **Hermes Desktop (Windows):** `%LOCALAPPDATA%\hermes`
> - **CLI / Linux / macOS:** `~/.hermes`

```yaml
name: construction-complete
version: 1.0.0
description: Auto-play completion sound after every final agent response
author: <your-name>
provides_hooks:
  - transform_llm_output
```

**`$HERMES_HOME/plugins/construction-complete/__init__.py`**

```python
import os
import subprocess

# ── Configure this ──────────────────────────────────────────
# Point to your .wav file. Examples:
#   Pack sound:  r"~\.claude\hooks\peon-ping\packs\ra2_eva_commander\sounds\construction-complete.wav"
#   Custom:      r"C:\Users\me\sounds\my-sound.wav"
#   macOS/Linux: "~/.claude/hooks/peon-ping/packs/ra2_eva_commander/sounds/construction-complete.wav"
_SOUND_PATH = os.path.expanduser(
    r"~\.claude\hooks\peon-ping\packs\ra2_eva_commander\sounds\construction-complete.wav"
)
# ─────────────────────────────────────────────────────────────

# Resolve ffplay at import time
_FFPLAY_PATH = None
_candidates = [
    "ffplay",
    "ffplay.exe",
    # WinGet path (adjust version as needed):
    os.path.expanduser(
        r"~\\AppData\\Local\\Microsoft\\WinGet\\Packages\\Gyan.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe\\ffmpeg-8.1.1-full_build\\bin\\ffplay.exe"
    ),
]
for c in _candidates:
    try:
        subprocess.run([c, "-version"], stdout=subprocess.DEVNULL,
                       stderr=subprocess.DEVNULL, timeout=3)
        _FFPLAY_PATH = c
        break
    except (FileNotFoundError, subprocess.TimeoutExpired):
        continue


def _play() -> None:
    if not _FFPLAY_PATH or not os.path.isfile(_SOUND_PATH):
        return
    try:
        subprocess.Popen(
            [_FFPLAY_PATH, "-nodisp", "-autoexit", _SOUND_PATH],
            stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
        )
    except Exception:
        pass


def _on_transform(**kwargs) -> str:
    _play()
    return kwargs.get("response_text", "")


def register(ctx):
    ctx.register_hook("transform_llm_output", _on_transform)
```

> **⚠️ Hook callback signature:** Use `def _on_transform(**kwargs)` — the
> keyword argument is named `response_text`, NOT `text`. Writing
> `def fn(text: str, **kwargs)` will raise `TypeError`.

### Enable in config

Edit `%LOCALAPPDATA%\hermes\config.yaml` (or `~/.hermes/config.yaml`):

```yaml
plugins:
  enabled:
    - construction-complete
```

---

## Step 3 — Restart

**Must restart the entire Hermes Desktop process.** Plugin discovery only runs
at process startup. `/reset` or `/new` is NOT sufficient.

Verification after restart:
- Run `hermes plugins list` — should show `construction-complete` as enabled
- Send a simple text-only message — you should hear the sound play

---

## Step 4 — Pick a Different Sound (Customisation)

To change the sound, edit the `_SOUND_PATH` variable in `__init__.py`:

```python
_SOUND_PATH = os.path.expanduser(
    r"~\.claude\hooks\peon-ping\packs\tf2_engineer\sounds\teleporter-ready.wav"
)
```

Then restart Hermes Desktop.

---

## Sharing Checklist (for a Friend)

To give someone the exact same setup:

```
📦 auto-completion-sound.zip
├── 📂 construction-complete/            → put in $HERMES_HOME/plugins/
│   ├── plugin.yaml
│   └── __init__.py                      (edit SOUND_PATH before shipping)
├── 📂 packs/                            → put in ~/.claude/hooks/peon-ping/
│   └── ra2_eva_commander/
│       ├── openpeon.json
│       └── sounds/*.wav
└── 📄 SETUP.md
    - Install ffplay (winget/brew/apt)
    - Copy plugins/ and packs/ to the right paths
    - Enable in config.yaml
    - Restart Hermes Desktop
```

Or just share the `__init__.py` and point them to the peon-ping repo for packs.
