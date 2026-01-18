# Configure Claude Code Status Line (Exact Replication)

Configure my Claude Code status line to show: time, context utilization bar with percentage, project name, directory, git status, and venv. Follow these steps **exactly**.

## Prerequisites

First, verify `jq` is installed:
```bash
which jq
```

If not installed, run:
```bash
brew install jq
```

## Step 1: Create the scripts directory

```bash
mkdir -p ~/.claude/scripts
```

## Step 2: Create the statusline script

Create the file `~/.claude/scripts/statusline.sh` with **exactly** this content:

```bash
#!/usr/bin/env bash

# Claude Code Statusline - Minimal, ADHD-friendly
# Colors: meaningful, not decorative
# - Dim gray: ambient info (time, stable state)
# - White: current focus (directory)
# - Cyan: context anchors (project, venv)
# - Green: good state
# - Yellow: attention needed
# - Red: action required

set -euo pipefail

# ANSI colors
DIM='\033[2m'
WHITE='\033[1;37m'
CYAN='\033[36m'
GREEN='\033[32m'
YELLOW='\033[33m'
RED='\033[31m'
RESET='\033[0m'

# Read context from Claude Code (stdin is JSON)
CONTEXT=$(cat 2>/dev/null || echo '{}')
CWD=$(echo "$CONTEXT" | jq -r '.cwd // empty' 2>/dev/null || pwd)

cd "$CWD" 2>/dev/null || cd ~

# ─────────────────────────────────────────────────────────────
# CONTEXT USAGE (most important - color by your 50% threshold)
# Uses current_usage (actual context) not total_*_tokens (cumulative session)
# ─────────────────────────────────────────────────────────────
context_display() {
    local size=$(echo "$CONTEXT" | jq -r '.context_window.context_window_size // 0' 2>/dev/null)
    [[ "$size" == "0" || -z "$size" ]] && return

    local usage=$(echo "$CONTEXT" | jq '.context_window.current_usage // null' 2>/dev/null)
    [[ "$usage" == "null" ]] && return

    local total=$(echo "$usage" | jq '.input_tokens + .cache_creation_input_tokens + .cache_read_input_tokens + .output_tokens')
    [[ -z "$total" || "$total" == "null" ]] && return

    local pct=$((total * 100 / size))

    # Color by threshold (conservative - catch degradation early)
    local color="$GREEN"
    [[ $pct -ge 33 ]] && color="$YELLOW"
    [[ $pct -ge 50 ]] && color="$RED"

    # Visual bar (10 chars)
    local filled=$((pct / 10))
    [[ $filled -gt 10 ]] && filled=10
    local empty=$((10 - filled))

    local bar=""
    for ((i=0; i<filled; i++)); do bar+="█"; done
    for ((i=0; i<empty; i++)); do bar+="░"; done

    echo -e "${color}${bar} ${pct}%${RESET}"
}

CTX=$(context_display)

# ─────────────────────────────────────────────────────────────
# TIME (dim - ambient, doesn't need attention)
# ─────────────────────────────────────────────────────────────
TIME="${DIM}$(date +%H:%M)${RESET}"

# ─────────────────────────────────────────────────────────────
# PROJECT (cyan - context anchor)
# Find nearest CLAUDE.md or AGENTS.md going up the tree
# ─────────────────────────────────────────────────────────────
find_project() {
    local dir="$1"
    while [[ "$dir" != "/" && "$dir" != "$HOME" ]]; do
        if [[ -f "$dir/CLAUDE.md" || -f "$dir/AGENTS.md" ]]; then
            basename "$dir"
            return
        fi
        dir=$(dirname "$dir")
    done
    echo ""
}

PROJECT_NAME=$(find_project "$CWD")
if [[ -n "$PROJECT_NAME" ]]; then
    PROJECT="${CYAN}${PROJECT_NAME}${RESET}"
else
    PROJECT=""
fi

# ─────────────────────────────────────────────────────────────
# DIRECTORY (white - current focus)
# Smart abbreviation: remove ~/projects prefix
# ─────────────────────────────────────────────────────────────
smart_dir() {
    local dir="$1"

    # Remove ~/projects prefix
    dir="${dir/#$HOME\/projects\//}"
    dir="${dir/#$HOME\/Projects\//}"
    dir="${dir/#$HOME/~}"

    # If same as project, just show subdirectory or "."
    if [[ -n "$PROJECT_NAME" ]]; then
        if [[ "$dir" == *"$PROJECT_NAME"* ]]; then
            local after="${dir#*$PROJECT_NAME}"
            if [[ -z "$after" || "$after" == "/" ]]; then
                echo "."
                return
            else
                echo ".${after}"
                return
            fi
        fi
    fi

    echo "$dir"
}

DIR="${WHITE}$(smart_dir "$CWD")${RESET}"

# ─────────────────────────────────────────────────────────────
# GIT STATUS (color-coded by urgency)
# ─────────────────────────────────────────────────────────────
git_status() {
    if ! git rev-parse --is-inside-work-tree &>/dev/null; then
        echo ""
        return
    fi

    local branch=$(git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD 2>/dev/null)
    local status=""
    local color="$GREEN"

    # Ahead/behind origin
    local upstream=$(git rev-parse --abbrev-ref @{upstream} 2>/dev/null)
    if [[ -n "$upstream" ]]; then
        local ahead=$(git rev-list --count @{upstream}..HEAD 2>/dev/null || echo 0)
        local behind=$(git rev-list --count HEAD..@{upstream} 2>/dev/null || echo 0)
        [[ $ahead -gt 0 ]] && status+="↑${ahead}" && color="$YELLOW"
        [[ $behind -gt 0 ]] && status+="↓${behind}" && color="$YELLOW"
        [[ $ahead -gt 0 && $behind -gt 0 ]] && color="$RED"
    fi

    # Working tree status
    local staged=$(git diff --cached --numstat 2>/dev/null | wc -l | tr -d ' ')
    local modified=$(git diff --numstat 2>/dev/null | wc -l | tr -d ' ')
    local untracked=$(git ls-files --others --exclude-standard 2>/dev/null | wc -l | tr -d ' ')
    local conflicts=$(git diff --name-only --diff-filter=U 2>/dev/null | wc -l | tr -d ' ')

    [[ $staged -gt 0 ]] && status+="●${staged}" && [[ "$color" == "$GREEN" ]] && color="$YELLOW"
    [[ $modified -gt 0 ]] && status+="✚${modified}" && [[ "$color" == "$GREEN" ]] && color="$YELLOW"
    [[ $untracked -gt 0 ]] && status+="…"
    [[ $conflicts -gt 0 ]] && status+="✖${conflicts}" && color="$RED"

    [[ -n "$status" ]] && status=" ${status}"

    echo -e "${color}${branch}${status}${RESET}"
}

GIT=$(git_status)

# ─────────────────────────────────────────────────────────────
# VENV (cyan - context anchor)
# ─────────────────────────────────────────────────────────────
if [[ -n "${VIRTUAL_ENV:-}" ]]; then
    VENV="${CYAN}$(basename "$VIRTUAL_ENV")${RESET}"
elif [[ -n "${CONDA_DEFAULT_ENV:-}" ]]; then
    VENV="${CYAN}${CONDA_DEFAULT_ENV}${RESET}"
else
    VENV=""
fi

# ─────────────────────────────────────────────────────────────
# OUTPUT - Context first (most actionable)
# ─────────────────────────────────────────────────────────────
SEP="${DIM}│${RESET}"
OUT="$TIME"

[[ -n "$CTX" ]] && OUT+=" $SEP $CTX"

if [[ -n "$PROJECT" ]]; then
    OUT+=" $SEP ${PROJECT}${DIM}:${RESET}${DIR}"
else
    OUT+=" $SEP $DIR"
fi

[[ -n "$GIT" ]] && OUT+=" $SEP $GIT"
[[ -n "$VENV" ]] && OUT+=" $SEP $VENV"

echo -e "$OUT"
```

