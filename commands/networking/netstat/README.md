# netstat — The Complete Reference

> **Print network connections, listening ports, and routing/interface statistics**
> A classic diagnostic tool for "what's listening on this machine, and
> what's connected to what" — now formally deprecated in favor of `ss`,
> but still extremely widely used and documented.

---

## Table of Contents

- [What is netstat?](#what-is-netstat)
- [Where does netstat live?](#where-does-netstat-live)
- [How netstat works internally](#how-netstat-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Connection States Explained](#connection-states-explained)
- [Common Combinations](#common-combinations)
- [All Key Options](#all-key-options)
- [netstat and /proc](#netstat-and-proc)
- [netstat vs ss vs lsof vs ip](#netstat-vs-ss-vs-lsof-vs-ip)
- [Related Commands](#related-commands)

---

## What is netstat?

`netstat` ("network statistics") prints information about the machine's network subsystem: active TCP/UDP connections, which ports are listening for incoming connections, the routing table, and per-interface traffic statistics — all from a single, long-standing tool.

```bash
netstat -tulpn
# Proto Recv-Q Send-Q Local Address    Foreign Address   State    PID/Program name
# tcp        0      0 0.0.0.0:22       0.0.0.0:*         LISTEN   612/sshd
# tcp        0      0 127.0.0.1:5432   0.0.0.0:*         LISTEN   1102/postgres
```

**Why netstat is still so commonly referenced despite being deprecated:** decades of documentation, tutorials, and muscle memory reference it directly, and it remains installed (via the `net-tools` package) on a large share of systems even though its official replacement, `ss`, is faster and actively maintained.

---

## Where does netstat live?

```
/usr/bin/netstat  (or /bin/netstat on some systems)
```

```bash
which netstat
netstat --version
# net-tools 2.10-alpha
```

Part of the **net-tools** package — the same collection that historically included `ifconfig`, `route`, and `arp`. This entire package has been in **maintenance mode / considered deprecated** on Linux for years, superseded by the `iproute2` package (`ss`, `ip`), and on many modern minimal installs (some container base images, newer distro defaults) **net-tools isn't installed at all** by default anymore.

---

## How netstat works internally

Like `ps` and `top`, `netstat` reads its data from the **`/proc` filesystem** on Linux — specifically files tracking active sockets, maintained continuously by the kernel's networking stack.

```bash
cat /proc/net/tcp     # active TCP sockets (IPv4), in raw hex format
cat /proc/net/udp      # active UDP sockets (IPv4)
cat /proc/net/tcp6      # active TCP sockets (IPv6)
```

`netstat` parses these raw, hex-encoded kernel structures and cross-references them against `/proc/<pid>/fd/` to determine which **process** owns each socket (only shown with `-p`, and only fully populated when run as root or the owning user). This cross-referencing step — matching sockets to owning processes — is comparatively slow, which is part of why `ss` (which uses a more efficient netlink-based kernel interface directly) has largely replaced it for this specific lookup.

---

## Syntax

```bash
netstat [OPTIONS]
```

With no options, `netstat` defaults to listing **active connections** (established TCP/UDP sockets), not listening ports — a very common point of confusion for anyone expecting to see listening services by default.

---

## Understanding the Output

```bash
netstat -tulpn
# Proto Recv-Q Send-Q Local Address      Foreign Address    State    PID/Program name
# tcp        0      0 0.0.0.0:22         0.0.0.0:*          LISTEN   612/sshd
# tcp        0      0 127.0.0.1:5432     0.0.0.0:*          LISTEN   1102/postgres
# tcp        0      0 192.168.1.5:22     192.168.1.10:54321 ESTABLISHED 2201/sshd
```

| Column | Meaning |
|---|---|
| `Proto` | Protocol — `tcp`, `udp`, `tcp6`, `udp6` |
| `Recv-Q` | Bytes received but not yet consumed by the local application |
| `Send-Q` | Bytes sent but not yet acknowledged by the remote end |
| `Local Address` | This machine's IP:port for the socket |
| `Foreign Address` | The remote end's IP:port (or `0.0.0.0:*` for a listening socket with no remote peer yet) |
| `State` | Connection state — see [Connection States Explained](#connection-states-explained) |
| `PID/Program name` | Owning process, only shown with `-p` and sufficient permissions |

---

## Connection States Explained

TCP connections progress through a well-defined set of states, all visible in netstat's `State` column:

| State | Meaning |
|---|---|
| `LISTEN` | Socket is waiting for incoming connections (a running server) |
| `ESTABLISHED` | An active, fully connected session with data flowing |
| `TIME_WAIT` | Connection has closed but the kernel is holding the socket briefly to catch any delayed packets |
| `CLOSE_WAIT` | The remote end has closed the connection, but the LOCAL application hasn't closed its side yet |
| `SYN_SENT` | This machine has sent a connection request and is waiting for a reply |
| `SYN_RECV` | This machine has received a connection request and is completing the handshake |
| `FIN_WAIT1` / `FIN_WAIT2` | This machine has initiated closing the connection |
| `CLOSING` | Both sides are closing simultaneously |
| `LAST_ACK` | This machine is waiting for a final acknowledgment before fully closing |

UDP, being connectionless, generally shows no meaningful `State` value at all for its sockets.

---

## Common Combinations

```bash
netstat -tulpn
# t = TCP, u = UDP, l = listening only, p = show owning process, n = numeric (skip DNS lookups)
# The single most common netstat invocation for "what's listening on this machine"

netstat -antp
# a = all sockets, n = numeric, t = TCP, p = show process
# Common variant for "show me every TCP connection, established or not"
```

---

## All Key Options

| Option | Description |
|---|---|
| `-t` | Show TCP sockets |
| `-u` | Show UDP sockets |
| `-l` | Show only listening sockets |
| `-a` | Show all sockets (listening and established) |
| `-n` | Show numeric addresses/ports — skip DNS/service-name resolution (much faster) |
| `-p` | Show the owning process's PID and name (requires root/ownership for full detail) |
| `-r` | Show the kernel routing table (equivalent to the older `route` command) |
| `-i` | Show per-interface statistics (packets, errors, drops) |
| `-s` | Show summary statistics per protocol |
| `-c` | Continuously refresh, like `watch` built in |
| `-e` | Extended info — also show the owning user |
| `-o` | Show networking timers |

---

## netstat and /proc

```bash
# The raw source data behind netstat's TCP output
cat /proc/net/tcp
#   sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt uid  ...
#   0: 00000000:0016 00000000:0000 0A 00000000:00000000 00:00000000 00000000 0 ...
# Local/remote addresses are encoded in little-endian HEX — 0016 is
# port 22 (0x16 = 22) in this raw form, which netstat decodes into
# the human-readable "0.0.0.0:22" shown in its normal output.
```

---

## netstat vs ss vs lsof vs ip

| Tool | Best for | Key difference from netstat |
|---|---|---|
| `ss` | The modern, faster, actively maintained replacement | Uses a direct netlink kernel interface instead of parsing `/proc/net/*` text files — significantly faster on systems with many connections |
| `lsof -i` | Cross-referencing network sockets with general open-file info | Broader tool (all open files, not just sockets); `-i` filters specifically to network connections |
| `ip addr` / `ip route` | Interface configuration and routing table | `ip` (from iproute2) is the modern replacement for netstat's `-i`/`-r` functionality specifically |
| `nmap` | Scanning ports on OTHER machines (external view) | netstat/ss show the LOCAL machine's own socket state; nmap probes REMOTE hosts from outside |

```bash
netstat -tulpn      # classic, widely documented, may need installing
ss -tulpn            # modern equivalent, faster, usually preinstalled
lsof -i :22           # what's using port 22 specifically, from the file-descriptor angle
ip addr                # interface configuration (replaces netstat -i / ifconfig)
```

---

## Related Commands

| Command | Relation |
|---|---|
| `ss` | The modern, recommended replacement for netstat's connection-listing functionality |
| `ip` | The modern replacement for netstat's routing/interface functionality (and `ifconfig`/`route`) |
| `lsof -i` | Cross-references sockets with the broader open-file view |
| `nmap` | External port scanning of remote hosts, as opposed to inspecting local socket state |
| `tcpdump` | Captures actual packet contents/traffic, rather than just socket state summaries |
| `curl` / `ping` | Application-level connectivity testing, complementary to netstat's state inspection |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
