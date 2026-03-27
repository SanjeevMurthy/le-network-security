# Linux Networking — Interview Q&A

## Table of Contents

- [Quick Reference](#quick-reference)
- [Q: Your proxy server shows "nf_conntrack: table full, dropping packet" in dmesg. The table is set to 262,144 entries. Walk through full diagnosis and remediation. Why is simply increasing nf_conntrack_max insufficient?](#q-your-proxy-server-shows-nf_conntrack-table-full-dropping-packet-in-dmesg-the-table-is-set-to-262144-entries-walk-through-full-diagnosis-and-remediation-why-is-simply-increasing-nf_conntrack_max-insufficient)
- [Q: A server has thousands of sockets in CLOSE_WAIT. The application team says the network dropped the connection. Prove this is an application bug and explain the TCP state machine.](#q-a-server-has-thousands-of-sockets-in-close_wait-the-application-team-says-the-network-dropped-the-connection-prove-this-is-an-application-bug-and-explain-the-tcp-state-machine)
- [Q: A service has 2% of requests timing out. You discover ICMP is blocked on the path and the path MTU is 1400. Explain the MTU black hole and all three ways to fix it.](#q-a-service-has-2-of-requests-timing-out-you-discover-icmp-is-blocked-on-the-path-and-the-path-mtu-is-1400-explain-the-mtu-black-hole-and-all-three-ways-to-fix-it)
- [Q: Your production iptables firewall has 15,000 rules. A newly added ACCEPT rule for port 8443 is not working. How does rule ordering cause this and how do you fix it structurally?](#q-your-production-iptables-firewall-has-15000-rules-a-newly-added-accept-rule-for-port-8443-is-not-working-how-does-rule-ordering-cause-this-and-how-do-you-fix-it-structurally)
- [Q: During a DDoS attack, SYN floods are overwhelming a public-facing service. How does the attack work and what are the three layers of defense?](#q-during-a-ddos-attack-syn-floods-are-overwhelming-a-public-facing-service-how-does-the-attack-work-and-what-are-the-three-layers-of-defense)
- [Q: A server has 50,000 sockets in TIME_WAIT. Is this a problem? What causes it and how do you reduce it without breaking things?](#q-a-server-has-50000-sockets-in-time_wait-is-this-a-problem-what-causes-it-and-how-do-you-reduce-it-without-breaking-things)
- [Q: How does kube-proxy iptables mode scale? When does it break?](#q-how-does-kube-proxy-iptables-mode-scale-when-does-it-break)
- [Q: eBPF vs iptables for Kubernetes networking — what's the trade-off?](#q-ebpf-vs-iptables-for-kubernetes-networking-whats-the-trade-off)
- [Q: Describe the complete path of a packet from NIC to a userspace socket buffer.](#q-describe-the-complete-path-of-a-packet-from-nic-to-a-userspace-socket-buffer)
- [Key Takeaways](#key-takeaways)

---

## Quick Reference

This section tests your mastery of the Linux kernel networking stack — the actual implementation layer where everything from Kubernetes networking to cloud VPCs bottoms out. Interviewers at Staff level expect you to reason from symptoms to kernel subsystems: "this error appears in CLOSE_WAIT" maps to a specific TCP state machine transition and a specific bug class. Every Q&A here maps to a production incident pattern. The source material for this section is `05_Advanced_Interview_QA.md` — extended with Kubernetes-specific scenarios. See `02-linux-networking/` for deeper reading.

---

## Q: Your proxy server shows "nf_conntrack: table full, dropping packet" in dmesg. The table is set to 262,144 entries. Walk through full diagnosis and remediation. Why is simply increasing nf_conntrack_max insufficient?

**What the interviewer is testing:** Whether you understand conntrack as a stateful resource with memory costs and structural alternatives.

**Model Answer:**

Conntrack maintains a hash table of all tracked connections. When it's full, new connections are dropped at the kernel level — the SYN never reaches the application.

**Immediate diagnosis:**
```bash
# Current count vs max
sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max

# Per-CPU drop statistics — look for "drop" and "insert_failed"
conntrack -S

# Breakdown by state — usually dominated by TIME_WAIT
conntrack -L -o extended | awk '{print $4}' | sort | uniq -c | sort -rn

# Breakdown by protocol — DNS UDP entries with long timeouts are common
conntrack -L -o extended | awk '{print $3}' | sort | uniq -c | sort -rn
```

**Root cause analysis — the three common scenarios:**

1. TIME_WAIT entries with 120s timeout: at 10,000 new connections/sec, the table fills to 1.2M entries from TIME_WAIT alone.
2. DNS UDP entries: default `nf_conntrack_udp_timeout=30s`. At 50K DNS queries/sec, this fills quickly.
3. Legitimate sustained load: a NAT gateway for a large fleet genuinely needs millions of entries.

**Remediation sequence:**

Step 1 — Reduce timeouts (lowest risk):
```bash
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30    # default 120
sysctl -w net.netfilter.nf_conntrack_udp_timeout=10              # default 30 (DNS)
sysctl -w net.netfilter.nf_conntrack_udp_timeout_stream=60       # streaming UDP
```

Step 2 — NOTRACK for stateless traffic:
```bash
# Health check endpoints don't need state tracking
iptables -t raw -A PREROUTING -p tcp --dport 8080 -j NOTRACK
iptables -t raw -A OUTPUT -p tcp --sport 8080 -j NOTRACK
# nftables equivalent:
nft add rule ip raw prerouting tcp dport 8080 notrack
```

Step 3 — Fix hash table sizing (critical, often missed):
```bash
# Ratio: buckets should be nf_conntrack_max / 4
# Default: 16384 buckets, 262144 max = 16 entries per bucket
# At high fill: hash chain traversal degrades O(n)
# Set at module load time:
echo "options nf_conntrack hashsize=131072" >> /etc/modprobe.d/nf_conntrack.conf
```

**Why simply increasing nf_conntrack_max is insufficient:**
- Linear memory cost: 262,144 entries × 320 bytes = 80MB. At 2M entries: 640MB of kernel memory (non-swappable).
- Hash table degradation: without proportional bucket scaling, lookups degrade as chain length grows.
- Root cause untouched: if short-lived connections are creating entries faster than they expire, a larger table just delays the next exhaustion.

The structural fix is NOTRACK for traffic that doesn't need stateful tracking, or migrating to Cilium which replaces conntrack with eBPF-based connection tracking that scales to 100M+ entries with better cache efficiency.

**Follow-up Questions:**
- How does conntrack interact with Kubernetes kube-proxy iptables rules?
- What is a conntrack zone and how do you use it for tenant isolation?
- How does Cilium avoid conntrack for pod-to-service traffic?

**Connecting Points:**
- `02-linux-networking/` for nf_conntrack sysctl reference
- `10-interview-prep/04-kubernetes-networking-qa.md` for kube-proxy and conntrack interaction

---

## Q: A server has thousands of sockets in CLOSE_WAIT. The application team says the network dropped the connection. Prove this is an application bug and explain the TCP state machine.

**What the interviewer is testing:** Whether you can use a state machine to definitively assign blame.

**Model Answer:**

CLOSE_WAIT is definitively an application bug. The TCP state machine makes this unambiguous:

```
Client (active closer):     Server (passive closer):
FIN_WAIT_1  ----FIN---->   CLOSE_WAIT   ← kernel has received FIN and ACKed it
FIN_WAIT_2  <---ACK-----   CLOSE_WAIT   ← application MUST now call close()
TIME_WAIT   <---FIN-----   LAST_ACK     ← only after application calls close()
(closed)    ----ACK---->   (closed)
```

CLOSE_WAIT means: the remote peer sent a FIN, the LOCAL KERNEL acknowledged it, but the LOCAL APPLICATION has not called `close()` on the file descriptor. The kernel cannot close the socket on behalf of the application — it would terminate an active I/O operation.

There is no kernel timeout for CLOSE_WAIT. It persists indefinitely until the application closes the FD or the process dies.

**Diagnosis — proving it's application-side:**
```bash
# Find all CLOSE_WAIT sockets and which process owns them
ss -tanp state close-wait

# Count CLOSE_WAIT per process
ss -tanp state close-wait | awk '{print $NF}' | sort | uniq -c | sort -rn

# Check file descriptor count for the suspect process
ls /proc/<pid>/fd | wc -l

# Check FD limits
cat /proc/<pid>/limits | grep "open files"

# Watch FD count growing over time (confirms leak)
watch -n5 'ls /proc/<pid>/fd | wc -l'
```

**Common causes:**
- Connection pool libraries that don't handle remote-initiated close (the pool never removes dead connections)
- Go HTTP client with unclosed response bodies (`resp.Body.Close()` missing)
- Java apps with keepalive-enabled HTTP clients where the server closed the connection

**Prevention:**
```bash
# Set SO_KEEPALIVE with aggressive intervals on the server socket
sysctl net.ipv4.tcp_keepalive_time=60      # probe after 60s idle (default: 7200)
sysctl net.ipv4.tcp_keepalive_intvl=10     # probe every 10s
sysctl net.ipv4.tcp_keepalive_probes=6     # 6 probes before RST
# When the dead client's side is detected, the kernel sends RST
# This triggers an error in the application's next read/write, forcing close()
```

**Follow-up Questions:**
- Can `ss -K` close CLOSE_WAIT sockets? What are the safety concerns?
- What's the difference between FIN_WAIT_2 and CLOSE_WAIT in terms of kernel timeout behavior?
- How would you alert on CLOSE_WAIT accumulation in Prometheus?

**Connecting Points:**
- `10-interview-prep/01-networking-fundamentals-qa.md` for TCP state machine overview
- `02-linux-networking/` for socket options SO_KEEPALIVE, SO_RCVTIMEO

---

## Q: A service has 2% of requests timing out. You discover ICMP is blocked on the path and the path MTU is 1400. Explain the MTU black hole and all three ways to fix it.

**What the interviewer is testing:** Whether you understand PMTUD dependency on ICMP and the MSS clamping workaround.

**Model Answer:**

Path MTU Discovery (PMTUD) requires ICMP Type 3, Code 4 (Fragmentation Needed) to signal "reduce your packet size." When firewalls block all ICMP, the sender never learns the path MTU. It keeps sending 1500-byte packets; the router drops them silently (because DF bit is set in all modern TCP stacks); the connection hangs.

The 2% pattern: only clients behind PPPoE (1492), VPN tunnels (1400-1460), or mobile GTP (~1440) are affected — the majority of internet paths have 1500 MTU.

**Detection:**
```bash
# Test with explicit MTU
ping -M do -s 1452 <remote-ip>    # 1452 + 28 header = 1480 bytes
ping -M do -s 1372 <remote-ip>    # 1372 + 28 = 1400 bytes

# Check server's PMTU cache for specific client
ip route get <client-ip>           # shows "mtu 1400" if PMTUD worked
# If no mtu field: server never received ICMP Fragmentation Needed

# Capture ICMP messages arriving at server
tcpdump -ni eth0 'icmp and icmp[icmptype] == 3 and icmp[icmpcode] == 4'

# nstat for PLPMTUD failures
nstat -a | grep MTUPFail
```

**Fix 1 — MSS Clamping (most reliable, works even when ICMP is blocked):**
```bash
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
# Or hardcode: (path MTU 1400) - (IP header 20) - (TCP header 20) = 1360
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1360
```
MSS clamping intercepts the TCP handshake and rewrites the MSS option, ensuring both sides agree to a segment size that fits the path.

**Fix 2 — Allow ICMP through all firewalls:**
```bash
iptables -A INPUT  -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -A OUTPUT -p icmp --icmp-type fragmentation-needed -j ACCEPT
ip6tables -A INPUT  -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
```
This is the correct long-term fix but requires coordination across all firewall teams.

**Fix 3 — PLPMTUD (Packetization Layer PMTU Discovery, kernel 5.1+):**
```bash
sysctl -w net.ipv4.tcp_mtu_probing=1    # enable PLPMTUD
sysctl -w net.ipv4.tcp_base_mss=1024    # starting probe size
```
PLPMTUD probes at the TCP layer without ICMP. When a TCP segment is not ACKed, it reduces the size and retries. Doesn't require any middlebox changes.

**VXLAN complication:** VXLAN adds 50 bytes overhead. If underlay MTU is 1500, inner MTU must be 1450. With a VPN reducing underlay to 1400, inner MTU becomes 1350. Always document and verify MTU budgets when stacking encapsulations.

**Follow-up Questions:**
- Why doesn't MSS clamping help QUIC or UDP applications?
- How do you verify MSS clamping is working on a specific connection?
- What is the difference between `tcp_mtu_probing=1` and `tcp_mtu_probing=2`?

**Connecting Points:**
- `01-networking-fundamentals/` for PMTUD and DF bit
- `04-kubernetes-networking/` for VXLAN/Geneve MTU overhead in CNI plugins

---

## Q: Your production iptables firewall has 15,000 rules. A newly added ACCEPT rule for port 8443 is not working. How does rule ordering cause this and how do you fix it structurally?

**What the interviewer is testing:** First-match semantics, use of counters for debugging, and awareness of performance implications.

**Model Answer:**

iptables uses **first-match-wins** evaluation. Rules are evaluated in sequence. If a REJECT or DROP rule matching port 8443 (or a catch-all rule) appears before line 12,000 where the ACCEPT was added, the ACCEPT is never reached.

**Diagnosis — use counters to find the matching rule:**
```bash
# Zero all counters, send test traffic, see which rule incremented
iptables -Z INPUT                              # zero counters
curl -k https://<server>:8443/ &              # send test traffic
iptables -L INPUT -n -v --line-numbers | awk '$1 > 0 {print}'    # rules with hits

# Search for any existing rule matching 8443
iptables -L INPUT -n -v --line-numbers | grep 8443

# Search for broad DROP/REJECT rules that match all TCP before your ACCEPT
iptables -L INPUT -n -v --line-numbers | grep -E "REJECT|DROP" | head -20
```

**Common ordering bugs in large rulesets:**
1. Catch-all REJECT before specific ACCEPT: `-A INPUT -p tcp -j REJECT` at line 5,000 blocks all TCP including 8443
2. Chain jump: traffic jumps to a sub-chain that returns REJECT before reaching the ACCEPT
3. Table precedence: a DNAT rule in `nat` table changes the destination port, so the `filter` table rule doesn't match

**Immediate fix:**
```bash
# Insert the rule before the conflicting DROP (at a specific line number)
iptables -I INPUT <line-before-drop> -p tcp --dport 8443 -s <trusted-cidr> -j ACCEPT
```

**Structural fix for 15,000-rule rulesets:**

The core problem is O(n) evaluation. At 100K pps × 15,000 rules = 1.5B rule evaluations/second. This is the primary reason for migrating to nftables or eBPF.

```bash
# Use ipset for large source/destination lists (O(1) lookup instead of O(n))
ipset create trusted-sources hash:ip
ipset add trusted-sources 10.0.1.0/24
iptables -I INPUT -m set --match-set trusted-sources src -p tcp --dport 8443 -j ACCEPT

# Organize rules into sub-chains by function
iptables -N WEB_SERVICES
iptables -A WEB_SERVICES -p tcp --dport 443  -j ACCEPT
iptables -A WEB_SERVICES -p tcp --dport 8443 -j ACCEPT
iptables -A INPUT -j WEB_SERVICES

# Migration to nftables (uses hash tables, interval trees — O(1) or O(log n))
nft add table inet filter
nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
nft add rule inet filter input tcp dport { 443, 8443 } accept
```

**Performance impact:** nftables rule sets that would be 15,000 iptables rules often collapse to 50-100 nftables rules using sets. Rule evaluation goes from milliseconds to microseconds per packet.

**Follow-up Questions:**
- How do you use the `ESTABLISHED,RELATED` state match to short-circuit rule traversal?
- What is the nftables `verdict map` and how does it replace chains of rules?
- How does Cilium eBPF policy enforcement compare to iptables at 10K rules?

**Connecting Points:**
- `02-linux-networking/` for iptables/nftables architecture
- `10-interview-prep/09-exotic-deep-dive-qa.md` for eBPF-based policy

---

## Q: During a DDoS attack, SYN floods are overwhelming a public-facing service. How does the attack work and what are the three layers of defense?

**What the interviewer is testing:** Whether you understand the attack mechanism at the kernel level and can describe defenses with specificity.

**Model Answer:**

A SYN flood exploits the TCP three-way handshake. The attacker sends SYN packets with spoofed source IPs. The server allocates a half-open connection entry in the SYN backlog queue for each SYN, sends SYN-ACK, and waits for ACK (which never arrives). The SYN backlog fills; new legitimate SYN packets are dropped.

**Attack math:** Default SYN backlog: `net.ipv4.tcp_max_syn_backlog=512`. At 10,000 spoofed SYNs/sec, the backlog fills in 50ms.

**Defense Layer 1 — SYN Cookies (kernel-level, default on Linux):**
```bash
sysctl net.ipv4.tcp_syncookies=1    # default: enabled

# How it works: instead of allocating backlog entry, the server encodes
# connection state in the SYN-ACK's sequence number (a cryptographic cookie).
# If the client is real, its ACK returns the cookie; the connection is established.
# If fake (spoofed IP), no ACK ever arrives — no backlog entry was wasted.

# When syncookies kicks in (not immediate):
sysctl net.ipv4.tcp_max_syn_backlog    # syncookies activates when backlog is full
```

**Defense Layer 2 — XDP/eBPF drop before kernel stack:**
```bash
# XDP program that rate-limits SYNs per source IP using an LRU hash map
# Can process 24M pps before the sk_buff is allocated — zero conntrack cost

# Load an XDP SYN flood protection program
ip link set eth0 xdp obj syn_flood_xdp.o sec xdp

# Check drop statistics
bpftool map dump name syn_stats
```

This is what Cloudflare uses: XDP drops DDoS traffic at line rate on the NIC before anything else processes it.

**Defense Layer 3 — Upstream rate limiting and BGP blackhole:**
```bash
# BGP RTBH (Remote Triggered Black Hole): announce the target IP with
# community 65535:666 to upstream providers — they drop traffic at their edge
# (source traffic scrubbing) before it reaches your network.

# Or: use a DDoS scrubbing service (AWS Shield Advanced, Cloudflare Magic Transit)
# that absorbs the attack traffic and forwards only clean traffic
```

**Additional tuning:**
```bash
sysctl -w net.ipv4.tcp_max_syn_backlog=65536     # larger backlog
sysctl -w net.core.somaxconn=65536               # application accept queue
sysctl -w net.ipv4.tcp_syn_retries=2             # reduce SYN-ACK retransmissions to attacker
```

**Follow-up Questions:**
- What is the performance cost of SYN cookies (no TCP options can be negotiated)?
- How do you distinguish SYN flood from a legitimate traffic spike?
- What is amplification DDoS and why is SYN flood not an amplification attack?

**Connecting Points:**
- `05-network-security/` for DDoS mitigation architecture
- `10-interview-prep/05-network-security-qa.md` for SYN cookie mechanics

---

## Q: A server has 50,000 sockets in TIME_WAIT. Is this a problem? What causes it and how do you reduce it without breaking things?

**What the interviewer is testing:** Whether you know that TIME_WAIT exists for a reason, and which "fixes" are dangerous.

**Model Answer:**

50,000 TIME_WAIT sockets is likely **not a problem** on a modern Linux server. TIME_WAIT:
- Consumes ~1KB of kernel memory each = 50MB total (negligible)
- Does NOT consume a port for incoming connections (TIME_WAIT sockets are not in the listen queue)
- Does consume a local port for outbound connections — this can cause EADDRNOTAVAIL under high outbound connection churn

**Why TIME_WAIT exists:** To prevent old duplicate packets from corrupting new connections. If a new connection reuses the same 4-tuple as a recently closed one, and an old duplicate packet arrives, the kernel might interpret it as valid data. TIME_WAIT's 2×MSL (120 seconds) ensures old packets have expired from the network.

**Diagnosis:**
```bash
ss -s | grep "time wait"       # count
ss -tn state time-wait | head  # which destinations
# If all TIME_WAIT sockets are to the same destination:
# your service is closing connections too aggressively — use persistent connections
```

**Safe reductions:**

1. **Persistent connections / HTTP keep-alive:** The primary fix. Reduce connection churn by reusing connections. TIME_WAIT exists because connections were closed — keep them open.

2. **tcp_tw_reuse (safe for outbound only):**
```bash
sysctl -w net.ipv4.tcp_tw_reuse=1
# Allows reuse of TIME_WAIT sockets for new OUTBOUND connections IF the new
# connection's timestamp is monotonically greater (requires TCP timestamps enabled)
# This is safe: timestamps prevent old duplicate packets from confusing the new connection
```

3. **Reduce fin_timeout:**
```bash
sysctl net.ipv4.tcp_fin_timeout    # FIN_WAIT_2 timeout, not TIME_WAIT
# TIME_WAIT duration is hardcoded as 2×MSL (60s) in Linux kernel
# There's no sysctl to reduce the actual TIME_WAIT duration (by design)
```

**Dangerous: tcp_tw_recycle** — was removed from kernel 4.12 because it broke NAT. When multiple clients share one NAT IP, their timestamps don't increase monotonically, causing the kernel to silently drop SYNs.

**Follow-up Questions:**
- Why can't you set a socket option to bypass TIME_WAIT?
- How does `SO_LINGER` with l_linger=0 affect TIME_WAIT?
- What is the relationship between TIME_WAIT and port exhaustion?

**Connecting Points:**
- `01-networking-fundamentals/` for TCP state machine and MSL
- `02-linux-networking/` for socket option reference

---

## Q: How does kube-proxy iptables mode scale? When does it break?

**What the interviewer is testing:** Whether you understand that kube-proxy is an iptables generator at scale, and what O(n^2) looks like in production.

**Model Answer:**

kube-proxy in iptables mode programs DNAT rules for every Kubernetes Service and endpoint. For each Service, it:
1. Adds a rule in `KUBE-SERVICES` chain matching the ClusterIP:port
2. Jumps to a `KUBE-SVC-<hash>` chain
3. Uses probability-based load balancing across endpoints:

```
-A KUBE-SVC-<hash> -m statistic --mode random --probability 0.333 -j KUBE-SEP-<ep1>
-A KUBE-SVC-<hash> -m statistic --mode random --probability 0.500 -j KUBE-SEP-<ep2>
-A KUBE-SVC-<hash>                                                 -j KUBE-SEP-<ep3>
```

For 3 endpoints: 3 rules. For N endpoints: N rules. For S services each with N endpoints: S × N rules total.

**The scaling problem:**
- 10,000 services × 10 endpoints = 100,000 iptables rules
- Each packet traverses ALL rules in worst case (O(n) per packet)
- iptables rule update is atomic: the entire ruleset is replaced on each sync (`iptables-restore`), which takes seconds at 100K+ rules
- During the update (lock held): new connections may be dropped or incorrectly routed
- At 5,000 services, kube-proxy can take 11+ seconds to update rules — during which the cluster is partially inconsistent

**Measured breaking points (from production reports):**
- ~5,000 services: noticeable latency in iptables sync
- ~10,000 services: kube-proxy update takes 10-15 seconds; pods experience connection errors during syncs
- ~20,000 services: iptables mode becomes operationally unusable

**What breaks first:**
- Endpoint updates (pod restarts): every pod restart triggers a full iptables sync
- New service creation: adds rules for all existing endpoints immediately
- Node restarts: kube-proxy rebuilds the full ruleset from scratch — can take minutes

**Follow-up Questions:**
- How does kube-proxy IPVS mode improve on this (O(1) via hash table)?
- When does Cilium's eBPF mode eliminate kube-proxy entirely?
- What is the nftables kube-proxy backend (K8s 1.29+) and how does it scale differently?

**Connecting Points:**
- `10-interview-prep/04-kubernetes-networking-qa.md` for Cilium vs kube-proxy comparison
- `04-kubernetes-networking/` for kube-proxy internals

---

## Q: eBPF vs iptables for Kubernetes networking — what's the trade-off?

**What the interviewer is testing:** Whether you understand eBPF as a systems technology, not just a buzzword.

**Model Answer:**

The trade-off is correctness/maturity vs performance/scalability.

**iptables:**
- Mature: 20+ years of production use, bugs are known and fixed
- Stateless rules with conntrack for stateful needs
- O(n) rule evaluation — degrades linearly with rule count
- Rule update is a full replace: iptables-restore takes O(rules) time, during which other operations block
- Debugging: `iptables -L -n -v --line-numbers` is universally understood
- Limitation: no programmability, no in-kernel data structures, no BPF maps

**eBPF (Cilium):**
- eBPF programs attach to kernel hooks (TC, XDP, cgroup-bpf) and execute in the kernel JIT
- BPF maps replace iptables chains: hash maps for policy lookup (O(1)), LPM tries for CIDR matching
- Service load balancing done at cgroup socket level — traffic never leaves the CPU for local service calls
- Policy enforcement at network namespace level — no need for conntrack for pod-to-pod traffic
- Kernel version dependency: Cilium requires kernel 5.10+ for full feature set
- Debugging: harder — `bpftool prog show`, `cilium monitor`, Hubble for flow visibility

**When eBPF wins decisively:**
- >5,000 services: iptables becomes a bottleneck; eBPF hash maps scale to millions of entries
- High connection churn: eBPF avoids conntrack table exhaustion for pod-to-service traffic
- Network policy at scale: 1,000+ NetworkPolicy objects degrade iptables; eBPF handles them with BPF maps

**When iptables is acceptable:**
- Clusters under 2,000 services with stable endpoint counts
- Teams unfamiliar with eBPF debugging
- Environments where kernel version cannot be controlled (some managed K8s offerings)
- Compliance requirements that mandate iptables-auditable rulesets

**Concrete numbers:**
- kube-proxy iptables: 10,000 services = ~100,000 rules, 11s sync time
- Cilium eBPF: 10,000 services = O(1) BPF map lookups, <100ms update time

**Follow-up Questions:**
- What is the BPF verifier and why does it reject some programs?
- How does Cilium's XDP-based load balancing differ from TC-BPF?
- What kernel capabilities does Cilium require that older kernels don't have?

**Connecting Points:**
- `10-interview-prep/09-exotic-deep-dive-qa.md` for BPF verifier deep dive
- `04-kubernetes-networking/` for Cilium architecture

---

## Q: Describe the complete path of a packet from NIC to a userspace socket buffer.

**What the interviewer is testing:** Whether you understand the Linux kernel networking stack at a level that enables precise performance tuning and debugging.

**Model Answer:**

The complete path (abbreviated — see `10-interview-prep/09-exotic-deep-dive-qa.md` for full detail):

```
Wire
→ NIC PHY/MAC — FCS validation
→ DMA ring buffer — NIC writes packet data via DMA, no CPU
→ Hardware IRQ — NIC signals CPU; interrupt handler calls napi_schedule()
→ NAPI softirq (NET_RX_SOFTIRQ) — adaptive polling, budget=300 packets/run
→ [XDP hook] — before sk_buff allocation; XDP_DROP returns here
→ GRO — coalesces multiple small TCP segments into one large sk_buff
→ sk_buff allocated — kernel data structure for packet
→ [TC ingress hook] — tc-bpf programs run here
→ protocol dispatch — EtherType → ip_rcv()
→ ip_rcv() — IP header validation
→ [NF_INET_PRE_ROUTING] — iptables PREROUTING, DNAT
→ routing decision — local? forward?
→ [NF_INET_LOCAL_IN] — iptables INPUT chain
→ tcp_v4_rcv() — socket lookup in hash table
→ TCP state machine processing — sequence numbers, ACKs, congestion window
→ sk->sk_receive_queue — data copied to socket receive buffer
→ wake_up() — wakes application waiting in recv()
→ tcp_recvmsg() — copies data to userspace buffer
→ Application recv() returns
```

**Key performance knobs at each stage:**
```bash
ethtool -G eth0 rx 4096          # ring buffer size (Stage 2)
sysctl net.core.netdev_budget=600 # NAPI budget (Stage 4)
ethtool -K eth0 gro on           # GRO enable (Stage 5)
sysctl net.ipv4.tcp_rmem="4096 87380 4194304"  # socket receive buffer (Stage 12)
```

**Debugging drops at each stage:**
```bash
ethtool -S eth0 | grep rx_dropped   # ring buffer overflow (Stage 2)
cat /proc/net/softnet_stat           # col 3: time_squeezed (NAPI budget exhausted)
nstat -a | grep TcpExtRcvPruned     # socket buffer pruning (Stage 12)
```

**Follow-up Questions:**
- Where does RSS distribute packets across CPUs in this path?
- What is the difference between XDP_REDIRECT and XDP_TX?
- How does AF_XDP bypass most of this path?

**Connecting Points:**
- `10-interview-prep/09-exotic-deep-dive-qa.md` for full kernel internals
- `02-linux-networking/` for NAPI and ring buffer tuning

---

## Key Takeaways

- Conntrack table exhaustion requires three fixes: timeout reduction + NOTRACK for stateless flows + hash table bucket sizing — not just increasing the max
- CLOSE_WAIT is always the application's fault — the kernel cannot call close() for you; the TCP state machine proves this
- iptables first-match semantics make large rulesets fragile; use counters to find the conflicting rule, then migrate to nftables or eBPF for structural fix
- kube-proxy iptables mode breaks at ~5,000 services due to O(n) rule evaluation and O(rules) full-replace update time
- TIME_WAIT is expected and correct; tcp_tw_recycle was removed from the kernel because it breaks NAT — never recommend it
