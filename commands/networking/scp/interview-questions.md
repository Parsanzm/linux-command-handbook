# scp — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Syntax and Usage](#syntax-and-usage)
- [Internals](#internals)
- [scp vs Other Tools](#scp-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does scp do?**
> Copies files and directories between hosts over an encrypted SSH connection, using `cp`-like syntax extended with a `host:path` notation to indicate a remote location.

---

**Q2 🔥 What underlying connection/infrastructure does scp rely on?**
> The same SSH infrastructure used by the `ssh` command itself — the same authentication, host-key verification, and encryption apply identically, since scp is part of the OpenSSH package and built directly on top of it.

---

**Q3. How do you specify a remote location in an scp command?**
> `[user@]host:path` — e.g., `alice@server.example.com:/home/alice/file.txt`. Either the source, the destination, or both (for a remote-to-remote copy) can use this notation.

---

## Syntax and Usage

**Q4 🔥 What flag is required to copy an entire directory with scp, and what happens without it?**
> `-r` (recursive). Without it, scp refuses to copy a directory at all, returning an error rather than copying just the top-level entry.

---

**Q5. What's the difference between scp's -p and -P flags?**
> Lowercase `-p` preserves file attributes (modification/access times, permissions). Uppercase `-P` specifies a non-default SSH port to connect on. This is the reverse of ssh's own convention, where lowercase `-p` sets the port — a well-known, easy-to-mix-up inconsistency between the two commands.

---

**Q6 🔥 Does scp overwrite an existing file at the destination without warning?**
> Yes — scp overwrites an existing destination file immediately and silently, with no confirmation prompt and no automatic backup of the previous content.

---

## Internals

**Q7. What underlying protocol does modern scp use by default, and how does that differ from older versions?**
> Since OpenSSH 9.0, scp defaults to using the SFTP protocol underneath rather than the original legacy scp/rcp protocol, specifically because the old protocol had known filename-handling and verification weaknesses. The legacy protocol can still be explicitly forced with `-O` if needed for compatibility.

---

**Q8 🔥 Does scp support resuming an interrupted transfer?**
> No — scp has no built-in resume capability. An interrupted transfer leaves whatever partial data had already arrived at the destination, with no automatic cleanup and no clear signal distinguishing a complete file from a truncated one without an explicit size/checksum check.

---

**Q9. Where does data actually flow during a remote-to-remote scp copy (copying directly between two remote hosts)?**
> Historically (and still by default on many systems), the data flows through the local machine initiating the command — server1 → local machine → server2 — rather than directly between the two remote hosts, meaning the local connection's bandwidth becomes the bottleneck for the entire transfer.

---

## scp vs Other Tools

**Q10 🔥 What's the key limitation of scp compared to rsync for repeated transfers of the same file?**
> scp has no concept of incremental/differential transfer — every invocation re-sends the entire file from scratch regardless of how similar it is to what's already at the destination. rsync only transfers the actual differences on subsequent runs, making it dramatically more efficient for repeated syncs.

---

**Q11. When would you use sftp instead of scp?**
> When an interactive, browsable file transfer session is wanted (navigating directories, listing contents, transferring several files selectively as you go) rather than a single, non-interactive one-shot copy command.

---

## Scenario-Based

**Q12 🔥 A teammate tries `scp -p 2222 file.txt alice@server:/tmp/` expecting to connect on port 2222, but it doesn't work as intended. What's the mistake?**
> Lowercase `-p` in scp means "preserve file attributes," not "specify port" — unlike ssh, where lowercase `-p` does set the port. The correct flag for a custom port in scp is uppercase `-P` (`scp -P 2222 ...`).

---

**Q13. A large configuration file is copied via scp, a single line is changed locally, and then the exact same scp command is run again. Why does this feel unexpectedly slow for such a small change, and what would be a more efficient tool for this workflow?**
> scp always re-transfers the entire file on every invocation, with no ability to send only the changed portion — even a one-line change results in a full re-upload of the whole file. `rsync` is purpose-built to solve exactly this, transferring only the actual differences between the local and remote versions on repeated runs.

---

**Q14 🔥 Someone runs `scp alice@server.example.com:/var/log/*.log ./` and gets an error suggesting no matching files were found locally, even though matching files clearly exist on the remote server. What's happening?**
> The wildcard is likely being expanded by the local shell before scp even runs, rather than being passed through as a literal pattern for the remote shell to expand against files that actually exist there. Quoting the remote path (`scp "alice@server.example.com:/var/log/*.log" ./`) prevents local expansion and lets the wildcard be evaluated remotely instead.

---

**Q15. A directory containing several thousand small files takes far longer to copy via scp than a single large file of the same total byte size. Why, and what's a common workaround?**
> Each individual file transfer via scp carries its own per-file protocol overhead (metadata exchange, handshake steps), which adds up significantly at scale, independent of the total data volume being moved. A common workaround is archiving the directory into a single file first (`tar czf`), transferring that one archive, and extracting it remotely — avoiding thousands of small individual transfers.

---

**Q16 🔥 A network interruption occurs midway through a large scp transfer. What state does the destination file end up in, and how would you verify whether the transfer actually completed successfully?**
> The destination is left with whatever partial data had already arrived — scp has no automatic resume or cleanup mechanism, so a truncated file can sit at the expected path looking deceptively like a completed transfer. Verifying the file's size (or, better, a checksum) against the known-good source value is the reliable way to confirm the transfer actually completed in full, rather than assuming file presence alone means success.

---

**Q17. Why might copying directly between two remote servers with scp end up being slower than expected, even though your own machine isn't one of the two endpoints?**
> Because the data historically (and often still by default) routes through the local machine initiating the command, rather than flowing directly between the two remote hosts — meaning the local connection's own upload/download bandwidth becomes the limiting factor for the entire transfer, despite appearing to be a "remote-to-remote" copy on the surface.

---

**Q18 🔥 What's a security-relevant reason someone might deliberately want to run `scp -O ...` instead of scp's current default behavior?**
> To explicitly force the older legacy scp/rcp protocol, typically for compatibility with something specifically depending on its older behavior — though this should be a deliberate, informed choice rather than a default habit, since the legacy protocol is exactly what modern OpenSSH moved away from due to its known filename-handling and verification weaknesses.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
