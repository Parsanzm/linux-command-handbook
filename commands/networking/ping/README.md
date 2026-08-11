# ping — The Complete Reference

> **Test reachability of a host and measure round-trip time**
> Almost certainly the very first command anyone runs when asking
> "is the network even working?"

---

## Table of Contents

- [What is ping?](#what-is-ping)
- [Where does ping live?](#where-does-ping-live)
- [How ping works internally](#how-ping-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Reading the Summary Statistics](#reading-the-summary-statistics)
- [All Key Options](#all-key-options)
- [ICMP Message Types Involved](#icmp-message-types-involved)
- [ping vs traceroute vs mtr vs curl](#ping-vs-traceroute-vs-mtr-vs-curl)
- [Related Commands](#related-commands)

---

## What is ping?

`ping` sends a small network packet (an ICMP Echo Request) to a target host and waits for that host to send back a reply (an ICMP Echo Reply), reporting whether a response arrived and how long the round trip took. It's the most basic, fundamental network reachability test in existence.

```bash
ping google.com
# PING google.com (142.250.80.14) 56(84) bytes of data.
# 64 bytes from lga25s79-in-f14.1e100.net (142.250.80.14): icmp_seq=1 ttl=115 time=12.4 ms
# 64 bytes from lga25s79-in-f14.1e100.net (142.250.80.14): icmp_seq=2 ttl=115 time=11.8 ms
# ^C
# --- google.com ping statistics ---
# 2 packets transmitted, 2 received, 0% packet loss, time 1001ms
# rtt min/avg/max/mdev = 11.800/12.100/12.400/0.300 ms
```

**Why ping is the universal first troubleshooting step:** it answers the most fundamental network question — "can this machine reach that machine at all" — before investigating anything more specific like DNS, a particular port, or an application-level issue.

---

## Where does ping live?

```
/usr/bin/ping  (or /bin/ping on some systems)
```

```bash
which ping
ping -V
# ping from iputils 20240117
```

Part of the **iputils** package on Linux. Present by default on virtually every operating system — Linux, macOS, BSD, and Windows all ship their own `ping` implementation, though flag syntax differs meaningfully between them (see [edge-cases.md](edge-cases.md)).

---

## How ping works internally

`ping` uses the **ICMP** (Internet Control Message Protocol) — a low-level protocol that operates directly at the IP layer, separate from TCP or UDP entirely. It sends an ICMP **Echo Request** packet to the target and waits for a matching ICMP **Echo Reply**.

```bash
# Each request/reply pair is matched using a sequence number:
# icmp_seq=1, icmp_seq=2, ... — letting ping detect if a specific
# reply is missing (packet loss) or arrives out of order
```

Because `ping` operates below the transport layer (no TCP/UDP port involved at all), it requires elevated privileges or a specific capability on some systems to create the raw socket needed to send ICMP packets directly — modern Linux typically handles this transparently via a setuid binary or granted capability, so most users never notice this requirement in practice.

```bash
# The specific capability involved, if inspecting binary permissions directly:
getcap /usr/bin/ping
# /usr/bin/ping cap_net_raw=ep
```

---

## Syntax

```bash
ping [OPTIONS] destination
```

By default, `ping` runs **indefinitely**, sending one packet per second, until manually stopped with `Ctrl+C` — at which point it prints summary statistics for everything sent so far.

---

## Understanding the Output

```bash
ping -c 4 example.com
# PING example.com (93.184.216.34) 56(84) bytes of data.
# 64 bytes from 93.184.216.34: icmp_seq=1 ttl=56 time=14.2 ms
# 64 bytes from 93.184.216.34: icmp_seq=2 ttl=56 time=13.9 ms
# 64 bytes from 93.184.216.34: icmp_seq=3 ttl=56 time=14.5 ms
# 64 bytes from 93.184.216.34: icmp_seq=4 ttl=56 time=13.7 ms
```

| Field | Meaning |
|---|---|
| `64 bytes` | Size of the reply packet received |
| `93.184.216.34` | The resolved IP address actually being pinged |
| `icmp_seq` | Sequence number, used to detect lost or reordered packets |
| `ttl` | Time To Live remaining on the reply — decreases by 1 at each router hop; a lower-than-expected value can hint at how many hops away the target is |
| `time` | Round-trip time in milliseconds — how long the request took to reach the target and the reply to come back |

---

## Reading the Summary Statistics

```bash
--- example.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 13.700/14.075/14.500/0.300 ms
```

| Field | Meaning |
|---|---|
| `packets transmitted` | How many Echo Requests were sent |
| `received` | How many Echo Replies actually came back |
| `packet loss` | Percentage of requests that got no reply at all |
| `rtt min/avg/max` | The fastest, average, and slowest round-trip times observed |
| `mdev` | Mean deviation — roughly how much round-trip times varied (jitter); a high value alongside a normal average suggests an inconsistent, unstable connection rather than a uniformly slow one |

---

## All Key Options

| Option | Description |
|---|---|
| `-c COUNT` | Stop after sending this many packets, instead of running indefinitely |
| `-i SECONDS` | Interval between packets (default 1 second) |
| `-W SECONDS` | Timeout to wait for a reply before considering it lost |
| `-s SIZE` | Set the payload size in bytes (default 56, resulting in 64-byte total packets) |
| `-t TTL` | Set the initial Time To Live value |
| `-q` | Quiet — only print the summary at the end, not each individual reply |
| `-v` | Verbose |
| `-4` / `-6` | Force IPv4 or IPv6 explicitly |
| `-f` | Flood ping — send packets as fast as possible (requires root; can be disruptive) |
| `-D` | Print a timestamp before each line |
| `-a` | Audible ping — beep on each successful reply |

---

## ICMP Message Types Involved

| ICMP Type | Meaning |
|---|---|
| Type 8 | Echo Request — what `ping` sends |
| Type 0 | Echo Reply — what a successful `ping` receives back |
| Type 3 | Destination Unreachable — the target (or an intermediate router) explicitly reports it can't be reached |
| Type 11 | Time Exceeded — a packet's TTL hit zero before reaching the destination (used by `traceroute`'s underlying mechanism) |

A **lack** of any reply at all (rather than an explicit Type 3 "unreachable") often means a firewall is silently **dropping** the ICMP packets rather than actively rejecting them — see [edge-cases.md](edge-cases.md) for why "no reply" doesn't necessarily mean "host is down."

---

## ping vs traceroute vs mtr vs curl

| Tool | Best for | Key difference from ping |
|---|---|---|
| `ping` | Basic reachability + round-trip time to ONE target | Answers "can I reach it, and how fast," nothing about the path taken |
| `traceroute` | Seeing the full path (each router hop) to a destination | Reveals WHERE along the route a problem occurs, not just whether the final destination responds |
| `mtr` | Continuous, live combination of ping + traceroute | Shows per-hop loss/latency continuously updating, rather than a single static traceroute snapshot |
| `curl` | Application-level (HTTP) reachability, not just network-level | Tests whether an actual SERVICE (web server) responds, which ping's ICMP-only test can't confirm at all |

```bash
ping example.com        # is the host reachable at the network level at all?
traceroute example.com   # which path/hops does traffic take to get there?
mtr example.com           # live, continuously updating version of the above
curl -I https://example.com   # is the actual WEB SERVER responding, not just the network path?
```

---

## Related Commands

| Command | Relation |
|---|---|
| `traceroute` / `tracepath` | Show the hop-by-hop path to a destination |
| `mtr` | Live, combined ping + traceroute view |
| `dig` / `nslookup` | DNS resolution — check if a hostname resolves before assuming a ping failure is network-related |
| `curl` / `nc` | Application/port-level connectivity testing beyond ICMP reachability alone |
| `arp` / `ip neigh` | Layer-2 (local network) address resolution, relevant for local-subnet ping troubleshooting |
| `netstat` / `ss` | Check whether a specific service/port is actually listening, once basic reachability is confirmed |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
