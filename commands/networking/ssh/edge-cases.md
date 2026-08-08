# ssh — Edge Cases & Gotchas

> `ssh` looks like a simple remote login tool, but host-key changes,
> permission checks, agent forwarding, and quoting rules routinely
> trip people up — often at the worst possible moment.

---

## Table of Contents

- ["REMOTE HOST IDENTIFICATION HAS CHANGED" Should Never Be Dismissed Casually](#remote-host-identification-has-changed-should-never-be-dismissed-casually)
- [Overly Permissive Private Key Permissions Cause a Silent (or Loud) Rejection](#overly-permissive-private-key-permissions-cause-a-silent-or-loud-rejection)
- [Multiple Keys Available Doesn't Mean the Right One Gets Offered First](#multiple-keys-available-doesnt-mean-the-right-one-gets-offered-first)
- [Quoting a Remote Command Wrong Changes Where Variables Get Expanded](#quoting-a-remote-command-wrong-changes-where-variables-get-expanded)
- [SSH Sessions Don't Survive a Closed Laptop Lid or Network Drop by Default](#ssh-sessions-dont-survive-a-closed-laptop-lid-or-network-drop-by-default)
- [Background Processes Started Over SSH Can Die When the Session Ends](#background-processes-started-over-ssh-can-die-when-the-session-ends)
- [Agent Forwarding (-A) Has Real Security Implications on a Compromised Host](#agent-forwarding--a-has-real-security-implications-on-a-compromised-host)
- [A Full known_hosts Line Can Reference an Outdated IP, Not Just a Hostname](#a-full-known_hosts-line-can-reference-an-outdated-ip-not-just-a-hostname)
- [BatchMode Prevents Password Prompts — Useful in Scripts, Confusing Otherwise](#batchmode-prevents-password-prompts--useful-in-scripts-confusing-otherwise)
- [Exit Status From a Remote Command Reflects the REMOTE Command, Not ssh Itself (Usually)](#exit-status-from-a-remote-command-reflects-the-remote-command-not-ssh-itself-usually)
- [Copy-Pasting a Multi-Line Command Can Behave Differently Than Typing It](#copy-pasting-a-multi-line-command-can-behave-differently-than-typing-it)

---

## "REMOTE HOST IDENTIFICATION HAS CHANGED" Should Never Be Dismissed Casually

### The single most important security warning ssh can show
```bash
ssh alice@server.example.com
# @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
# @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
# @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
# ⚠️ This means the host key the server is presenting RIGHT NOW does
# NOT match the one recorded from a previous connection — this can be
# a completely legitimate reason (the server was reinstalled, its SSH
# host keys were intentionally rotated, or its IP got reassigned to a
# different machine) OR it can mean an active man-in-the-middle
# attack is intercepting the connection and presenting a different
# key to impersonate the real server.

# NEVER blindly delete the offending known_hosts line just to make the
# warning go away without first VERIFYING the change is expected —
# ideally by confirming the new fingerprint out-of-band (calling the
# server admin, checking a change log) rather than assuming benign intent:
ssh-keygen -R server.example.com   # only AFTER verifying legitimacy
```

---

## Overly Permissive Private Key Permissions Cause a Silent (or Loud) Rejection

### ssh refuses to use a key file that looks insecurely shared
```bash
chmod 644 ~/.ssh/id_ed25519       # readable/writable by group/others too
ssh -i ~/.ssh/id_ed25519 alice@server.example.com
# Permissions 0644 for '~/.ssh/id_ed25519' are too open.
# It is required that your private key files are NOT accessible by others.
# ⚠️ ssh REFUSES to use a private key file with group/world-readable
# permissions, as a basic sanity check against accidentally shared
# key material — this isn't a bug or overly cautious default; it's
# intentional and should be fixed by restricting permissions, not
# worked around.

chmod 600 ~/.ssh/id_ed25519       # readable/writable ONLY by the owner
```

---

## Multiple Keys Available Doesn't Mean the Right One Gets Offered First

### ssh tries keys in a specific order, which can cause unexpected auth failures
```bash
ls ~/.ssh/
# id_ed25519  id_ed25519.pub  id_rsa_old  id_rsa_old.pub  id_ecdsa_work.pub

ssh alice@server.example.com
# Permission denied (publickey).
# ⚠️ With multiple key files present, ssh tries its DEFAULT identity
# files in a built-in order (typically id_rsa, id_ecdsa, id_ed25519,
# etc.) — if the server only accepts a SPECIFIC key that isn't among
# ssh's default set, or if too many failed attempts trip the server's
# MaxAuthTries limit before the correct key is even offered, the
# connection fails even though the right key genuinely exists locally.

# Force ssh to use ONLY the specific key you intend, skipping its
# default search order entirely:
ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes alice@server.example.com
```

---

## Quoting a Remote Command Wrong Changes Where Variables Get Expanded

### A classic source of "why did my remote command use the WRONG value" confusion
```bash
ssh alice@server "echo $HOME"
# ⚠️ Because of the DOUBLE quotes, $HOME is expanded by the LOCAL
# shell BEFORE the command is even sent to the remote host — this
# prints YOUR local machine's $HOME value, not the remote server's,
# even though it visually looks like it's running "on" the remote side.

ssh alice@server 'echo $HOME'
# Using SINGLE quotes instead prevents local expansion — $HOME is
# now expanded by the REMOTE shell once the command arrives there,
# correctly showing the remote user's home directory.

# Mixed cases (some variables meant to expand locally, others
# remotely, in the SAME command) require careful, deliberate escaping
# — a common source of subtle scripting bugs when this isn't intentional.
```

---

## SSH Sessions Don't Survive a Closed Laptop Lid or Network Drop by Default

### A long-running remote task can be silently killed by an interrupted connection
```bash
ssh alice@server
$ ./long_running_build.sh
# ... laptop goes to sleep, WiFi drops, or a VPN reconnects ...
# ⚠️ The SSH session (and, typically, whatever it was running
# interactively in the foreground) is ABRUPTLY terminated the moment
# the underlying TCP connection breaks — there's no built-in
# reconnection or session persistence in plain ssh itself.

# The standard mitigation: run long tasks inside a terminal
# multiplexer on the REMOTE side, which keeps running independently
# of your ssh connection's lifetime:
ssh alice@server
$ tmux new -s build
$ ./long_running_build.sh
# Ctrl+B, D to detach — the session (and the task) keeps running even
# after you disconnect; reconnect later with:
$ tmux attach -t build
```

---

## Background Processes Started Over SSH Can Die When the Session Ends

### & alone is often NOT enough to make a remote process survive disconnection
```bash
ssh alice@server "./long_task.sh &"
# ⚠️ Even with a trailing &, this process can still receive SIGHUP
# and terminate when the ssh session itself closes, depending on the
# shell and exact invocation — backgrounding within the SAME ssh
# command doesn't reliably protect against session-close signals the
# way it might for a purely local background job.

# More reliable alternatives, in increasing order of robustness:
ssh alice@server "nohup ./long_task.sh > /tmp/out.log 2>&1 &"
# nohup explicitly makes the process ignore SIGHUP

ssh alice@server "setsid ./long_task.sh > /tmp/out.log 2>&1 < /dev/null &"
# setsid fully detaches it from the controlling terminal/session

ssh alice@server "tmux new-session -d -s work './long_task.sh'"
# tmux/screen are the most robust — the process lives inside its OWN
# independent session entirely, unaffected by ssh disconnecting
```

---

## Agent Forwarding (-A) Has Real Security Implications on a Compromised Host

### A convenience feature that expands the blast radius of a compromised intermediate host
```bash
ssh -A alice@bastion.example.com
# From here, hop further using the SAME forwarded key:
ssh internal-host.local
# ⚠️ Agent forwarding lets the REMOTE host (bastion, in this example)
# make authentication requests THROUGH your local ssh-agent, without
# ever seeing your actual private key material directly. However, this
# means that if the bastion host is compromised WHILE you're
# connected with agent forwarding active, an attacker on that host
# can use your forwarded agent to authenticate AS YOU to any other
# server your key has access to — for the duration of that active
# connection.

# Prefer ProxyJump (-J) over agent forwarding for reaching an internal
# host through a bastion when possible — it doesn't expose the
# forwarded agent to the intermediate host at all:
ssh -J bastion.example.com alice@internal-host.local
```

---

## A Full known_hosts Line Can Reference an Outdated IP, Not Just a Hostname

### Both hostname AND IP-based entries can exist and go stale independently
```bash
cat ~/.ssh/known_hosts | grep server.example.com
# server.example.com,203.0.113.5 ssh-ed25519 AAAA...
# ⚠️ known_hosts entries can be keyed by hostname, by IP, or both
# together on the same line — if a server's IP address changes (DNS
# update, cloud instance replacement) while the hostname is reused,
# you may hit a host-key mismatch warning tied to the STALE IP
# portion specifically, even though the hostname itself is genuinely
# still correct and expected.

# Remove just the specific stale entry rather than the whole file:
ssh-keygen -R server.example.com
ssh-keygen -R 203.0.113.5
```

---

## BatchMode Prevents Password Prompts — Useful in Scripts, Confusing Otherwise

### An automated script hanging forever, or failing instead of prompting
```bash
ssh -o BatchMode=yes alice@server.example.com
# Permission denied (publickey).
# ⚠️ With BatchMode=yes, ssh will NEVER fall back to an interactive
# password/passphrase prompt — if key-based auth fails for any
# reason, it fails immediately and cleanly instead of hanging while
# waiting for input that a script (running non-interactively) could
# never actually provide. This is exactly the desired behavior for
# automation, but confusing if encountered unexpectedly during manual
# testing, where a normal interactive ssh session WOULD have simply
# prompted for a password instead of failing outright.
```

---

## Exit Status From a Remote Command Reflects the REMOTE Command, Not ssh Itself (Usually)

### A script checking $? after ssh needs to know which failure it's actually seeing
```bash
ssh alice@server "exit 42"
echo $?
# 42     ← ssh propagates the REMOTE command's own exit status back
#           as its own exit status, when the connection and
#           authentication themselves succeeded

ssh alice@nonexistent-host "exit 42"
echo $?
# 255    ← but a CONNECTION-level failure (unreachable host, auth
#           failure, etc.) uses ssh's OWN generic error code (255),
#           which can be confused with a genuine remote exit code of
#           255 from a command that actually ran successfully

# Distinguishing "the connection failed" from "the remote command
# itself returned 255" isn't always possible from exit status alone —
# checking stderr output or using more granular remote error handling
# is more reliable when this distinction genuinely matters.
```

---

## Copy-Pasting a Multi-Line Command Can Behave Differently Than Typing It

### Terminal paste behavior, not ssh itself, but a very common ssh-session gotcha
```bash
# Pasting several lines of a script directly into an active ssh
# session can behave unexpectedly if the remote shell has features
# like bracketed paste mode disabled, autocomplete/history expansion
# interfering mid-paste, or if the pasted content contains characters
# the remote shell interprets differently than intended mid-stream.
# ⚠️ This is especially risky when pasting content containing
# sensitive commands or credentials — a malformed paste can execute
# PARTIAL lines unexpectedly before the full intended command arrives.

# Safer for anything nontrivial: transfer the script as a FILE and
# run it, rather than relying on interactive paste behavior:
scp script.sh alice@server:/tmp/
ssh alice@server "bash /tmp/script.sh"
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
