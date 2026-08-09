# netstat — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Reading the Output](#reading-the-output)
- [Connection States](#connection-states)
- [Internals](#internals)
- [netstat vs Other Tools](#netstat-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does netstat do?**
> Prints information about the machine's network subsystem — active TCP/UDP connections, listening ports, the kernel routing table, and per-interface traffic statistics — all accessible through a single, long-standing command.

---

**Q2 🔥 Is netstat still actively developed?**
> No — it's part of the `net-tools` package, which has been deprecated in favor of `iproute2` (the `ss` and `ip` commands) for years and is now effectively in a maintenance-only state. It still works and remains widely used and documented, but doesn't receive new functionality.

---

**Q3. What does plain `netstat` with no options show by default?**
> Active (established) connections — not listening ports. This surprises many first-time users who expect to see listening services by default and need `-l` (or a combination like `-tuln`) explicitly to see those instead.

---

## Reading the Output

**Q4 🔥 What's the difference between the Local Address and Foreign Address columns?**
> Local Address is this machine's own IP and port for the socket. Foreign Address is the remote end's IP and port — shown as `0.0.0.0:*` for a listening socket that has no connected remote peer yet.

---

**Q5. What does it mean if a service's Local Address shows `0.0.0.0:3000` versus `127.0.0.1:3000`?**
> `0.0.0.0` means the service is bound to every available network interface, reachable from the local network (and potentially further, depending on firewall rules). `127.0.0.1` means it's bound only to the loopback interface, reachable exclusively from processes on that same machine — a meaningfully different exposure.

---

**Q6 🔥 What do Recv-Q and Send-Q represent?**
> Recv-Q is the number of bytes received by the kernel but not yet consumed by the local application. Send-Q is the number of bytes sent but not yet acknowledged by the remote end. Persistently non-zero values here can indicate an application not reading data fast enough, or network-level delivery problems.

---

## Connection States

**Q7 🔥 What does LISTEN mean, versus ESTABLISHED?**
> LISTEN means a socket is waiting for incoming connection attempts — a running server ready to accept clients. ESTABLISHED means an active, fully connected session currently exchanging data.

---

**Q8. What's the difference between TIME_WAIT and CLOSE_WAIT?**
> TIME_WAIT is a brief, normal state the kernel holds a socket in after closing, to catch any stray delayed packets — generally not a problem even in large numbers. CLOSE_WAIT means the remote end has closed the connection but the LOCAL application hasn't closed its own side yet — a persistently growing CLOSE_WAIT count usually does indicate an application failing to properly close sockets.

---

**Q9 🔥 Is a very high TIME_WAIT count something to be concerned about?**
> Generally not by itself — it's the normal, expected state for connections that have recently closed, and a busy server handling many short-lived connections routinely accumulates a large TIME_WAIT count as part of healthy operation, not evidence of a leak.

---

## Internals

**Q10. Where does netstat get its data from on Linux?**
> The `/proc` filesystem — specifically files like `/proc/net/tcp`, `/proc/net/udp`, and their IPv6 equivalents, which the kernel maintains continuously with raw, hex-encoded socket data that netstat parses and formats.

---

**Q11 🔥 Why does using `-p` sometimes show a dash instead of a process name, even for a socket that's clearly in use?**
> Because determining the owning process requires cross-referencing the socket against other processes' `/proc/<pid>/fd/` entries, which requires sufficient privileges — without root (or ownership of the process), netstat silently shows a dash rather than an error, which is easy to misread as "nothing owns this socket."

---

**Q12. Why does `-n` matter beyond just formatting?**
> Without `-n`, netstat attempts reverse DNS lookups for every foreign IP and service-name lookups for every port shown — on a busy connection table, or with slow/unreachable DNS, this can make the command noticeably slower. `-n` skips all of this resolution and shows raw numeric values instead.

---

## netstat vs Other Tools

**Q13 🔥 What is netstat's modern recommended replacement, and why?**
> `ss`, from the `iproute2` package — it uses a more efficient, direct netlink kernel interface rather than parsing `/proc/net/*` text files, making it significantly faster, especially on systems with a large number of active connections, and it's the tool actively receiving new functionality going forward.

---

**Q14. How does `netstat -tulpn` differ conceptually from `nmap`?**
> `netstat` reports the LOCAL machine's own view of its socket state (from the inside) — what's genuinely listening or connected according to the kernel itself. `nmap` probes a REMOTE host's ports from outside the network, inferring what's open based on how that remote host responds to connection attempts — a fundamentally different, external vantage point.

---

## Scenario-Based

**Q15 🔥 A colleague runs `netstat` with no options expecting to see which services are listening, but the output looks nothing like what they expected. What's the likely misunderstanding?**
> Plain `netstat` defaults to showing active (established) connections, not listening ports — a very common first-time confusion. They need `-l` (commonly combined as `-tuln` for TCP+UDP, numeric, listening-only) to see listening services specifically.

---

**Q16. `netstat -tulpn` shows a dash in the process column for several sockets, even though the user is confident those services are running. What's the fix?**
> The command is very likely being run without sufficient privileges — process-to-socket mapping requires reading other users' `/proc/<pid>/fd/` entries, which needs root or ownership of the process. Re-running with `sudo netstat -tulpn` should reveal the process names for sockets the current unprivileged user couldn't previously see.

---

**Q17 🔥 A newly deployed service doesn't appear at all in `netstat -tulpn` output, even though the application logs claim it started successfully and is listening. What are two likely explanations worth checking?**
> First, the service might actually be communicating over a UNIX domain socket rather than a TCP/UDP network socket, in which case `-tu` alone won't show it — `-x` (for UNIX sockets) is needed instead. Second, if running inside a container, network namespace isolation means netstat run from the HOST won't see a socket that's only bound inside a different container's namespace (or vice versa) — checking from the correct namespace/context matters.

---

**Q18. During a security review, someone flags a large number of connections in TIME_WAIT as an urgent problem, but a teammate says it's likely fine. Who's more likely correct, and why?**
> The teammate — TIME_WAIT is the normal, expected state for a connection that has recently closed, held briefly by the kernel to catch delayed packets, and a busy server naturally accumulates many of them as routine, healthy behavior. A genuinely concerning pattern would instead be a persistently growing, never-shrinking CLOSE_WAIT count specifically, which more reliably points to an application failing to close its sockets properly.

---

**Q19 🔥 A script runs `netstat -tulpn` on a fresh minimal Docker base image and gets "command not found." What's the most likely explanation, and what's the better long-term fix rather than just installing net-tools?**
> `net-tools` (the package providing netstat) is deprecated and frequently omitted from minimal/modern base images by default. While installing `net-tools` explicitly would resolve the immediate error, the more sustainable fix is switching the script to use `ss`, netstat's actively maintained replacement, which is far more likely to be present by default and receives ongoing support.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
