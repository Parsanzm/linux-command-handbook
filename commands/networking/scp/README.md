# scp — The Complete Reference

> **Securely copy files between hosts over SSH**
> The quick, no-setup way to move a file to or from a remote machine,
> built directly on the same encrypted connection `ssh` uses.

---

## Table of Contents

- [What is scp?](#what-is-scp)
- [Where does scp live?](#where-does-scp-live)
- [How scp works internally](#how-scp-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Copying To, From, and Between Remote Hosts](#copying-to-from-and-between-remote-hosts)
- [Recursive Copies and Preserving Attributes](#recursive-copies-and-preserving-attributes)
- [All Key Options](#all-key-options)
- [scp's Two Underlying Protocol Modes](#scps-two-underlying-protocol-modes)
- [scp vs rsync vs sftp](#scp-vs-rsync-vs-sftp)
- [Related Commands](#related-commands)

---

## What is scp?

`scp` ("secure copy") copies files and directories between hosts over an SSH connection, encrypting the transfer the same way an interactive `ssh` session is encrypted. It uses `cp`-like syntax extended with a `host:path` notation to indicate a remote location.

```bash
scp report.pdf alice@server.example.com:/home/alice/reports/
# report.pdf                    100%  2048KB   5.2MB/s   00:00
```

**Why scp remains popular despite newer alternatives:** it requires zero setup beyond SSH access that's usually already configured, uses syntax nearly everyone already recognizes from `cp`, and handles the common "just get this one file over there" case with a single short command.

---

## Where does scp live?

```
/usr/bin/scp
```

```bash
which scp
scp -V
# OpenSSH_9.6p1
```

Part of **OpenSSH**, the same package providing `ssh` itself — anywhere `ssh` is installed, `scp` is essentially always available too, since it relies on the exact same client/server infrastructure.

---

## How scp works internally

Historically, `scp` worked by invoking the remote `scp` program over an SSH connection, using a simple, purpose-built wire protocol to transfer file data and metadata (name, size, permissions, timestamps) directly. Modern OpenSSH has **deprecated this legacy protocol** due to some inherent limitations (see [edge-cases.md](edge-cases.md)) in favor of using the SFTP protocol underneath instead, while keeping the same familiar `scp` command-line interface.

```bash
# See which underlying protocol mode a given scp invocation is using
scp -v report.pdf alice@server:/tmp/
# debug1: Sending SSH2_FXP_INIT
# ⚠️ or, on legacy-protocol systems:
# debug1: Executing: program /usr/bin/ssh ... -s scp ...
```

Either way, the actual file transfer rides on top of a normal SSH connection — the same authentication, host-key verification, and encryption used by an interactive `ssh` login apply identically here.

---

## Syntax

```bash
scp [OPTIONS] source target
```

Either `source` or `target` (or both, for host-to-host copies) can be a remote location, specified as:

```
[user@]host:path
```

```bash
scp file.txt alice@server.example.com:/home/alice/
# local file  →  remote destination

scp alice@server.example.com:/home/alice/file.txt ./
# remote file  →  local destination
```

---

## Understanding the Output

```bash
scp bigfile.zip alice@server.example.com:/tmp/
# bigfile.zip                   45%  450MB  12.1MB/s   00:35 ETA
```

| Field | Meaning |
|---|---|
| Filename | The file currently being transferred |
| Percentage | Progress of the current file's transfer |
| Size transferred | Bytes moved so far |
| Rate | Current transfer speed |
| ETA / elapsed time | Estimated time remaining, or elapsed time once complete |

For multi-file/recursive copies, this progress line repeats for each file individually as it's transferred.

---

## Copying To, From, and Between Remote Hosts

```bash
# Local → remote
scp file.txt alice@server.example.com:/home/alice/

# Remote → local
scp alice@server.example.com:/home/alice/file.txt ./

# Remote → remote (copies directly between the two remote hosts)
scp alice@server1.example.com:/data/file.txt bob@server2.example.com:/backup/

# Copy multiple local files to one remote destination
scp file1.txt file2.txt alice@server.example.com:/home/alice/
```

For a remote-to-remote copy, by default the data still flows **through your local machine** unless the `-3` flag or newer direct-transfer behavior is used — see [edge-cases.md](edge-cases.md) for the practical implications of this.

---

## Recursive Copies and Preserving Attributes

```bash
# Copy an entire directory recursively
scp -r ./project/ alice@server.example.com:/home/alice/project/

# Preserve modification times, access times, and file modes
scp -p file.txt alice@server.example.com:/home/alice/

# Combine both — a full, attribute-preserving recursive copy
scp -rp ./project/ alice@server.example.com:/home/alice/project/
```

---

## All Key Options

| Option | Description |
|---|---|
| `-r` | Recursively copy an entire directory |
| `-p` | Preserve modification/access times and file permissions |
| `-P PORT` | Connect to a non-default SSH port (note: capital `P`, unlike `ssh`'s lowercase `-p`) |
| `-i FILE` | Use a specific private key file for authentication |
| `-C` | Compress data during transfer |
| `-l LIMIT` | Limit bandwidth used, in Kbit/s |
| `-v` | Verbose output, useful for troubleshooting |
| `-q` | Quiet mode — suppress the progress meter |
| `-3` | Route a remote-to-remote copy's data through the local machine explicitly (or, on newer versions, control direct vs. local-routed transfer) |
| `-o OPTION=VALUE` | Pass an arbitrary ssh-config-style option directly |

---

## scp's Two Underlying Protocol Modes

```bash
# The legacy "scp/rcp" protocol — simple, but historically had
# notable filename-handling and verification weaknesses (see edge-cases.md)
scp -O file.txt alice@server.example.com:/tmp/
# -O explicitly forces the OLD legacy protocol on newer OpenSSH
# versions where SFTP-based transfer is now the default

# The modern default on current OpenSSH — uses the SFTP protocol
# underneath, addressing those legacy weaknesses
scp file.txt alice@server.example.com:/tmp/
```

Since OpenSSH 9.0, `scp` defaults to using **SFTP** as its underlying transfer mechanism rather than the original legacy protocol, specifically because of security and correctness issues in the old approach — this is a meaningful, version-dependent behavior change worth being aware of.

---

## scp vs rsync vs sftp

| Tool | Best for | Key difference from scp |
|---|---|---|
| `scp` | A quick, one-off copy of a known file or directory | Simple, `cp`-like syntax; always re-transfers the full file, even if it already exists at the destination |
| `rsync` | Repeated syncs, large directory trees, resumable transfers | Only transfers the actual DIFFERENCES between source and destination on repeated runs — dramatically more efficient for anything beyond a single one-time copy |
| `sftp` | Interactive, browsable file transfer sessions | An interactive shell-like session (`ls`, `cd`, `get`, `put`) rather than a single non-interactive command |

```bash
scp file.txt alice@server:/tmp/          # quick one-off copy
rsync -avz ./dir/ alice@server:/backup/   # efficient, resumable sync
sftp alice@server                          # interactive browsing session
```

---

## Related Commands

| Command | Relation |
|---|---|
| `ssh` | The underlying connection scp is built on top of |
| `rsync` | The more efficient choice for repeated syncs or large/resumable transfers |
| `sftp` | Interactive alternative for browsing and transferring files |
| `ssh-keygen` / `ssh-copy-id` | Set up key-based authentication, avoiding repeated password prompts for scp too |
| `tar` | Commonly piped through ssh/scp for transferring many small files as one archive, faster than many individual small transfers |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
