# netstat — Edge Cases & Gotchas

> `netstat` looks like a straightforward "show me connections" tool, but
> its deprecation status, permission requirements, and default output
> scope routinely mislead people expecting something different.

---

## Table of Contents

- [netstat May Not Be Installed at All on a Modern/Minimal System](#netstat-may-not-be-installed-at-all-on-a-modernminimal-system)
- [Plain netstat Shows Active Connections, NOT Listening Ports](#plain-netstat-shows-active-connections-not-listening-ports)
- [-p Without root Silently Hides the Process Column](#-p-without-root-silently-hides-the-process-column)
- [0.0.0.0 vs 127.0.0.1 in Local Address Means Completely Different Exposure](#000000-vs-127001-in-local-address-means-completely-different-exposure)
- [-n Isn't Just Cosmetic — Skipping It Can Be Dramatically Slower](#-n-isnt-just-cosmetic--skipping-it-can-be-dramatically-slower)
- [A High TIME_WAIT Count Usually Isn't a Problem by Itself](#a-high-time_wait-count-usually-isnt-a-problem-by-itself)
- [netstat Inside a Container Only Shows That Container's Own Network Namespace](#netstat-inside-a-container-only-shows-that-containers-own-network-namespace)
- [netstat's Output Format Differs Meaningfully Between Linux, macOS, and BSD](#netstats-output-format-differs-meaningfully-between-linux-macos-and-bsd)
- [Deprecated Doesn't Mean Gone — But New Features Land in ss, Not netstat](#deprecated-doesnt-mean-gone--but-new-features-land-in-ss-not-netstat)
- [A Snapshot, Not a Live Feed — Fast-Churning Connections Can Be Missed](#a-snapshot-not-a-live-feed--fast-churning-connections-can-be-missed)
- [UNIX Domain Sockets Need a Separate Flag Entirely](#unix-domain-sockets-need-a-separate-flag-entirely)

---

## netstat May Not Be Installed at All on a Modern/Minimal System

### A frequent surprise on newer distros, minimal images, and containers
```bash
netstat -tulpn
# bash: netstat: command not found
# ⚠️ net-tools (the package providing netstat) has been deprecated in
# favor of iproute2 (ss, ip) for years, and many modern minimal
# installs — slim container base images, newer distro defaults, some
# cloud VM images — simply don't include it anymore.

# Either install it explicitly:
sudo apt install net-tools    # Debian/Ubuntu
sudo dnf install net-tools    # Fedora/RHEL

# Or, more sustainably, use the modern equivalent that's virtually
# always present instead:
ss -tulpn
```

---

## Plain netstat Shows Active Connections, NOT Listening Ports

### A very common first-time surprise
```bash
netstat
# Active Internet connections (w/o servers)
# Proto Recv-Q Send-Q Local Address    Foreign Address    State
# tcp        0      0 myhost:52104     93.184.216.34:https ESTABLISHED
# ⚠️ Plain, no-argument netstat shows ESTABLISHED connections by
# default — it does NOT show listening services at all, which is
# often what someone actually wants when they first reach for netstat
# ("what's running/listening on this machine").

# For listening services specifically, -l (or -tuln for the common
# TCP+UDP, numeric, listening-only combination) is required explicitly:
netstat -tuln
```

---

## -p Without root Silently Hides the Process Column

### No error, just missing information
```bash
netstat -tulpn
# tcp   0   0  0.0.0.0:22   0.0.0.0:*   LISTEN   -
# tcp   0   0  127.0.0.1:5432  0.0.0.0:* LISTEN   -
# ⚠️ Without sufficient privileges, netstat can't cross-reference a
# socket back to its owning process (which requires reading other
# users' /proc/<pid>/fd/ entries) — rather than an error, it just
# silently shows a dash "-" in the PID/Program column for anything
# it doesn't have permission to inspect. This is easy to mistake for
# "nothing owns this socket" rather than "I don't have permission to see."

sudo netstat -tulpn
# tcp   0   0  0.0.0.0:22   0.0.0.0:*   LISTEN   612/sshd
# The PID/process names for OTHER users' sockets are now visible with
# elevated privileges
```

---

## 0.0.0.0 vs 127.0.0.1 in Local Address Means Completely Different Exposure

### A single-character difference with major security/reachability implications
```bash
netstat -tulpn | grep myapp
# tcp   0   0  127.0.0.1:3000   0.0.0.0:*   LISTEN   1234/myapp
# ⚠️ 127.0.0.1 means the service is bound ONLY to the loopback
# interface — reachable exclusively from processes running ON THIS
# SAME MACHINE, not from anywhere else on the network at all.

netstat -tulpn | grep myapp
# tcp   0   0  0.0.0.0:3000     0.0.0.0:*   LISTEN   1234/myapp
# ⚠️ 0.0.0.0 means the service is bound to EVERY available network
# interface — reachable from the local network (and potentially the
# broader internet, if not otherwise firewalled) — a meaningfully
# different, much wider exposure than the 127.0.0.1 case above.

# Confusing these two when auditing what's actually reachable from
# outside the machine is a common and consequential misreading.
```

---

## -n Isn't Just Cosmetic — Skipping It Can Be Dramatically Slower

### DNS/service-name resolution happens per-entry, and can hang on unreachable resolvers
```bash
netstat -atp
# ⚠️ WITHOUT -n, netstat attempts to resolve EVERY foreign IP address
# to a hostname (reverse DNS) and every port number to a known service
# name — on a busy connection table, or in an environment with slow or
# unreachable DNS resolution, this can make the command take
# noticeably (sometimes dramatically) longer to complete than expected.

netstat -atpn
# -n skips ALL of this resolution, showing raw numeric IPs/ports —
# almost always the better default for anything beyond casual,
# occasional interactive use, and essential when scripting
```

---

## A High TIME_WAIT Count Usually Isn't a Problem by Itself

### A frequent overreaction to normal, expected TCP behavior
```bash
netstat -ant | grep -c TIME_WAIT
# 1847
# ⚠️ A large number of TIME_WAIT connections often LOOKS alarming,
# but TIME_WAIT is the NORMAL, expected state a connection sits in
# briefly after closing (holding the socket for a period to catch any
# stray delayed packets) — a busy server handling many short-lived
# connections (a web server under normal load, for instance) can
# accumulate a large TIME_WAIT count as routine, healthy behavior,
# not evidence of a leak or a problem on its own.

# A genuinely concerning pattern is instead a persistently GROWING,
# never-shrinking count of CLOSE_WAIT specifically (which usually
# does indicate an application failing to close sockets it's finished
# with), rather than a large but STABLE TIME_WAIT count:
netstat -ant | awk '{print $6}' | sort | uniq -c | sort -rn
```

---

## netstat Inside a Container Only Shows That Container's Own Network Namespace

### Namespace isolation changes visibility, similar to how it affects ps
```bash
docker run --rm -it debian bash -c "apt install -y net-tools && netstat -tulpn"
# ⚠️ Thanks to network namespace isolation, netstat run inside a
# properly isolated container only sees sockets within that
# container's OWN network namespace — not the host's full socket
# table, and not other containers' sockets. This is expected and
# usually desired, but can be confusing if checking from inside a
# container while expecting to see host-level services (a reverse
# proxy, another container) that are actually invisible from this
# isolated vantage point.

# Checking from the HOST's own network namespace shows the full
# picture instead, if that's genuinely what's needed:
sudo netstat -tulpn   # run on the HOST, not inside the container
```

---

## netstat's Output Format Differs Meaningfully Between Linux, macOS, and BSD

### Flags and column layouts aren't fully portable across Unix variants
```bash
# Linux (net-tools):
netstat -tulpn

# macOS's built-in netstat:
netstat -an
# ⚠️ macOS's netstat doesn't support -p (no direct process-to-socket
# mapping the same way) and uses a somewhat different column layout
# and flag set overall — cross-platform scripts relying on netstat's
# exact output format need OS-specific handling rather than assuming
# a single universal syntax works everywhere.

# On macOS, `lsof -i` is generally the more practical way to map a
# listening port back to its owning process instead:
lsof -i :3000
```

---

## Deprecated Doesn't Mean Gone — But New Features Land in ss, Not netstat

### net-tools is effectively frozen; don't expect new capabilities here
```bash
# net-tools (netstat, ifconfig, route, arp) has been in a maintenance-
# only state for a long time — it still WORKS and is still packaged
# by most distros, but it does not receive new functionality, and
# some newer kernel networking features may not be reflected in its
# output at all, or only imperfectly.

# ⚠️ Documentation, scripts, and habits built around netstat remain
# valid for existing use cases, but anything requiring newer kernel
# networking visibility (certain modern socket options, newer
# protocol support) is far more likely to be properly supported in
# ss rather than netstat going forward.
ss --version
```

---

## A Snapshot, Not a Live Feed — Fast-Churning Connections Can Be Missed

### The same fundamental limitation ps and top share
```bash
netstat -ant
# ⚠️ This is a single, static SNAPSHOT at the moment it's run — a
# connection that opens and closes very quickly (a fast health-check
# ping, a brief handshake-and-disconnect) can be entirely invisible
# if it happens to occur between two separate netstat invocations,
# the same way a short-lived process can escape a single `ps` snapshot.

# For genuinely continuous monitoring, loop it (some builds also
# support -c for continuous mode) or use a purpose-built capture tool:
watch -n 1 'netstat -ant | grep ESTABLISHED | wc -l'
# or, for actual packet-level visibility:
sudo tcpdump -i any port 443
```

---

## UNIX Domain Sockets Need a Separate Flag Entirely

### -t/-u only cover network sockets, not local IPC sockets
```bash
netstat -tulpn | grep myapp
# (nothing found, even though myapp is definitely running and
# listening on a local socket file)
# ⚠️ -t and -u restrict output to TCP and UDP (network) sockets only
# — a process communicating over a UNIX domain socket (a local
# filesystem-path-based socket, common for things like Docker's own
# daemon socket, or many local IPC mechanisms) won't appear at all
# under -tu alone, since it's a fundamentally different socket family.

# The separate flag for UNIX domain sockets:
netstat -xlp
# x = UNIX domain sockets specifically
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
