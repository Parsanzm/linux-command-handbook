# scp — Edge Cases & Gotchas

> `scp` looks like a simple `cp`-with-a-hostname tool, but wildcard
> expansion, remote-to-remote data routing, and its recent protocol
> switch all produce results that differ from what most people expect.

---

## Table of Contents

- [Uppercase -P for Port, Lowercase -p for Preserve — Easy to Mix Up](#uppercase--p-for-port-lowercase--p-for-preserve--easy-to-mix-up)
- [scp Always Re-Transfers the Whole File, Even If It Barely Changed](#scp-always-re-transfers-the-whole-file-even-if-it-barely-changed)
- [Remote-to-Remote Copies Historically Route Through Your Local Machine](#remote-to-remote-copies-historically-route-through-your-local-machine)
- [Wildcards in a Remote Path Are Expanded by the REMOTE Shell, Not Locally](#wildcards-in-a-remote-path-are-expanded-by-the-remote-shell-not-locally)
- [Copying a Directory Without -r Fails With an Unhelpful-Looking Error](#copying-a-directory-without--r-fails-with-an-unhelpful-looking-error)
- [Modern OpenSSH Switched scp's Default Protocol — Some Old Behaviors No Longer Apply](#modern-openssh-switched-scps-default-protocol--some-old-behaviors-no-longer-apply)
- [A Trailing Slash on the Destination Directory Matters](#a-trailing-slash-on-the-destination-directory-matters)
- [scp Silently Overwrites an Existing Destination File With No Confirmation](#scp-silently-overwrites-an-existing-destination-file-with-no-confirmation)
- [Copying Many Small Files Is Much Slower Than One Archive of the Same Total Size](#copying-many-small-files-is-much-slower-than-one-archive-of-the-same-total-size)
- [A Colon in a Local Filename Can Be Misread as a Remote Host Specifier](#a-colon-in-a-local-filename-can-be-misread-as-a-remote-host-specifier)
- [Interrupted Transfers Leave a Partial File Behind, Not Automatically Cleaned Up](#interrupted-transfers-leave-a-partial-file-behind-not-automatically-cleaned-up)

---

## Uppercase -P for Port, Lowercase -p for Preserve — Easy to Mix Up

### A single-case difference between scp and its sibling command ssh
```bash
scp -p 2222 file.txt alice@server.example.com:/tmp/
# ⚠️ Lowercase -p in scp means "PRESERVE file attributes" (times,
# permissions) — NOT "connect on this PORT" the way ssh's lowercase
# -p does. Using lowercase -p here doesn't set the port at all; it's
# silently interpreted as trying to preserve attributes while "2222"
# gets treated as part of the file list, producing a confusing error
# or unintended behavior rather than connecting to the intended port.

scp -P 2222 file.txt alice@server.example.com:/tmp/
# UPPERCASE -P is scp's actual port flag — the inconsistency with
# ssh's lowercase -p is a well-known, frequently-tripped-over quirk
```

---

## scp Always Re-Transfers the Whole File, Even If It Barely Changed

### No incremental/differential transfer capability at all
```bash
scp large_config.yaml alice@server.example.com:/etc/myapp/
# ... you fix a single typo locally, then run the exact same command again ...
scp large_config.yaml alice@server.example.com:/etc/myapp/
# ⚠️ scp has NO concept of transferring only the CHANGED portions of
# a file — every single invocation re-sends the ENTIRE file from
# scratch, regardless of how similar it is to what's already at the
# destination. For a huge file changed only slightly, this wastes
# significant bandwidth and time compared to a tool that can diff.

# rsync is specifically designed to solve exactly this:
rsync -avz large_config.yaml alice@server.example.com:/etc/myapp/
# only transfers the actual DIFFERENCES on subsequent runs
```

---

## Remote-to-Remote Copies Historically Route Through Your Local Machine

### A copy that "looks" direct can actually double the network hops involved
```bash
scp alice@server1.example.com:/data/huge_file.tar bob@server2.example.com:/backup/
# ⚠️ Historically (and still by default on many systems/versions),
# a remote-to-remote scp copy actually flows: server1 → YOUR
# LOCAL MACHINE → server2 — NOT directly between the two remote
# hosts. This means your local machine's own upload/download
# bandwidth becomes the bottleneck for the ENTIRE transfer, even
# though neither endpoint is your own computer, and a slow local
# connection can make this dramatically slower than expected.

# Some newer OpenSSH versions/configurations support a genuinely
# direct server-to-server transfer instead — verify actual behavior
# for your specific scp version, or use ssh directly for a true
# server-to-server pipe:
ssh alice@server1.example.com "cat /data/huge_file.tar" | \
  ssh bob@server2.example.com "cat > /backup/huge_file.tar"
```

---

## Wildcards in a Remote Path Are Expanded by the REMOTE Shell, Not Locally

### Quoting a remote glob pattern correctly requires care
```bash
scp alice@server.example.com:/var/log/*.log ./
# ⚠️ Depending on quoting, this glob pattern (*.log) can be expanded
# by your LOCAL shell BEFORE scp ever sends the command remotely —
# which usually fails, since no matching local files exist — rather
# than being sent as a literal pattern for the REMOTE shell to expand
# against files that actually exist over there.

# Quote the remote path so the wildcard survives to be expanded
REMOTELY, on the far end, against the actual files that exist there:
scp "alice@server.example.com:/var/log/*.log" ./
```

---

## Copying a Directory Without -r Fails With an Unhelpful-Looking Error

### A very common first mistake
```bash
scp ./project/ alice@server.example.com:/home/alice/
# scp: project: not a regular file
# ⚠️ Without -r, scp refuses to copy a directory at all — the error
# message doesn't always make the missing flag obviously clear to
# someone unfamiliar with this specific requirement.

scp -r ./project/ alice@server.example.com:/home/alice/
# -r enables recursive copying of the directory and everything inside it
```

---

## Modern OpenSSH Switched scp's Default Protocol — Some Old Behaviors No Longer Apply

### A version-dependent change worth knowing about, especially for older tutorials
```bash
# Since OpenSSH 9.0, scp defaults to using SFTP as its underlying
# transfer mechanism instead of the original legacy scp/rcp protocol,
# specifically because the old protocol had known filename-handling
# and verification weaknesses.

# ⚠️ Tutorials, scripts, or habits built around the OLD protocol's
# specific quirks/behaviors may not transfer cleanly to a system
# running a newer OpenSSH version with the new default — and some
# advanced remote-path wildcard/expansion edge cases can behave
# slightly differently between the two underlying protocol modes.

# Force the legacy protocol explicitly if a specific old behavior is
# genuinely required for compatibility with something depending on it:
scp -O file.txt alice@server.example.com:/tmp/
```

---

## A Trailing Slash on the Destination Directory Matters

### Similar in spirit to rsync's well-known trailing-slash sensitivity, though less severe
```bash
scp -r ./project alice@server.example.com:/home/alice/existing-dir
# Copies "project" itself INTO existing-dir, resulting in:
# /home/alice/existing-dir/project/...

scp -r ./project alice@server.example.com:/home/alice/existing-dir/
# ⚠️ Generally behaves the SAME as above in most scp implementations
# (unlike rsync, where the trailing slash on the SOURCE changes
# behavior significantly) — but destination behavior can still vary
# subtly by exact scp/SFTP version when the destination doesn't
# already exist as a directory. Always verify the actual resulting
# path structure after a recursive copy rather than assuming.
```

---

## scp Silently Overwrites an Existing Destination File With No Confirmation

### No interactive prompt, unlike some cp implementations with -i
```bash
scp updated_config.yaml alice@server.example.com:/etc/myapp/config.yaml
# ⚠️ If /etc/myapp/config.yaml already exists at the destination, scp
# overwrites it IMMEDIATELY and SILENTLY — no confirmation prompt, no
# backup of the previous version, nothing. A single mistaken
# invocation can permanently destroy a remote file's previous content
# with no built-in safety net or undo.

# Back up the existing remote file first if there's any doubt:
ssh alice@server.example.com "cp /etc/myapp/config.yaml /etc/myapp/config.yaml.bak"
scp updated_config.yaml alice@server.example.com:/etc/myapp/config.yaml
```

---

## Copying Many Small Files Is Much Slower Than One Archive of the Same Total Size

### Per-file overhead adds up quickly with scp's file-at-a-time transfer model
```bash
scp -r ./node_modules/ alice@server.example.com:/tmp/
# ⚠️ A directory with thousands of small files can take DRAMATICALLY
# longer to copy via scp than a single archive containing the exact
# same total amount of data — each individual file transfer carries
# its own per-file protocol overhead (metadata exchange, handshake
# steps), which adds up significantly at scale, independent of the
# actual total byte count being moved.

# Archive first, transfer the single resulting file, then extract remotely:
tar czf archive.tar.gz ./node_modules/
scp archive.tar.gz alice@server.example.com:/tmp/
ssh alice@server.example.com "cd /tmp && tar xzf archive.tar.gz"
```

---

## A Colon in a Local Filename Can Be Misread as a Remote Host Specifier

### An unusual but real source of confusion with certain filenames
```bash
scp "meeting-notes:2026-08-11.txt" alice@server.example.com:/tmp/
# ⚠️ scp determines whether an argument refers to a REMOTE location
# by checking for a colon (host:path syntax) — a purely LOCAL
# filename that happens to contain a colon character (unusual, but
# possible, especially on filesystems/tools that allow it) can be
# misinterpreted as an attempt to specify a remote host, producing a
# confusing "unknown host" style error instead of copying the
# intended local file.

# A leading "./" disambiguates a local path unambiguously in this situation:
scp "./meeting-notes:2026-08-11.txt" alice@server.example.com:/tmp/
```

---

## Interrupted Transfers Leave a Partial File Behind, Not Automatically Cleaned Up

### An incomplete copy can look deceptively complete at a glance
```bash
scp largefile.iso alice@server.example.com:/tmp/
# ... connection drops partway through ...
# ⚠️ scp has no automatic resume capability at all (unlike rsync or
# wget -c) — an interrupted transfer simply leaves whatever partial
# data had already arrived sitting at the destination path, with NO
# automatic cleanup and no clear built-in signal distinguishing a
# genuinely complete file from a truncated one just by its presence.

# Verify file size or checksum explicitly after any transfer where
# interruption is a real possibility, rather than assuming presence
# alone means success:
ssh alice@server.example.com "ls -la /tmp/largefile.iso"
# compare against the expected local file size
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
