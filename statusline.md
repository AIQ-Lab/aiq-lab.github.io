  Set up a custom Claude Code status line with the following requirements:

  1. Create the script at ~/.claude/scripts/statusline.sh:
    - Shows context usage as a visual bar (█░) with percentage, colored by threshold:
        - Green: <33%
      - Yellow: 33-50%
      - Red: ≥50%
    - Dim time (HH:MM)
    - Project name (cyan) found by walking up to nearest CLAUDE.md
    - Smart directory path (white) that removes ~/projects prefix and shows relative to project
    - Git branch with status indicators (●staged ✚modified …untracked ↑ahead ↓behind ✖conflicts)
    - Active venv/conda env (cyan)
    - Separated by dim │ pipe characters
  2. Configure Claude Code - add to ~/.claude/settings.json:
  {
    "statusLine": {
      "type": "command",
      "command": "~/.claude/scripts/statusline.sh"
    }
  }

  3. Key design principles:
    - ADHD-friendly: context usage first (most actionable), then location/git
    - Color meaning: dim=ambient, white=focus, cyan=anchors, green/yellow/red=urgency
    - Minimal: only show what needs attention
    - Script reads JSON from stdin with .cwd and .context_window fields

  Create the script with set -euo pipefail, parse JSON with jq, and make it executable.