## Step 3: Make the script executable

```bash
chmod +x ~/.claude/scripts/statusline.sh
```

## Step 4: Configure Claude Code settings

Read the current settings file:
```bash
cat ~/.claude/settings.json
```

Then edit `~/.claude/settings.json` to add the `statusLine` configuration. The final file must contain this key at the top level:

```json
{
  "statusLine": {
    "type": "command",
    "command": "/Users/YOUR_USERNAME/.claude/scripts/statusline.sh"
  }
}
```

**Important**: Replace `YOUR_USERNAME` with the actual username. Get it with `whoami`.

If the settings file already has other keys (like `model`, `mcpServers`), merge them. The `statusLine` object should be a sibling to other top-level keys.

## Step 5: Verify

1. Test the script manually:
   ```bash
   echo '{"cwd":"'$(pwd)'"}' | ~/.claude/scripts/statusline.sh
   ```
   You should see output like: `14:30 │ ~/projects │ main`

2. Restart Claude Code (exit and reopen)

3. The status line should now show:
   - **Time** (dim gray): `HH:MM`
   - **Context bar** (colored): `████░░░░░░ 35%` - green <33%, yellow 33-50%, red >50%
   - **Project:Dir** (cyan:white): `myproject:.` or `myproject:./subdir`
   - **Git** (colored): `main` (green=clean), `main ↑2` (yellow=ahead), `main ✚3` (yellow=modified)
   - **Venv** (cyan): `venv-name` (only if active)

## Troubleshooting

If context utilization doesn't appear:
- The bar only shows after the first API round-trip (when Claude responds)
- Verify jq can parse the JSON: `echo '{"context_window":{"context_window_size":200000,"current_usage":{"input_tokens":1000,"cache_creation_input_tokens":0,"cache_read_input_tokens":0,"output_tokens":500}}}' | jq '.context_window.current_usage'`

If nothing shows:
- Check the script runs without errors: `bash -x ~/.claude/scripts/statusline.sh < /dev/null`
- Verify the path in settings.json is absolute and correct
