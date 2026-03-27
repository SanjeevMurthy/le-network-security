# Lab 01: Linux Networking

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Prerequisites](#prerequisites)
- [Debugging Methodology Alignment](#debugging-methodology-alignment)
- [Lab 1A: Build a Network Namespace Bridge from Scratch](#lab-1a-build-a-network-namespace-bridge-from-scratch)
  - [Setup: Create the Bridge and Namespaces](#setup-create-the-bridge-and-namespaces)
  - [Setup: Create veth Pairs and Attach to Bridge](#setup-create-veth-pairs-and-attach-to-bridge)
  - [Verify: Connectivity Between Namespaces](#verify-connectivity-between-namespaces)
  - [Setup: Enable Internet Access (NAT/MASQUERADE)](#setup-enable-internet-access-natmasquerade)
  - [Inspect the Bridge Forwarding Database](#inspect-the-bridge-forwarding-database)
  - [Cleanup](#cleanup)
- [Lab 1B: Reproduce and Fix Conntrack Table Exhaustion](#lab-1b-reproduce-and-fix-conntrack-table-exhaustion)
  - [Background](#background)
  - [Setup: Verify Current Conntrack State](#setup-verify-current-conntrack-state)
  - [The Break: Reduce nf_conntrack_max to 100](#the-break-reduce-nf_conntrack_max-to-100)
  - [Symptoms: Flood Connections to Trigger Exhaustion](#symptoms-flood-connections-to-trigger-exhaustion)
  - [Diagnosis: Confirm Table Exhaustion](#diagnosis-confirm-table-exhaustion)
  - [Root Cause](#root-cause)
  - [Fix](#fix)
  - [Verify](#verify)
  - [Cleanup](#cleanup)
- [Lab 1C: Debug iptables Blocking with TRACE](#lab-1c-debug-iptables-blocking-with-trace)
  - [Setup: Start a Test Service](#setup-start-a-test-service)
  - [The Break: Add a DROP Rule](#the-break-add-a-drop-rule)
  - [Symptoms](#symptoms)
  - [Diagnosis: Identify the Blocking Rule with TRACE](#diagnosis-identify-the-blocking-rule-with-trace)
  - [Root Cause](#root-cause)
  - [Fix](#fix)
  - [Verify](#verify)
  - [Cleanup](#cleanup)
- [Summary: Linux Networking Lab Checklist](#summary-linux-networking-lab-checklist)
- [Interview Discussion Points](#interview-discussion-points)

---

## Learning Objectives

- Build container-style networking from scratch using Linux primitives (network namespaces, veth pairs, bridges)
- Reproduce and diagnose conntrack table exhaustion — a silent production killer
- Use iptables TRACE to find which rule is dropping traffic without grep-and-pray

## Prerequisites

- Ubuntu 22.04 (bare metal, VM, or WSL2 with systemd)
- Packages: `iproute2`, `iptables`, `conntrack`, `netcat-openbsd`, `curl`
- Root or sudo access
- Recommended: run in a throwaway VM — you will modify kernel parameters

```bash
sudo apt-get update && sudo apt-get install -y \
  iproute2 iptables conntrack netcat-openbsd curl dnsutils
```

## Debugging Methodology Alignment

These labs reinforce the **5-Layer Diagnostic Model** from `07-debugging-playbooks/00-debugging-methodology.md`:
- Lab 1A targets L2 (bridge/MAC) and L3 (IP routing between namespaces)
- Lab 1B targets L3/L4 (conntrack is a stateful L4 component implemented in netfilter)
- Lab 1C targets L3 (iptables firewall rules blocking IP/TCP flows)

---

## Lab 1A: Build a Network Namespace Bridge from Scratch

**Objective:** Recreate the exact sequence Docker uses to give a container its network interface — two namespaces communicating via veth pairs attached to a Linux bridge.

---

### Setup: Create the Bridge and Namespaces

```bash
# Create two network namespaces (analogous to two containers)
sudo ip netns add ns1
sudo ip netns add ns2

# Verify creation
ip netns list
# Expected:
# ns2 (id: 1)
# ns1 (id: 0)

# Create a Linux bridge (analogous to docker0)
sudo ip link add br0 type bridge
sudo ip link set br0 up
sudo ip addr add 10.0.0.1/24 dev br0

# Verify bridge is up
ip addr show br0
# Expected:
# 3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
#     inet 10.0.0.1/24 scope global br0
```

### Setup: Create veth Pairs and Attach to Bridge

```bash
# Create veth pair for ns1: veth1 (host side) <--> veth1-ns (ns side)
sudo ip link add veth1 type veth peer name veth1-ns

# Create veth pair for ns2: veth2 (host side) <--> veth2-ns (ns side)
sudo ip link add veth2 type veth peer name veth2-ns

# Move namespace-side ends into their namespaces
sudo ip link set veth1-ns netns ns1
sudo ip link set veth2-ns netns ns2

# Attach host-side ends to the bridge
sudo ip link set veth1 master br0
sudo ip link set veth2 master br0

# Bring up host-side veth interfaces
sudo ip link set veth1 up
sudo ip link set veth2 up

# Assign IPs inside each namespace
sudo ip netns exec ns1 ip addr add 10.0.0.10/24 dev veth1-ns
sudo ip netns exec ns1 ip link set veth1-ns up
sudo ip netns exec ns1 ip link set lo up

sudo ip netns exec ns2 ip addr add 10.0.0.20/24 dev veth2-ns
sudo ip netns exec ns2 ip link set veth2-ns up
sudo ip netns exec ns2 ip link set lo up
```

### Verify: Connectivity Between Namespaces

```bash
# Ping ns2 from ns1
sudo ip netns exec ns1 ping -c 3 10.0.0.20
# Expected:
# PING 10.0.0.20 (10.0.0.20) 56(84) bytes of data.
# 64 bytes from 10.0.0.20: icmp_seq=1 ttl=64 time=0.082 ms
# 64 bytes from 10.0.0.20: icmp_seq=2 ttl=64 time=0.071 ms
# 3 packets transmitted, 3 received, 0% packet loss

# Ping the bridge gateway from ns1
sudo ip netns exec ns1 ping -c 2 10.0.0.1
# Expected: replies from 10.0.0.1

# Inspect ARP table inside ns1 — shows ns2's MAC was learned
sudo ip netns exec ns1 ip neigh show
# Expected:
# 10.0.0.20 dev veth1-ns lladdr <mac-of-veth2-ns> REACHABLE
```

### Setup: Enable Internet Access (NAT/MASQUERADE)

```bash
# Enable IP forwarding on the host
sudo sysctl -w net.ipv4.ip_forward=1

# Add default route in namespaces pointing to the bridge
sudo ip netns exec ns1 ip route add default via 10.0.0.1
sudo ip netns exec ns2 ip route add default via 10.0.0.1

# MASQUERADE outbound traffic from the 10.0.0.0/24 namespace network
# Replace eth0 with your host's outbound interface name
HOST_IFACE=$(ip route | awk '/default/ {print $5; exit}')
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o "$HOST_IFACE" -j MASQUERADE

# Test internet access from a namespace
sudo ip netns exec ns1 curl -s --max-time 5 http://1.1.1.1 | head -c 100
# Expected: HTML or a response body (Cloudflare's page)
```

### Inspect the Bridge Forwarding Database

```bash
# Show which MACs are known on which bridge ports — this is L2 forwarding
bridge fdb show dev br0
# Expected:
# 33:33:00:00:00:01 dev veth1 self permanent
# 33:33:00:00:00:01 dev veth2 self permanent
# <mac-veth1-ns> dev veth1 master br0
# <mac-veth2-ns> dev veth2 master br0

# Show the full topology
ip link show type bridge
ip link show master br0
```

### Cleanup

```bash
sudo ip netns del ns1
sudo ip netns del ns2
sudo ip link del br0        # deletes br0; veth1/veth2 were already gone with br0
sudo iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o "$HOST_IFACE" -j MASQUERADE
sudo sysctl -w net.ipv4.ip_forward=0
```

**Key Takeaway:** Docker/K8s container networking is not magic. It is: `ip netns` + `veth pair` + `bridge` + `iptables MASQUERADE`. Every container gets one veth end; the other end lives on the host attached to a bridge. The bridge does L2 forwarding; iptables does the NAT for outbound traffic.

---

## Lab 1B: Reproduce and Fix Conntrack Table Exhaustion

**Objective:** Trigger the "nf_conntrack: table full, dropping packet" error that silently kills production traffic during high-connection events (load tests, bot traffic, SYN floods).

---

### Background

The Linux connection tracking table (`nf_conntrack`) stores state for every active connection. When full, new connections are silently dropped — no RST, no ICMP, just silence. This is L3/L4 invisible failure.

### Setup: Verify Current Conntrack State

```bash
# Check current maximum
cat /proc/sys/net/netfilter/nf_conntrack_max
# Typical production value: 131072 or higher

# Check current count
conntrack -L 2>/dev/null | wc -l
# Or via /proc
cat /proc/sys/net/netfilter/nf_conntrack_count
```

### The Break: Reduce nf_conntrack_max to 100

```bash
# Save the original value
ORIGINAL_MAX=$(cat /proc/sys/net/netfilter/nf_conntrack_max)
echo "Original max: $ORIGINAL_MAX"

# Reduce to 100 to make exhaustion easy to trigger
sudo sysctl -w net.netfilter.nf_conntrack_max=100

# Verify
cat /proc/sys/net/netfilter/nf_conntrack_max
# Expected: 100
```

### Symptoms: Flood Connections to Trigger Exhaustion

```bash
# Terminal 1: Start a simple HTTP server
python3 -m http.server 8888 &
HTTP_PID=$!

# Terminal 2: Flood with connections using a loop
# Each curl opens a new TCP connection and creates a conntrack entry
for i in $(seq 1 200); do
    curl -s --max-time 1 http://127.0.0.1:8888/ > /dev/null &
done
wait

# Check dmesg for the kernel message
dmesg | grep -i conntrack | tail -20
# Expected (once table is full):
# [12345.678] nf_conntrack: nf_conntrack: table full, dropping packet
# [12345.679] nf_conntrack: nf_conntrack: table full, dropping packet
```

### Diagnosis: Confirm Table Exhaustion

```bash
# Step 1: Current count vs max
echo "Current count: $(cat /proc/sys/net/netfilter/nf_conntrack_count)"
echo "Max allowed:   $(cat /proc/sys/net/netfilter/nf_conntrack_max)"
# If count >= max: confirmed exhaustion

# Step 2: List conntrack entries — shows TCP states piling up
conntrack -L 2>/dev/null | head -20
# Expected (lots of TIME_WAIT entries consuming slots):
# tcp      6 56 TIME_WAIT src=127.0.0.1 dst=127.0.0.1 sport=54321 dport=8888 ...
# tcp      6 56 TIME_WAIT src=127.0.0.1 dst=127.0.0.1 sport=54322 dport=8888 ...

# Step 3: Count TIME_WAIT entries specifically
conntrack -L 2>/dev/null | grep TIME_WAIT | wc -l

# Step 4: nstat shows kernel-level drop counters
nstat -az | grep -i conntrack
# Expected:
# NfConntrackTableFull             47                0.0
# NfConntrackFull                   47                0.0

# Step 5: conntrack stats per CPU
conntrack -S
# Expected:
# cpu=0   found=0 invalid=0 insert=100 insert_failed=47 drop=47 ...
# The insert_failed and drop fields confirm exhaustion
```

### Root Cause

`nf_conntrack_max=100` is far too low. Each TCP connection occupies one conntrack entry and stays in `TIME_WAIT` for 120 seconds by default (`nf_conntrack_tcp_timeout_time_wait`). Under any realistic load, the 100-entry table fills immediately.

### Fix

```bash
# Fix 1: Increase the max to a production-appropriate value
# Rule of thumb: (RAM in GB) * 32768 minimum; for 8GB host: 262144
sudo sysctl -w net.netfilter.nf_conntrack_max=131072

# Fix 2: Reduce TIME_WAIT timeout from 120s to 30s
# TIME_WAIT connections hold table slots but the session is already done
sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30

# Fix 3: For services that don't need per-connection tracking (internal LB)
# Consider NOTRACK rules to exempt high-volume internal traffic
# sudo iptables -t raw -A PREROUTING -s 10.0.0.0/8 -j NOTRACK
# sudo iptables -t raw -A OUTPUT -d 10.0.0.0/8 -j NOTRACK

# Make persistent (Ubuntu/Debian)
cat <<EOF | sudo tee /etc/sysctl.d/99-conntrack.conf
net.netfilter.nf_conntrack_max = 131072
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
net.netfilter.nf_conntrack_tcp_timeout_established = 300
EOF

sudo sysctl --system
```

### Verify

```bash
# Confirm new max is applied
cat /proc/sys/net/netfilter/nf_conntrack_max
# Expected: 131072

# Re-run the flood test — should no longer see drops
for i in $(seq 1 200); do
    curl -s --max-time 1 http://127.0.0.1:8888/ > /dev/null &
done
wait

dmesg | grep -i "table full" | tail -5
# Expected: no new messages

nstat -az | grep NfConntrackFull
# Expected: count should not have increased
```

### Cleanup

```bash
kill $HTTP_PID 2>/dev/null
sudo sysctl -w net.netfilter.nf_conntrack_max=$ORIGINAL_MAX
```

**Key Takeaway:** Conntrack exhaustion is silent — connections drop with no error message to the application. Always monitor `nf_conntrack_count` vs `nf_conntrack_max` as a metric. In production: alert at 80% utilization. The fix is always `nf_conntrack_max` + `timeout_time_wait` reduction.

---

## Lab 1C: Debug iptables Blocking with TRACE

**Objective:** Use the iptables TRACE target to identify exactly which rule in which chain is dropping traffic — without reading every iptables rule manually.

---

### Setup: Start a Test Service

```bash
# Start a simple HTTP server on port 8080
python3 -m http.server 8080 &
HTTP_PID=$!

# Confirm it's listening
ss -tnlp | grep 8080
# Expected:
# LISTEN 0 5 0.0.0.0:8080 0.0.0.0:* users:(("python3",pid=XXXX))

# Confirm it works before the break
curl -s http://127.0.0.1:8080
# Expected: HTML directory listing
```

### The Break: Add a DROP Rule

```bash
# Add a rule to INPUT chain that drops all TCP traffic to port 8080
sudo iptables -I INPUT -p tcp --dport 8080 -j DROP

# Verify the rule was added (position 1 in INPUT)
sudo iptables -L INPUT -n -v --line-numbers
# Expected:
# Chain INPUT (policy ACCEPT)
# num  pkts bytes target  prot opt in out source      destination
# 1       0     0 DROP    tcp  --  *  *   0.0.0.0/0   0.0.0.0/0   tcp dpt:8080
```

### Symptoms

```bash
# This will hang (kernel drops packet, no RST sent)
curl --max-time 5 http://127.0.0.1:8080
# Expected output after 5 seconds:
# curl: (28) Connection timed out after 5001 milliseconds

# NOT a "connection refused" — that would mean the port is closed
# A timeout means something is silently dropping the packet
```

### Diagnosis: Identify the Blocking Rule with TRACE

The TRACE target logs every rule that matches a packet as it traverses the netfilter chains. Load the `nf_log_ipv4` module first.

```bash
# Step 1: Load the logging module
sudo modprobe nf_log_ipv4
sudo sysctl -w net.netfilter.nf_log.2=nf_log_ipv4

# Step 2: Add TRACE rules in the raw table (before conntrack, before INPUT)
# PREROUTING catches incoming packets
sudo iptables -t raw -I PREROUTING -p tcp --dport 8080 -j TRACE
# OUTPUT catches locally-generated packets (for loopback tracing)
sudo iptables -t raw -I OUTPUT -p tcp --dport 8080 -j TRACE

# Step 3: Trigger the blocked request
curl --max-time 3 http://127.0.0.1:8080 &

# Step 4: Read kernel trace logs
dmesg | grep "TRACE:" | tail -30
# Expected output (one line per rule the packet hits):
# [1234.5] TRACE: raw:PREROUTING:rule:1 IN=lo OUT= SRC=127.0.0.1 DST=127.0.0.1 PROTO=TCP SPT=54321 DPT=8080
# [1234.5] TRACE: raw:PREROUTING:policy:2 IN=lo ...
# [1234.5] TRACE: mangle:PREROUTING:policy:1 IN=lo ...
# [1234.5] TRACE: nat:PREROUTING:policy:1 IN=lo ...
# [1234.5] TRACE: filter:INPUT:rule:1 IN=lo ...   <--- STOPS HERE = this rule drops it

# The trace STOPS at the rule that matches and DROPs. The last line tells you:
# Table: filter
# Chain: INPUT
# Rule:  1  (rule number 1 in the INPUT chain)
```

```bash
# Step 5: Confirm which rule is rule:1 in filter:INPUT
sudo iptables -t filter -L INPUT -n -v --line-numbers
# Expected:
# 1    42   2520 DROP  tcp -- * * 0.0.0.0/0 0.0.0.0/0 tcp dpt:8080
# Rule 1 in filter INPUT = the DROP rule we added. Confirmed.
```

### Root Cause

An `iptables -I INPUT -p tcp --dport 8080 -j DROP` rule was inserted at position 1, which intercepts all TCP packets destined for port 8080 before any ACCEPT rule can match.

### Fix

```bash
# Remove the specific DROP rule by number
sudo iptables -D INPUT 1

# Or remove it by specification (safer, more explicit)
sudo iptables -D INPUT -p tcp --dport 8080 -j DROP

# Verify it's gone
sudo iptables -L INPUT -n -v --line-numbers
# Expected: no rule for dpt:8080

# Clean up TRACE rules
sudo iptables -t raw -D PREROUTING -p tcp --dport 8080 -j TRACE
sudo iptables -t raw -D OUTPUT -p tcp --dport 8080 -j TRACE
```

### Verify

```bash
curl -s http://127.0.0.1:8080
# Expected: HTML directory listing (no timeout)
```

### Cleanup

```bash
kill $HTTP_PID 2>/dev/null
```

**Key Takeaway:** When traffic hangs (timeout, not refused), the culprit is a DROP rule — not a closed port. Use `iptables -t raw -I PREROUTING -j TRACE` + `dmesg` to find the exact rule. The trace stops at the matching rule. Rule number in the trace output maps directly to `iptables -L --line-numbers`.

---

## Summary: Linux Networking Lab Checklist

| Lab | Core Skill | Production Scenario |
|-----|-----------|---------------------|
| 1A  | netns + veth + bridge + NAT | Understand Docker/K8s networking primitives |
| 1B  | conntrack exhaustion diagnosis | Silent connection drops under load |
| 1C  | iptables TRACE debugging | Finding invisible DROP rules in a complex ruleset |

## Interview Discussion Points

1. "Walk me through how a Docker container gets its IP address." — Answer is Lab 1A.
2. "We see random connection failures under load, no error in the app log." — Think conntrack.
3. "Traffic from one service to another is silently dropping, curl hangs but doesn't refuse." — TRACE + iptables inspection.
