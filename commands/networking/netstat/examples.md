# netstat — Practical Examples

> Real-world patterns for finding listening services, diagnosing
> connection issues, and inspecting routing/interface stats.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Finding What's Listening on a Port](#finding-whats-listening-on-a-port)
- [Inspecting Active Connections](#inspecting-active-connections)
- [Routing and Interface Statistics](#routing-and-interface-statistics)
- [Filtering and Counting](#filtering-and-counting)
- [Combining netstat with Other Tools](#combining-netstat-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# All active connections (default — established, not listening)
netstat

# TCP and UDP listening sockets, with owning process, numeric output
netstat -tulpn

# Every TCP connection, established or not
netstat -antp
```

---

## Finding What's Listening on a Port

```bash
# All listening TCP/UDP ports and what owns them
sudo netstat -tulpn

# Just check whether a specific port is in use
sudo netstat -tulpn | grep ':8080'

# Find the PID of whatever's listening on port 5432
sudo netstat -tulpn | grep ':5432' | awk '{print $7}' | cut -d/ -f1
```

---

## Inspecting Active Connections

```bash
# All established TCP connections
netstat -atp | grep ESTABLISHED

# Connections to/from a specific remote IP
netstat -an | grep '203.0.113.5'

# Count connections per state
netstat -ant | awk '{print $6}' | sort | uniq -c | sort -rn

# Show connections along with the owning user (not just PID)
sudo netstat -tup -e
```

---

## Routing and Interface Statistics

```bash
# Kernel routing table
netstat -r

# Numeric version, skipping DNS/service lookups (faster, and useful
# when name resolution itself might be part of the problem)
netstat -rn

# Per-interface packet/error/drop statistics
netstat -i

# Summary statistics broken down by protocol
netstat -s
```

---

## Filtering and Counting

```bash
# Count how many connections are in TIME_WAIT
netstat -ant | grep -c TIME_WAIT

# Find the top remote IPs by number of connections to this machine
netstat -ant | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head

# List only LISTEN-state sockets, no established connections cluttering the view
netstat -tuln
```

---

## Combining netstat with Other Tools

```bash
# Cross-reference a listening port with the fuller lsof view
sudo lsof -i :22

# Watch connection counts change live (netstat has no built-in refresh
# in some builds, so pair it with watch)
watch -n 2 'netstat -ant | grep -c ESTABLISHED'

# Combine with grep to check for a specific known-bad IP actively connected
netstat -an | grep '198.51.100.0'
```

---

## Real-World Recipes

```bash
# --- Quick "what's this server actually listening on" audit ---
sudo netstat -tulpn | grep LISTEN

# --- Confirm a Service Actually Bound to the Expected Port After Deploy ---
sudo netstat -tulpn | grep ':3000'
# empty output means the service ISN'T listening as expected — investigate

# --- Detect a Possible Connection Leak (excessive TIME_WAIT/CLOSE_WAIT) ---
netstat -ant | awk '{print $6}' | sort | uniq -c | sort -rn
# a very high CLOSE_WAIT count often points to an application not
# properly closing sockets it's done with

# --- Check Whether a Service Is Bound to localhost Only, or All Interfaces ---
sudo netstat -tulpn | grep myapp
# 127.0.0.1:3000   ← only reachable locally
# 0.0.0.0:3000     ← reachable from any interface/network

# --- Fleet-Wide Check for an Unexpected Open Port ---
for server in web1 web2 db1; do
  echo "=== $server ==="
  ssh "$server" "sudo netstat -tulpn | grep ':4444'"
done

# --- Basic Traffic Volume Sanity Check ---
netstat -i
# review RX-OK/TX-OK vs RX-ERR/TX-ERR/RX-DRP/TX-DRP columns for signs
# of a flaky NIC or an overloaded interface
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
