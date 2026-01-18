  Install and configure high-performance CLI alternatives

  You're on macOS. Install these Rust-based CLI tools and configure yourself to use them:
  ┌─────────┬──────────┬─────────────────────────────┐
  │  Tool   │ Replaces │            Usage            │
  ├─────────┼──────────┼─────────────────────────────┤
  │ ripgrep │ grep     │ rg - respects .gitignore    │
  ├─────────┼──────────┼─────────────────────────────┤
  │ fd      │ find     │ fd -e py finds Python files │
  ├─────────┼──────────┼─────────────────────────────┤
  │ bat     │ cat      │ syntax highlighting         │
  ├─────────┼──────────┼─────────────────────────────┤
  │ sd      │ sed      │ sd "old" "new" file         │
  ├─────────┼──────────┼─────────────────────────────┤
  │ dust    │ du       │ visual disk usage           │
  ├─────────┼──────────┼─────────────────────────────┤
  │ eza     │ ls       │ eza --tree for tree view    │
  ├─────────┼──────────┼─────────────────────────────┤
  │ tokei   │ wc -l    │ LOC by language             │
  └─────────┴──────────┴─────────────────────────────┘

  
  
  Tasks:

  1. Check for Homebrew: Run which brew. If missing, ask the user: "Homebrew isn't installed. These tools require it. Would you like me to install Homebrew, or skip this setup?" If they decline, stop here gracefully.
  
  2. Install Homebrew (only if user approves):
      /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
      Then configure PATH per the post-install instructions (Apple Silicon: /opt/homebrew/bin/brew).
  
  3. Install all tools:
      brew install ripgrep fd bat sd dust eza tokei
  
  4. Add to ~/.claude/CLAUDE.md (create if missing, append if exists):
  ```
  ## High-Performance CLI
  | Instead of | Use | Notes |
  |------------|-----|-------|
  | `grep` | `rg` | respects .gitignore |
  | `find` | `fd` | `fd -e py` |
  | `cat` | `bat` | syntax highlighting |
  | `sed` | `sd` | `sd "old" "new" file` |
  | `du` | `dust` | visual |
  | `ls` | `eza` | `eza --tree` |
  | `wc -l` (code) | `tokei` | LOC by language |
  ```

  5. Optional shell aliases — ask user if they want system-wide aliases in ~/.zshrc:
  ```
  # Modern CLI alternatives
  alias grep='rg'
  alias find='fd'
  alias cat='bat'
  alias ls='eza'
  alias du='dust'
  alias tree='eza --tree'
  ```
     (Don't alias sed→sd — syntax differs enough to break scripts.)

  6. Verify each tool with --version.
