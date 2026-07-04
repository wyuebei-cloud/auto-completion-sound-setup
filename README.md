# Auto Completion Sound Setup

[![skills.sh](https://skills.sh/b/wyuebei-cloud/auto-completion-sound-setup)](https://skills.sh/wyuebei-cloud/auto-completion-sound-setup)

> Hermes Agent skill: automatic completion sound after every final response — zero LLM involvement, R5 compliant.

**One-time setup:**
1. Install ffplay (`winget install Gyan.FFmpeg` / `brew install ffmpeg`)
2. Get sound packs from [peon-ping](https://github.com/NousResearch/hermes-agent) or use your own .wav
3. Copy the `construction-complete/` plugin to `$HERMES_HOME/plugins/`
4. Enable in `config.yaml`: `plugins.enabled: [construction-complete]`
5. Restart Hermes Desktop

See `SKILL.md` or `references/plugin-hook-approach.md` for full details.

## Install via Hermes

```bash
hermes skills install wyuebei-cloud/auto-completion-sound-setup
```
