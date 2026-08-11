# ping — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Reading the Output](#reading-the-output)
- [ICMP Fundamentals](#icmp-fundamentals)
- [Internals](#internals)
- [ping vs Other Tools](#ping-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does ping do?**
> Sends an ICMP Echo Request to a target host and waits for an ICMP Echo Reply, reporting whether a response arrived and how long the round trip took — the most basic network reachability test available.

---

**Q2 🔥 What protocol does ping use, and how is that different from TCP or UDP?**
> ICMP (Internet Control Message Protocol) — a low-level protocol operating directly at the IP layer, with no concept of ports the way TCP/UDP have. It's used for network-level diagnostics and control messages, not for carrying application data.

---

**Q3. Does ping run once and exit, or continuously by default?**
> Continuously by default — it sends one packet per second indefinitely until manually stopped with Ctrl+C, at which point it prints summary statistics. `-c COUNT` limits it to a fixed number of packets instead.

---

## Reading the Output

**Q4 🔥 What does the ttl field in a ping reply indicate?**
> The remaining Time To Live value on the received reply packet — TTL starts at some initial value (OS-dependent) and decreases by one at each router hop; the value on a reply can give a rough sense of how many hops away the target is, though it's not a precise or fully reliable measurement.

---

**Q5. What's the difference between packet loss and round-trip time (RTT) in ping's summary?**
> Packet loss is the percentage of sent requests that received no reply at all. RTT (round-trip time) measures how long successful replies took to come back — the two are separate metrics; a connection can have low packet loss but high RTT (slow but reliable), or vice versa.

---

**Q6 🔥 What does mdev represent in ping's summary statistics?**
> Mean deviation — roughly a measure of how much round-trip times varied across all the replies (jitter). A high mdev alongside a normal average RTT suggests an inconsistent, unstable connection rather than a uniformly slow but stable one.

---

## ICMP Fundamentals

**Q7. What ICMP message type does ping send, and what type does a successful reply use?**
> Ping sends an ICMP Type 8 (Echo Request) and a successful response comes back as an ICMP Type 0 (Echo Reply).

---

**Q8 🔥 If ping receives no reply at all, does that always mean the target host is powered off or crashed?**
> No — it could also mean a firewall along the path is silently dropping ICMP packets (very common and often intentional), or the target is deliberately configured to ignore ICMP Echo Requests as a security practice, while the actual service on that host remains fully functional and reachable via its own protocol.

---

## Internals

**Q9. Why does ping sometimes require elevated privileges or a specific capability to run?**
> Because it operates at a level below TCP/UDP, requiring a raw socket to construct and send ICMP packets directly — modern Linux typically grants this transparently via a setuid binary or a specific capability (`cap_net_raw`) rather than requiring the user to run as root explicitly.

---

**Q10 🔥 How does ping match up a specific reply to the request that generated it?**
> Via the icmp_seq sequence number included in each Echo Request/Reply pair — this lets ping detect which specific requests received no reply (packet loss) or if replies arrived out of order.

---

## ping vs Other Tools

**Q11 🔥 What's the difference between ping and traceroute?**
> `ping` tests reachability and round-trip time to a single destination directly. `traceroute` reveals the full path — each router hop — that traffic takes to reach that destination, which helps identify WHERE along the route a problem is occurring, not just whether the final target responds.

---

**Q12. Why might curl be a better tool than ping for confirming a website is actually working?**
> Because a successful ping only confirms the host responds to ICMP — it says nothing about whether the actual web server application is running or responding correctly. `curl` (or similar) tests the real HTTP service directly, which is what actually matters for "is the website up."

---

## Scenario-Based

**Q13 🔥 A teammate says "example.com must be down, ping isn't getting any replies" — but the website loads fine in a browser. What's the most likely explanation?**
> The web server itself is very likely fine — many cloud providers, CDNs, and load balancers deliberately block or rate-limit ICMP traffic as standard policy, which is unrelated to whether the actual HTTP service is functioning. A successful `curl -I https://example.com` alongside a failed ping would confirm this is expected ICMP filtering, not an actual outage.

---

**Q14. A script embeds a bare `ping example.com` with no additional flags and appears to hang indefinitely when run non-interactively. Why?**
> Without `-c`, ping runs forever by default, sending packets once per second until manually interrupted — in a non-interactive script context with no Ctrl+C available, this hangs indefinitely. Adding `-c` (and typically `-W` for a bounded timeout) fixes this.

---

**Q15 🔥 `ping example.com` fails with "Name or service not known," while `ping 93.184.216.34` (that same site's known IP) succeeds. What does this tell you about where the actual problem lies?**
> The failure is specifically a DNS resolution problem, not a general network connectivity issue — the hostname couldn't be translated to an IP address at all, while the underlying network path to that IP works fine once resolution is bypassed. This isolates the fault to DNS rather than routing/reachability.

---

**Q16. During an incident, `ping` to a server shows consistently high round-trip times (300ms+) compared to its usual baseline. Does this definitively mean the network path is congested?**
> Not definitively — while it could reflect genuine network path latency or congestion, it could equally reflect the target host itself being under heavy load and deprioritizing ICMP replies, or a rate-limiting policy intentionally delaying (not blocking) responses. Ping alone can't distinguish between these causes; tools like `traceroute`/`mtr` help localize where along the path delay is actually being introduced.

---

**Q17 🔥 Someone writes `ping -c 1 example.com | grep "bytes from"` and then checks `$?` expecting it to reflect whether the host was reachable, but it behaves inconsistently. What's wrong?**
> `$?` after a pipeline reflects the exit status of the LAST command in that pipeline — `grep` in this case — not `ping` itself, even though ping is what actually determined reachability. Checking ping's own exit status requires not piping its output into another command first (e.g., `ping -c 1 -W 2 example.com > /dev/null 2>&1; echo $?`).

---

**Q18. A colleague reports "1% packet loss" from a 100-packet ping test and is worried about a serious network problem. How would you help them assess whether this actually matters?**
> A very small amount of occasional packet loss (1-2%, especially over a larger sample size) is often normal background network behavior rather than a genuine ongoing problem. The more meaningful signal is a SUSTAINED, consistently high loss percentage recurring across multiple separate test runs, not an isolated single drop within an otherwise clean result.

---

**Q19 🔥 Someone tries `ping6 fe80::1` and gets "connect: Invalid argument." What's the issue, and how is it fixed?**
> IPv6 link-local addresses (the `fe80::/10` range) are only meaningful within the context of a specific network interface, since the same link-local address could exist on multiple interfaces simultaneously — the destination is ambiguous without specifying which interface to use. Adding `-I <interface>` (e.g., `ping6 -I eth0 fe80::1`) resolves the ambiguity.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
