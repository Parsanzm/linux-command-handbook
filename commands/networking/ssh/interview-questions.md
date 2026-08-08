# ssh — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Authentication](#authentication)
- [Host Keys and Security](#host-keys-and-security)
- [Internals](#internals)
- [Port Forwarding](#port-forwarding)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does ssh do, and why did it replace tools like telnet?**
> `ssh` opens an encrypted connection to a remote machine for an interactive shell or a single remote command. It replaced telnet/rlogin because those older tools transmitted everything, including login credentials, in plain text — trivially interceptable on the network — while ssh encrypts the entire session from the initial handshake.

---

**Q2 🔥 What's the difference between the ssh client and sshd?**
> `ssh` is the client program used to initiate an outgoing connection. `sshd` is the server-side daemon that listens for and accepts incoming connections. A machine can have either, both, or neither installed independently.

---

**Q3. How do you run a single remote command without opening an interactive shell?**
> `ssh user@host "command"` — this runs the given command on the remote host, streams its output back, and disconnects, rather than dropping into an interactive session.

---

## Authentication

**Q4 🔥 What's the difference between password authentication and public key authentication in ssh?**
> Password authentication prompts for a secret shared with the server directly. Public key authentication proves identity using a private key that never leaves the client, matched against a public key already authorized on the server — generally preferred since it's not brute-forceable in the same way and enables non-interactive/automated connections.

---

**Q5. What are the two files created by ssh-keygen, and which one is safe to share?**
> A private key file (e.g., `id_ed25519`) and a public key file (`id_ed25519.pub`). Only the public key is safe to share/distribute; the private key must never be shared and should be kept with restrictive file permissions.

---

**Q6 🔥 What does ssh-copy-id actually do?**
> It appends the contents of a local public key file to the remote server's `~/.ssh/authorized_keys` file, enabling passwordless (public key) login for that key going forward.

---

## Host Keys and Security

**Q7 🔥 What is a host key, and what problem does it solve?**
> A cryptographic identity unique to a given SSH server that stays consistent across connections. It lets the client verify it's genuinely talking to the expected server (rather than an impostor intercepting the connection) by comparing the presented key against a previously recorded value.

---

**Q8. Where does ssh store host keys it has previously seen and accepted?**
> `~/.ssh/known_hosts` — on every subsequent connection, ssh compares the server's presented host key against the entry recorded there.

---

**Q9 🔥 What does the "REMOTE HOST IDENTIFICATION HAS CHANGED" warning mean, and should it ever be dismissed automatically?**
> It means the host key the server just presented doesn't match the one previously recorded for that host — which can be a legitimate reason (server reinstall, intentional key rotation, IP reassignment) or an active man-in-the-middle attack. It should never be dismissed without first verifying the change is actually expected, ideally through an out-of-band confirmation.

---

## Internals

**Q10. What are the broad phases of an ssh connection, from start to finish?**
> TCP connection, protocol/version exchange, key exchange (negotiating a shared session encryption key), host verification (checking the server's host key against known_hosts), authentication (proving the client's identity), and finally the encrypted session itself.

---

**Q11 🔥 Why does ssh refuse to use a private key file with overly permissive file permissions?**
> As a basic sanity check against accidentally shared or exposed key material — a private key readable by other users on the same system defeats much of the purpose of key-based authentication, so ssh intentionally refuses to use such a file until its permissions are restricted (typically to `600`).

---

**Q12. What's the purpose of ssh-agent?**
> It caches a decrypted private key in memory for the duration of a session, so a passphrase-protected key doesn't need to be re-entered for every single ssh connection — subsequent connections use the already-unlocked key held by the running agent.

---

## Port Forwarding

**Q13 🔥 What's the difference between local (-L) and remote (-R) port forwarding?**
> Local forwarding (`-L`) makes a port on the remote side reachable through a local port on your own machine — useful for accessing a service only reachable from inside the remote network. Remote forwarding (`-R`) does the reverse: it exposes a port on your local machine to the remote side, reachable from that remote host.

---

**Q14. What does -D (dynamic port forwarding) do?**
> Turns the ssh connection into a SOCKS proxy — traffic sent to the specified local port is routed dynamically through the encrypted ssh connection to whatever destination the connecting application requests, rather than forwarding to one single fixed remote address/port like `-L`/`-R` do.

---

## Scenario-Based

**Q15 🔥 A teammate has several SSH key files in `~/.ssh/` but keeps getting "Permission denied (publickey)" against a server they know accepts one of their keys. What's a likely cause, and how would you fix it?**
> ssh tries its default identity files in a built-in order, and if the server only accepts a specific key that isn't offered early enough (or the server's max auth attempts are exhausted first), authentication fails even though the correct key exists locally. Forcing ssh to use only the intended key explicitly — `ssh -i ~/.ssh/the_right_key -o IdentitiesOnly=yes user@host` — resolves this by skipping the default search order entirely.

---

**Q16. Someone runs `ssh server "echo $HOME"` expecting to see the remote user's home directory, but instead sees their own local home directory path. What happened?**
> Double quotes let the local shell expand `$HOME` before the command is even sent to the remote host, so the value substituted is the local machine's, not the remote server's. Using single quotes instead (`ssh server 'echo $HOME'`) prevents local expansion, letting the remote shell expand the variable once the command arrives there.

---

**Q17 🔥 A long-running script started interactively over ssh dies the moment the connection drops (laptop sleep, WiFi hiccup). Why, and what's the standard fix?**
> Plain interactive ssh sessions don't survive a broken underlying TCP connection — whatever was running in the foreground terminates along with the session. The standard mitigation is running the task inside a terminal multiplexer (`tmux`/`screen`) on the remote side, which keeps running independently of the ssh connection's own lifetime, allowing reconnection and reattachment later.

---

**Q18. What's a security concern with using ssh agent forwarding (-A) to reach a further internal host through a bastion server?**
> If the bastion host is compromised while agent forwarding is active, an attacker on that host can use the forwarded agent to authenticate as the connecting user against any other server the key has access to, for the duration of that active connection — since the bastion can make authentication requests through the forwarded agent. `ProxyJump` (`-J`) is generally preferred for reaching an internal host through a bastion, since it doesn't expose the forwarded agent to the intermediate host at all.

---

**Q19 🔥 A background remote task started with `ssh server "./task.sh &"` unexpectedly stops when the ssh session ends, even though it was backgrounded. Why doesn't `&` alone protect it?**
> The backgrounded process can still receive `SIGHUP` when the ssh session itself closes, depending on the shell and exact invocation — simple backgrounding within the same ssh command doesn't reliably protect against session-close signals. Wrapping the command with `nohup` (to ignore `SIGHUP`), `setsid` (to fully detach from the session), or running it inside `tmux`/`screen` are the standard, more robust fixes.

---

**Q20. In an automation script, why might someone deliberately add `-o BatchMode=yes` to an ssh invocation?**
> It prevents ssh from ever falling back to an interactive password/passphrase prompt — if key-based authentication fails, the connection fails immediately and cleanly instead of hanging indefinitely waiting for input a non-interactive script could never provide, which is the desired, predictable failure behavior for automated pipelines.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
