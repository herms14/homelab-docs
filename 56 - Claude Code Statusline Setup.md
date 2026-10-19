---
tags: [homelab, claude-code, tooling, setup]
created: 2026-05-23
updated: 2026-05-23
type: tutorial
status: evergreen
---

# Claude Code Statusline Setup

> Configure Claude Code to show a context window progress bar in the status line, plus Oh My Posh in the terminal prompt.

**Source repo**: https://github.com/herms14/claude-code-session-checker

Related: [00 - Homelab Index](00%20-%20Homelab%20Index.md) | [16 - SSH Configuration](16%20-%20SSH%20Configuration.md)

---

## What This Does

Adds a live context window progress bar to the bottom of every Claude Code session:

```
[████████░░░░░░░░░░░░] 42%  Claude Sonnet 4.6  my-project
```

**Color coding:**
- 🟢 Green — below 50% used
- 🟡 Yellow — 50–89% used
- 🔴 Red — 90%+ used (time to start a new session)

Also configures **Oh My Posh** in Git Bash so the shell prompt renders correctly inside Claude Code's terminal.

---

## Prerequisites

| Tool | Required | Notes |
|------|----------|-------|
| Claude Code CLI | ✅ | `claude --version` to verify |
| Git for Windows | ✅ | Provides the Git Bash shell Claude Code uses |
| Python 3.x | ✅ | Used to parse the JSON status data |
| Oh My Posh | Optional | For themed shell prompt |

---

## Step 1 — Find Your Python Path

Claude Code runs on Git Bash. Find where Python is installed:

```bash
# In Git Bash
which python3 || which python
# or check common Windows paths:
ls /c/Python3*/python.exe 2>/dev/null
```

Note the full path — you'll need it in Step 3. On this machine it is:
```
/c/Python314/python.exe
```

---

## Step 2 — Clone the Repo and Copy the Script

```bash
# Clone the repo
git clone https://github.com/herms14/claude-code-session-checker.git
cd claude-code-session-checker

# Copy script to Claude Code config directory
mkdir -p ~/.claude
cp statusline-command.sh ~/.claude/statusline-command.sh
```

Then open `~/.claude/statusline-command.sh` and update the `PYTHON=` line at the top to match your Python path from Step 1.

---

### Script Reference

The full script (`statusline-command.sh`) is maintained in the repo. Key section to configure:

```bash
# The one line you must update after cloning:
PYTHON="/c/Python314/python.exe"   # <-- change to your Python path
```

Full script for reference:

```bash
#!/usr/bin/env bash
# Claude Code statusline — context window progress bar

PYTHON="/c/Python314/python.exe"   # <-- update this path for your machine
input=$(cat)

PY='
import sys, json
try:
    d = json.loads(sys.argv[1])
    cw = d.get("context_window", {})
    m  = d.get("model", {})
    ws = d.get("workspace", {})
    print(cw.get("used_percentage",  "") or "")
    print(cw.get("remaining_percentage", "") or "")
    print(m.get("display_name", "") or "")
    print(ws.get("current_dir", "") or "")
except:
    print(""); print(""); print(""); print("")
'

mapfile -t parsed < <("$PYTHON" -c "$PY" "$input" 2>/dev/null | tr -d '\r')
used="${parsed[0]}"
model="${parsed[2]}"
cwd="${parsed[3]}"

dir=$(basename "$cwd")

if [[ "$used" =~ ^[0-9]+([.][0-9]+)?$ ]]; then
  pct=${used%.*}

  if   (( pct >= 90 )); then color="\033[0;31m"   # red
  elif (( pct >= 50 )); then color="\033[0;33m"   # yellow
  else                        color="\033[0;32m"   # green
  fi
  reset="\033[0m"

  filled=$(( pct * 20 / 100 ))
  empty=$(( 20 - filled ))
  bar=""
  for (( i=0; i<filled; i++ )); do bar+="█"; done
  for (( i=0; i<empty;  i++ )); do bar+="░"; done

  ctx_part="${color}[${bar}] ${pct}%${reset}"
else
  ctx_part="[░░░░░░░░░░░░░░░░░░░░] --%"
fi

model_part="${model:+  ${model}}"
dir_part="${dir:+  ${dir}}"

printf "%b%s%s\n" "$ctx_part" "$model_part" "$dir_part"
```

---

## Step 3 — Register It in Claude Code Settings

Edit `~/.claude/settings.json` (create it if it doesn't exist) and add the `statusLine` block:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash C:/Users/<you>/.claude/statusline-command.sh"
  }
}
```

> Use forward slashes in the path, even on Windows.

---

## Step 4 — Oh My Posh in Git Bash (Optional)

Claude Code uses Git Bash for shell commands, not PowerShell. Oh My Posh must be initialized separately for bash.

Create `~/.bashrc` (`C:\Users\<you>\.bashrc`):

```bash
# Oh My Posh — use your existing theme
OMP_CONFIG="/c/Users/<you>/path/to/your-theme.omp.json"

if command -v oh-my-posh &>/dev/null && -f "$OMP_CONFIG"; then
  eval "$(oh-my-posh init bash --config "$OMP_CONFIG")"
fi
```

Create `~/.bash_profile` (`C:\Users\<you>\.bash_profile`):

```bash
# Load .bashrc for login shells too
.bashrc && source ~/.bashrc
```

On this machine the theme is at:
```
C:\Users\herms14\OneDrive - Microsoft\Documents\WindowsPowerShell\azure-terraform.omp.json
```

---

## Step 5 — Test

Verify the script parses correctly before opening Claude Code:

```bash
# In Git Bash — test with mock data
echo '{"context_window":{"used_percentage":42,"remaining_percentage":58},"model":{"display_name":"Claude Sonnet 4.6"},"workspace":{"current_dir":"/c/homelab"}}' \
  | bash ~/.claude/statusline-command.sh
```

Expected output (with ANSI color codes):
```
[████████░░░░░░░░░░░░] 42%  Claude Sonnet 4.6  homelab
```

---

## Troubleshooting

### `ctx: --` always shows

The script could not parse the JSON. Check:

1. **Python path wrong** — run `ls /c/Python3*/python.exe` and update `PYTHON=` in the script
2. **jq not installed** — this script uses Python intentionally (jq not required)
3. **Fresh session** — `--` is correct until you send your first message; it populates after the first API call

### Oh My Posh not showing in Claude Code terminal

Claude Code uses Git Bash, not PowerShell. Your `Microsoft.PowerShell_profile.ps1` does not apply here. Make sure `~/.bashrc` exists with the `oh-my-posh init bash` line (Step 4).

### Windows line ending issues (`\r` in variables)

Python on Windows outputs `\r\n`. The script already handles this with `tr -d '\r'`. If you see garbled output, verify that line is present in the script.

---

## File Locations Summary

| File | Purpose |
|------|---------|
| `~/.claude/settings.json` | Registers the statusline command with Claude Code |
| `~/.claude/statusline-command.sh` | The progress bar script |
| `~/.bashrc` | Initializes Oh My Posh for Git Bash (interactive shells) |
| `~/.bash_profile` | Sources `.bashrc` for login shells |

---

## Related Documentation

- [16 - SSH Configuration](16%20-%20SSH%20Configuration.md) — SSH key setup
- [Homelab Changelog](Homelab%20Changelog.md) — Change history

---

*Last configured: 2026-05-23*
