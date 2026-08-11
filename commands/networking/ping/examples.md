# ping — Practical Examples

> Real-world patterns for connectivity checks, latency measurement,
> and basic network troubleshooting.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Limiting the Number of Packets](#limiting-the-number-of-packets)
- [Adjusting Timing and Payload](#adjusting-timing-and-payload)
- [Scripting with ping](#scripting-with-ping)
- [Checking Multiple Hosts](#checking-multiple-hosts)
- [Combining ping with Other Tools](#combining-ping-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Ping continuously until Ctrl+C
ping google.com

# Ping a fixed number of times, then stop automatically
ping -c 4 google.com

# Only show the final summary, not each individual reply
ping -c 10 -q google.com
```

---

## Limiting the Number of Packets

```bash
# Send exactly one packet — the quickest possible reachability check
ping -c 1 example.com

# Send 20 packets to get a more reliable average/packet-loss reading
ping -c 20 example.com
```

---

## Adjusting Timing and Payload

```bash
# Faster interval between packets (0.2 seconds instead of the default 1)
ping -i 0.2 -c 10 example.com

# Set a specific timeout for waiting on each reply
ping -c 5 -W 2 example.com

# Send a larger payload, useful for testing MTU-related issues
ping -c 4 -s 1400 example.com

# Test whether a specific packet size can be sent without fragmentation
ping -c 4 -M do -s 1472 example.com
```

---

## Scripting with ping

```bash
# Simple up/down check in a script
if ping -c 1 -W 2 example.com > /dev/null 2>&1; then
  echo "Host is reachable"
else
  echo "Host is NOT reachable"
fi

# Extract just the average round-trip time
ping -c 4 example.com | tail -1 | awk -F '/' '{print $5}'

# Extract just the packet loss percentage
ping -c 10 example.com | grep -oP '\d+(?=% packet loss)'

# Wait in a loop until a host comes back online
until ping -c 1 -W 1 example.com > /dev/null 2>&1; do
  echo "Still down, retrying..."
  sleep 5
done
echo "Host is back up"
```

---

## Checking Multiple Hosts

```bash
# Ping a list of hosts, one after another
for host in web1 web2 db1 api.example.com; do
  echo -n "$host: "
  ping -c 1 -W 1 "$host" > /dev/null 2>&1 && echo "UP" || echo "DOWN"
done

# Run pings against several hosts in parallel, collecting results
for host in web1 web2 db1; do
  (ping -c 3 -q "$host" | tail -2 > "/tmp/ping_$host.log") &
done
wait
cat /tmp/ping_*.log
```

---

## Combining ping with Other Tools

```bash
# Confirm DNS resolution succeeds as part of the same check
ping -c 1 example.com || dig example.com

# Follow up a failed ping with a traceroute to see where it breaks
ping -c 2 example.com || traceroute example.com

# Log a periodic latency check to a file
while true; do
  echo "$(date): $(ping -c 1 -W 2 example.com | tail -1)" >> /var/log/latency.log
  sleep 60
done
```

---

## Real-World Recipes

```bash
# --- Quick Internet Connectivity Sanity Check ---
ping -c 3 -q 1.1.1.1

# --- Confirm a Newly Provisioned Server Is Reachable Before Deploying ---
until ping -c 1 -W 2 "$NEW_SERVER_IP" > /dev/null 2>&1; do
  echo "Waiting for $NEW_SERVER_IP to come online..."
  sleep 5
done
echo "$NEW_SERVER_IP is reachable — proceeding with deployment"

# --- Fleet-Wide Reachability Sweep ---
for ip in $(cat server_ips.txt); do
  if ping -c 1 -W 1 "$ip" > /dev/null 2>&1; then
    echo "$ip: reachable"
  else
    echo "$ip: UNREACHABLE"
  fi
done

# --- Basic Latency Baseline Before/After a Network Change ---
ping -c 20 -q gateway.example.com > before_change.txt
# ... make the network change ...
ping -c 20 -q gateway.example.com > after_change.txt
diff before_change.txt after_change.txt

# --- Alert on Packet Loss Above a Threshold ---
LOSS=$(ping -c 20 -q example.com | grep -oP '\d+(?=% packet loss)')
if [ "$LOSS" -gt 5 ]; then
  echo "WARNING: ${LOSS}% packet loss to example.com"
fi
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
