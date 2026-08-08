# ssh — The Complete Reference

> **Securely log into or run commands on a remote machine**
> The standard, encrypted replacement for telnet/rlogin, and the backbone
> of nearly every remote server workflow in existence.

---

## Table of Contents

- [What is ssh?](#what-is-ssh)
- [Where does ssh live?](#where-does-ssh-live)
- [How ssh works internally](#how-ssh-works-internally)
- [Syntax](#syntax)
- [Understanding Host Keys and known_hosts](#understanding-host-keys-and-known_hosts)
- [Authentication Methods](#authentication-methods)
- [SSH Key Pairs — Public and Private Keys](#ssh-key-pairs--public-and-private-keys)
- [All Key Options](#all-key-options)
- [The SSH Config File](#the-ssh-config-file)
- [ssh vs scp vs sftp vs mosh](#ssh-vs-scp-vs-sftp-vs-mosh)
- [Related Commands](#related-commands)

---

## What is ssh?

`ssh` (Secure Shell) opens an encrypted connection to a remote machine, giving you an interactive login shell there, or lets you run a single remote command and return its output — all traffic (including the password or key exchange itself) is encrypted end-to-end.

```bash
ssh alice@server.example.com
# alice@server.example.com's password:
# Welcome to Ubuntu 24.04 LTS ...
# alice@server:~$
```

**Why ssh replaced telnet/rlogin/rsh entirely:** those older tools sent everything — including login credentials — in **plain text** over the network, trivially interceptable by anyone on the same network path. `ssh` encrypts the entire session from the very first handshake onward.

---

## Where does ssh live?

```
/usr/bin/ssh
```

```bash
which ssh
ssh -V
# OpenSSH_9.6p1, OpenSSL 3.0.13
```

Part of **OpenSSH**, the dominant open-source SSH implementation shipped by default on virtually every Linux distribution, macOS, and (via an optional feature) modern Windows. The client (`ssh`) and server (`sshd`) are separate components — having the client installed doesn't mean the machine accepts incoming SSH connections, and vice versa.

---

## How ssh works internally

Connecting involves several distinct phases:

1. **TCP connection** — to the remote host, port 22 by default.
2. **Protocol/version exchange** — client and server announce which SSH protocol version they support.
3. **Key exchange** — client and server negotiate a shared session encryption key using asymmetric cryptography (e.g., Diffie-Hellman), without ever transmitting the key itself over the wire.
4. **Host verification** — the client checks the server's **host key** against its local `known_hosts` file, to confirm it's actually talking to the expected server and not an impostor (see below).
5. **Authentication** — the client proves its identity, typically via a public key or a password (see [Authentication Methods](#authentication-methods)).
6. **Session** — once authenticated, an encrypted channel carries the interactive shell (or a single command's stdin/stdout/stderr) back and forth.

```bash
ssh -v alice@server.example.com
# -v shows this entire handshake process happening step by step —
# useful for diagnosing exactly where a connection is failing
```

---

## Syntax

```bash
ssh [OPTIONS] [user@]hostname [command]
```

If `command` is given, `ssh` runs it remotely and returns its output/exit status instead of opening an interactive shell:

```bash
ssh alice@server.example.com "df -h"
# Runs df -h on the remote machine, prints the result, then disconnects
```

---

## Understanding Host Keys and known_hosts

Every SSH server has its own **host key** — a cryptographic identity that doesn't change between connections (barring a server reinstall or intentional key rotation). The very first time you connect to a new host, `ssh` shows a fingerprint and asks you to confirm it:

```bash
ssh alice@newserver.example.com
# The authenticity of host 'newserver.example.com (203.0.113.5)' can't be established.
# ED25519 key fingerprint is SHA256:AbCdEf1234...
# Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Once accepted, that host key is recorded in `~/.ssh/known_hosts`. On every future connection, `ssh` compares the server's presented key against this stored value:

```bash
# If the server's key has genuinely changed (reinstall, restored from
# backup, DNS/IP reassignment) OR if something is actively intercepting
# the connection, ssh refuses to connect and shows a loud warning:
# WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

This mechanism is what makes SSH resistant to man-in-the-middle interception on subsequent connections — it's the single most important safety check `ssh` performs, and the warning should never be dismissed without understanding *why* the key changed.

---

## Authentication Methods

```bash
# Password authentication — prompts interactively
ssh alice@server.example.com

# Public key authentication — no password prompt if the matching
# private key is available and the public key is authorized remotely
ssh -i ~/.ssh/id_ed25519 alice@server.example.com

# Explicit verbosity to see which authentication method actually succeeded
ssh -v alice@server.example.com 2>&1 | grep -i "authenticat"
```

Public key authentication is generally preferred over passwords: it's not guessable/brute-forceable in the same way, and it enables fully non-interactive/automated connections (scripts, CI pipelines) without embedding a plaintext password anywhere.

---

## SSH Key Pairs — Public and Private Keys

```bash
# Generate a new key pair (Ed25519 is the modern recommended default)
ssh-keygen -t ed25519 -C "alice@laptop"
# Creates:
#   ~/.ssh/id_ed25519       ← PRIVATE key — never share this, ever
#   ~/.ssh/id_ed25519.pub   ← PUBLIC key — safe to share/distribute

# Copy the public key to a remote server's authorized list
ssh-copy-id -i ~/.ssh/id_ed25519.pub alice@server.example.com
# Appends the public key's contents to ~/.ssh/authorized_keys on the
# remote server — after this, ssh to that server no longer prompts
# for a password
```

The private key should be protected with restrictive file permissions (`600`) and, ideally, its own passphrase — `ssh` and most tools refuse to use a private key file with overly permissive permissions as a basic sanity check.

---

## All Key Options

| Option | Description |
|---|---|
| `-p PORT` | Connect to a non-default port instead of 22 |
| `-i FILE` | Use a specific private key file for authentication |
| `-v` / `-vv` / `-vvv` | Verbose (increasingly detailed) diagnostic output |
| `-L LOCAL:HOST:REMOTE` | Local port forwarding (see below) |
| `-R REMOTE:HOST:LOCAL` | Remote port forwarding |
| `-D PORT` | Dynamic port forwarding (SOCKS proxy through the SSH connection) |
| `-N` | Don't execute a remote command — useful with `-L`/`-D` for pure tunneling |
| `-f` | Go to background after authentication (commonly combined with `-N`) |
| `-X` / `-Y` | Enable X11 forwarding (run remote GUI apps, displayed locally) |
| `-A` | Forward the local SSH agent to the remote host |
| `-o OPTION=VALUE` | Set any config-file-style option directly on the command line |
| `-C` | Compress data — occasionally helpful over slow links |
| `-t` | Force pseudo-terminal allocation (needed for some interactive remote commands) |

---

## The SSH Config File

`~/.ssh/config` lets you define per-host shortcuts and defaults, avoiding long repeated command lines:

```
Host myserver
    HostName server.example.com
    User alice
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_work

Host *.internal.example.com
    User admin
    ProxyJump bastion.example.com
```

```bash
# With the config above, this:
ssh myserver
# is exactly equivalent to:
ssh -p 2222 -i ~/.ssh/id_ed25519_work alice@server.example.com
```

`ProxyJump` (or the older `-J` flag / `ProxyCommand`) is the standard way to reach a host only accessible through an intermediate bastion/jump server, without manually chaining two separate `ssh` commands.

---

## ssh vs scp vs sftp vs mosh

| Tool | Best for | Key difference from ssh |
|---|---|---|
| `ssh` | Interactive shell access, running remote commands | The base protocol and connection; everything else here builds on it |
| `scp` | Copying files to/from a remote host | A file-transfer tool that uses the SSH connection/protocol underneath |
| `sftp` | Interactive, browsable file transfer session | Also SSH-based; provides an interactive `ls`/`get`/`put`-style session rather than a one-shot copy |
| `rsync -e ssh` | Efficient, resumable file synchronization | Uses SSH as its transport, but only transfers the actual differences between files |
| `mosh` | Unstable/high-latency connections (mobile, spotty Wi-Fi) | Built on SSH for initial authentication, then uses its own UDP-based protocol that survives network changes/roaming better than a raw SSH session |

```bash
ssh alice@server "ls /var/log"     # run one remote command
scp file.txt alice@server:/tmp/     # copy a file over
sftp alice@server                    # interactive file browsing session
rsync -avz -e ssh ./dir/ alice@server:/backup/   # efficient sync
```

---

## Related Commands

| Command | Relation |
|---|---|
| `sshd` | The server-side daemon that accepts incoming `ssh` connections |
| `ssh-keygen` | Generate SSH key pairs |
| `ssh-copy-id` | Install a public key onto a remote server's authorized list |
| `ssh-agent` | Cache a decrypted private key in memory so passphrases aren't re-entered constantly |
| `scp` / `sftp` / `rsync` | File-transfer tools built on top of the SSH protocol |
| `known_hosts` / `authorized_keys` | The two key files underpinning host verification and public-key auth, respectively |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
