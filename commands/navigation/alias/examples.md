# alias — Practical Examples

> Real-world alias collections for everyday productivity, safety, and workflow speed.

---

## Table of Contents

- [Basic Shortcuts](#basic-shortcuts)
- [Safety Aliases](#safety-aliases)
- [Navigation Shortcuts](#navigation-shortcuts)
- [Git Aliases](#git-aliases)
- [System Monitoring Shortcuts](#system-monitoring-shortcuts)
- [Package Management Shortcuts](#package-management-shortcuts)
- [Network & Connectivity](#network--connectivity)
- [Docker & Container Shortcuts](#docker--container-shortcuts)
- [Productivity Combos](#productivity-combos)
- [Temporary / One-Off Aliases](#temporary--one-off-aliases)
- [Organizing a Full Alias File](#organizing-a-full-alias-file)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Shortcuts

```bash
# Better default listing
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'

# Human-readable sizes everywhere
alias df='df -h'
alias du='du -h'
alias free='free -h'

# Colorized output by default
alias grep='grep --color=auto'
alias ls='ls --color=auto'

# Clear the screen quickly
alias c='clear'

# Show a directory tree, ignoring common clutter
alias tree='tree -I "node_modules|.git|__pycache__"'
```

---

## Safety Aliases

```bash
# Confirm before destructive operations — the single most common
# safety-oriented alias setup
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Prevent accidental recursive deletion of root-level paths (extra caution)
alias rm='rm -I'          # GNU-specific: prompts only once for 3+ files, or when recursive

# "Dry run" style alias for rsync — always preview before actually syncing
alias rsync-preview='rsync -avzn'

# Prevent accidentally overwriting files with redirection mistakes
set -o noclobber           # not an alias, but commonly paired with the above
alias no-overwrite='set -o noclobber'
```

---

## Navigation Shortcuts

```bash
# Jump up directory levels quickly
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'

# Go straight to commonly used project/work directories
alias proj='cd ~/projects'
alias dl='cd ~/Downloads'
alias docs='cd ~/Documents'

# Return to the previous directory instantly
alias back='cd -'

# Show current path clearly
alias where='pwd'
```

---

## Git Aliases

```bash
alias gs='git status'
alias ga='git add'
alias gaa='git add --all'
alias gc='git commit -m'
alias gp='git push'
alias gpl='git pull'
alias gco='git checkout'
alias gb='git branch'
alias gl='git log --oneline --graph --decorate --all'
alias gd='git diff'
alias gcl='git clone'

# Combine common workflows
alias gac='git add --all && git commit -m'
# Usage: gac "commit message"
```

---

## System Monitoring Shortcuts

```bash
# Quick resource overview
alias meminfo='free -h -l -t'
alias cpuinfo='lscpu'
alias diskspace='df -h --total'

# Top processes by memory / CPU usage
alias psmem='ps auxf | sort -nr -k 4 | head -10'
alias pscpu='ps auxf | sort -nr -k 3 | head -10'

# Watch system load continuously
alias watchload='watch -n 1 uptime'

# Quick listening ports check
alias ports='sudo netstat -tulanp'
alias ports2='sudo ss -tulanp'
```

---

## Package Management Shortcuts

```bash
# Debian/Ubuntu
alias update='sudo apt update && sudo apt upgrade -y'
alias install='sudo apt install'
alias remove='sudo apt remove'
alias search='apt search'
alias cleanup='sudo apt autoremove && sudo apt autoclean'

# Fedora/RHEL
alias update='sudo dnf upgrade --refresh'
alias install='sudo dnf install'

# Arch
alias update='sudo pacman -Syu'
alias install='sudo pacman -S'
```

---

## Network & Connectivity

```bash
# Quick external IP check
alias myip='curl -s ifconfig.me'

# Ping with a sane default count instead of running forever
alias ping='ping -c 5'

# Flush DNS cache (systemd-resolved based systems)
alias flushdns='sudo systemd-resolve --flush-caches'

# Quick reachability test to a common reliable host
alias netcheck='ping -c 3 8.8.8.8'
```

---

## Docker & Container Shortcuts

```bash
alias d='docker'
alias dc='docker compose'
alias dps='docker ps'
alias dpsa='docker ps -a'
alias dimg='docker images'
alias dstop='docker stop $(docker ps -q)'                 # stop ALL running containers
alias dclean='docker system prune -af'                    # remove unused data
alias dlog='docker logs -f'

# Usage:
# dlog my_container_name
```

---

## Productivity Combos

```bash
# Combine multiple steps into one memorable command
alias update-all='sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y'

# Create a directory and immediately move into it (needs a FUNCTION,
# not a plain alias — shown here for contrast, see README.md)
mkcd() { mkdir -p "$1" && cd "$1"; }

# Reload shell config without closing the terminal
alias reload='source ~/.bashrc'

# Quickly edit and reload aliases in one motion
alias aliases='nano ~/.bash_aliases && source ~/.bash_aliases'

# Show all currently defined aliases, nicely sorted
alias showaliases='alias | sort'
```

---

## Temporary / One-Off Aliases

```bash
# Define an alias just for this session (won't survive closing the terminal)
alias tmp-debug='ls -la /var/log | grep error'
tmp-debug

# Remove it once you're done, if you want a clean session
unalias tmp-debug

# Quickly bypass an alias for a single command without removing it
alias rm='rm -i'
\rm unwanted_file.txt        # runs the REAL rm, skips the -i confirmation just this once
```

---

## Organizing a Full Alias File

```bash
# ~/.bash_aliases — a clean, dedicated file for all custom aliases

# --- Navigation ---
alias ..='cd ..'
alias ...='cd ../..'

# --- Listing ---
alias ll='ls -alF'
alias la='ls -A'

# --- Safety ---
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# --- Git ---
alias gs='git status'
alias gc='git commit -m'
alias gp='git push'

# --- System ---
alias df='df -h'
alias meminfo='free -h'

# --- Load this file from ~/.bashrc ---
# if [ -f ~/.bash_aliases ]; then
#     . ~/.bash_aliases
# fi
```

```bash
# Make the new aliases active immediately, without restarting the terminal
source ~/.bashrc
```

---

## Real-World Recipes

```bash
# --- New Machine Setup ---

cat >> ~/.bashrc << 'EOF'
alias ll='ls -alF'
alias update='sudo apt update && sudo apt upgrade -y'
alias gs='git status'
alias rm='rm -i'
EOF
source ~/.bashrc

# --- Sharing Your Alias Setup Across Machines ---

# Keep aliases in a dotfiles repo and symlink them into place
git clone https://example.com/dotfiles.git ~/dotfiles
ln -sf ~/dotfiles/.bash_aliases ~/.bash_aliases
echo '[ -f ~/.bash_aliases ] && . ~/.bash_aliases' >> ~/.bashrc
source ~/.bashrc

# --- Auditing What's Currently Defined ---

alias | wc -l                # count how many aliases are active
alias | grep git              # find all git-related aliases quickly

# --- Debugging "command not found" for something you SWEAR you aliased ---

type mycommand
# mycommand: command not found     ← confirms it's not defined at all in THIS session
grep -r "alias mycommand" ~/.bashrc ~/.bash_aliases ~/.zshrc 2>/dev/null
# search your config files to see if it was defined but never sourced
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
