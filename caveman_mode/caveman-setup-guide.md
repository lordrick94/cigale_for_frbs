# Caveman Mode Setup Guide

**"why use many token when few do trick"**

Caveman mode is a Claude Code plugin that makes Claude respond in compressed prose, cutting ~75% of output tokens while maintaining full technical accuracy.

---

## What is Caveman Mode?

Caveman mode transforms verbose AI responses:

| Normal Claude | Caveman Claude |
|---------------|----------------|
| "Sure! I'd be happy to help you with that. The issue you're experiencing is most likely caused by your authentication middleware not properly validating the token expiry." | "Bug in auth middleware. Token expiry check bad. Fix:" |

Same technical accuracy. 75% fewer tokens. Faster responses.

---

## Installation Methods

### Method 1: Claude Code Plugin (Recommended)

Run these two commands:

```bash
claude plugin marketplace add JuliusBrussee/caveman
claude plugin install caveman@caveman
```

This installs:
- Auto-activation hooks (caveman loads every session)
- All caveman skills (`/caveman`, `/caveman-commit`, `/caveman-review`, etc.)
- Statusline badge showing current mode

### Method 2: Standalone Hooks (Without Plugin System)

**macOS / Linux / WSL:**
```bash
bash <(curl -s https://raw.githubusercontent.com/JuliusBrussee/caveman/main/hooks/install.sh)
```

**Windows (PowerShell):**
```powershell
irm https://raw.githubusercontent.com/JuliusBrussee/caveman/main/hooks/install.ps1 | iex
```

**From a local clone:**
```bash
# macOS / Linux
bash hooks/install.sh

# Windows
powershell -File hooks\install.ps1
```

**Uninstall:**
```bash
bash hooks/uninstall.sh
# or on Windows:
powershell -File hooks\uninstall.ps1
```

### Method 3: Other AI Agents

Caveman works with many agents via `npx skills`:

| Agent | Install Command |
|-------|-----------------|
| Cursor | `npx skills add JuliusBrussee/caveman -a cursor` |
| Windsurf | `npx skills add JuliusBrussee/caveman -a windsurf` |
| Copilot | `npx skills add JuliusBrussee/caveman -a github-copilot` |
| Cline | `npx skills add JuliusBrussee/caveman -a cline` |
| Gemini CLI | `gemini extensions install https://github.com/JuliusBrussee/caveman` |
| Any other | `npx skills add JuliusBrussee/caveman` |

**Windows note:** If symlinks fail, add `--copy`: `npx skills add JuliusBrussee/caveman --copy`

---

## Requirements

- **Claude Code** (for plugin/hooks install)
- **Node.js** (required for hooks installation)
- **Git Bash / WSL** on Windows (for bash scripts)

---

## Usage

### Activate Caveman Mode

After installation, caveman auto-activates on session start. You can also trigger it manually:

- `/caveman` - Activate caveman mode
- "talk like caveman"
- "caveman mode"
- "less tokens please"

### Stop Caveman Mode

- "stop caveman"
- "normal mode"

### Intensity Levels

| Level | Command | Description |
|-------|---------|-------------|
| Lite | `/caveman lite` | Drop filler, keep grammar. Professional but no fluff |
| Full | `/caveman full` | Default. Drop articles, fragments, full grunt |
| Ultra | `/caveman ultra` | Maximum compression. Telegraphic. Abbreviate everything |

### Classical Chinese (Wenyan) Mode

| Level | Command | Description |
|-------|---------|-------------|
| Wenyan-Lite | `/caveman wenyan-lite` | Semi-classical |
| Wenyan-Full | `/caveman wenyan` | Full classical terseness |
| Wenyan-Ultra | `/caveman wenyan-ultra` | Extreme ancient scholar mode |

---

## Additional Skills

| Skill | Command | What it does |
|-------|---------|--------------|
| Caveman Commit | `/caveman-commit` | Terse commit messages, Conventional Commits format, ≤50 char subject |
| Caveman Review | `/caveman-review` | One-line PR comments: `L42: bug: user null. Add guard.` |
| Caveman Compress | `/caveman:compress <file>` | Compress memory files (CLAUDE.md) to save input tokens |
| Caveman Help | `/caveman-help` | Quick-reference card for all commands |

---

## File Locations (After Installation)

| File | Purpose |
|------|---------|
| `~/.claude/hooks/caveman-activate.js` | SessionStart hook - auto-loads rules |
| `~/.claude/hooks/caveman-mode-tracker.js` | Tracks mode changes |
| `~/.claude/hooks/caveman-statusline.sh` | Statusline badge display |
| `~/.claude/hooks/caveman-config.js` | Configuration options |
| `~/.claude/.caveman-active` | Flag file storing current mode |
| `~/.claude/settings.json` | Hook registrations |

---

## Statusline Badge

After installation, your Claude Code statusline shows:
- `[CAVEMAN]` - Full mode active
- `[CAVEMAN:LITE]` - Lite mode
- `[CAVEMAN:ULTRA]` - Ultra mode
- etc.

If you have an existing custom statusline, the installer won't overwrite it. See `hooks/README.md` in the repo for merge instructions.

---

## Always-On for Agents Without Hooks

For agents that don't support hooks (most except Claude Code/Codex/Gemini), paste this into your agent's system prompt or rules file:

```
Terse like caveman. Technical substance exact. Only fluff die.
Drop: articles, filler (just/really/basically), pleasantries, hedging.
Fragments OK. Short synonyms. Code unchanged.
Pattern: [thing] [action] [reason]. [next step].
ACTIVE EVERY RESPONSE. No revert after many turns. No filler drift.
Code/commits/PRs: normal. Off: "stop caveman" / "normal mode".
```

---

## Troubleshooting

**Caveman not activating:**
1. Restart Claude Code after installation
2. Check that hooks exist in `~/.claude/hooks/`
3. Verify `~/.claude/settings.json` has the hook registrations

**Reinstall/repair:**
```bash
bash hooks/install.sh --force
```

**Check installation status:**
```bash
ls -la ~/.claude/hooks/caveman*
cat ~/.claude/settings.json | grep -A5 caveman
```

---

## Source

- GitHub: https://github.com/JuliusBrussee/caveman
- License: MIT

---

**Same fix. 75% less word. Brain still big.**
