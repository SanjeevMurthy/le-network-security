# Networking Fundamentals — Interview Q&A

## Quick Reference

This section tests whether you understand the protocols that everything else is built on. Interviewers use fundamentals questions to assess your mental model quality — a candidate who can trace a TCP state machine or explain ECMP hash polarization has the foundation to reason about unfamiliar systems. Every answer here should be grounded in specific numbers (port ranges, byte counts, timer values) and reference a real failure mode. See `01-networking-fundamentals/` for deeper reading.

---

## Q: Walk me through the TCP state machine. A client connects to a server and then the client crashes. What states does the server go through, and what kernel timer eventually clears the connection?

**What the interviewer is testing:** Whether you understand TCP state as a distributed system where each side maintains independent state, not shared state.

**Model Answer:**

The TCP state machine has 11 states. For this scenario, trace from the server's perspective:

Normal connection lifecycle:
```
Server: LISTEN → SYN_RCVD → ESTABLISHED
Client: SYN_SENT → ESTABLISHED
```

After client crash (no FIN sent, no RST — kernel is dead):

The server stays in ESTABLISHED. It has no way to know the client is gone because TCP requires a RST or FIN to signal closure. The kernel will eventually send keepalive probes if `SO_KEEPALIVE` is enabled:

```bash
# Keepalive parameters
sysctl net.ipv4.tcp_keepalive_time    # default 7200s (2 hours before first probe)
sysctl net.ipv4.tcp_keepalive_intvl   # default 75s between probes
sysctl net.ipv4.tcp_keepalive_probes  # default 9 probes before RST
# Total: 7200 + (75 × 9) = 7875 seconds ≈ 2.2 hours before connection cleared
```

Without keepalive, the ESTABLISHED connection persists until a data packet fails to be ACKed, triggering `tcp_retries2` (default 15) retransmissions over ~924 seconds (15+ minutes), after which the kernel sends RST and clears the socket.

**Key CLOSE_WAIT vs TIME_WAIT distinction:**
- `CLOSE_WAIT` — server received FIN from client, kernel ACKed it, application has NOT called `close()`. This is always an application bug, never a network issue.
- `TIME_WAIT` — server (or client) sent the final FIN and received ACK. Held for 2×MSL (120 seconds by default, `net.ipv4.tcp_fin_timeout=60` for FIN_WAIT_2) to prevent stale packets from corrupting new connections. TIME_WAIT is expected and correct.

Diagnosis:
```bash
ss -tanp state close-wait    # who's leaking file descriptors?
ss -s | grep "close wait"    # how many?
ls /proc/<pid>/fd | wc -l    # fd count for the suspect process
```

**Follow-up Questions:**
- Why can't the server clear CLOSE_WAIT with a timeout?
- What is `ss -K` and when is it safe to use?
- What is the difference between FIN_WAIT_1 and FIN_WAIT_2?

**Connecting Points:**
- `10-interview-prep/02-linux-networking-qa.md` Q on CLOSE_WAIT accumulation
- `02-linux-networking/` for SO_KEEPALIVE implementation details

---

## Q: BGP path selection: give me a scenario where local_pref overrides AS_PATH length. Why does BGP prioritize local_pref over AS_PATH?

**What the interviewer is testing:** Whether you understand BGP is a policy protocol, not a pure shortest-path protocol.

**Model Answer:**

BGP path selection order (first tiebreaker wins):
1. Highest **Weight** (Cisco-specific, local to router)
2. Highest **LOCAL_PREF** (affects outbound traffic, within your AS)
3. Locally originated routes
4. Shortest **AS_PATH**
5. Lowest **MED** (Multi-Exit Discriminator)
6. eBGP over iBGP
7. Lowest IGP metric to next-hop
8. Lowest router-ID

Scenario where LOCAL_PREF beats AS_PATH:

```
Your AS (AS65000) has two upstream providers:
  - Provider A (AS1) → connects to destination in 2 hops (AS1 → AS2 → dest)
    AS_PATH: [1, 2, dest_AS]
    LOCAL_PREF: 100 (default)

  - Provider B (AS3) → connects to destination in 5 hops
    AS_PATH: [3, 4, 5, 6, 7, dest_AS]
    LOCAL_PREF: 200 (set by your policy — Provider B has more bandwidth)
```

BGP selects Provider B because LOCAL_PREF 200 > 100, even though AS_PATH is 3× longer.

This is intentional design. LOCAL_PREF is set by YOUR network operators and reflects YOUR policy (cost, bandwidth, SLA). AS_PATH reflects the number of autonomous systems traversed, which is a rough proxy for distance but says nothing about link quality, cost, or your business relationship. A 2-hop path through a flaky ISP is worse than a 5-hop path through a tier-1 backbone.

Real-world use: if you have a primary provider and a backup provider, set LOCAL_PREF 200 on primary and 100 on backup. Traffic flows through the primary even if the backup has a shorter AS_PATH.

```bash
# FRRouting: set LOCAL_PREF on inbound routes from Provider A
route-map PROVIDER_A_IN permit 10
  set local-preference 200
neighbor <provider-a-ip> route-map PROVIDER_A_IN in
```

**Follow-up Questions:**
- When would you use MED instead of LOCAL_PREF?
- How does AS_PATH prepending affect inbound traffic (not outbound)?
- What is COMMUNITIES in BGP and how do cloud providers use it?

**Connecting Points:**
- `01-networking-fundamentals/` BGP section for full path selection walkthrough
- `10-interview-prep/07-cross-domain-scenarios.md` for BGP in cloud VPC contexts

---

## Q: You are getting EADDRNOTAVAIL errors on a high-throughput proxy server. How do you diagnose NAT port exhaustion and what are the structural fixes?

**What the interviewer is testing:** Whether you understand the 5-tuple and port range limits, not just that "NAT has limits."

**Model Answer:**

`EADDRNOTAVAIL` on connect() means the kernel cannot find an available (source IP, source port) pair for the 5-tuple (src_ip, src_port, dst_ip, dst_port, proto). This is NAT port exhaustion.

**The math:**
A single source IP has 65,535 ports. Subtract ephemeral port range start:
```bash
sysctl net.ipv4.ip_local_port_range    # default: 32768-60999 = 28,231 ports
```

For connections to a single destination IP:port, you have 28,231 concurrent connections per source IP. AWS NAT Gateway further limits this to 55,000 connections per destination IP:port.

**Diagnosis:**
```bash
# Check current ephemeral port usage
ss -tn | awk '{print $4}' | awk -F: '{print $NF}' | sort -n | uniq -c | sort -rn | head

# Check for EADDRNOTAVAIL in application logs
# Check kernel: TcpExtTWRecycled or TcpExtPAWSEstab failures
nstat -a | grep -E "TcpExt(TWRecycled|PAWSPassive|SynRetrans)"

# Check per-destination connection counts
ss -tn dst <specific-ip> | wc -l

# Check TIME_WAIT accumulation (consuming ports)
ss -s | grep "time wait"
```

**Structural fixes, in order of preference:**

1. **Expand ephemeral port range:**
```bash
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
# Gains ~32,000 additional ports per source IP
```

2. **Enable SO_REUSEADDR / SO_REUSEPORT:** Allows multiple sockets to share a port when the 4-tuple is unique.

3. **Add source IPs (SNAT to multiple IPs):** Each source IP gets its own port range.
```bash
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 10.0.1.1-10.0.1.10
# 10 IPs × 28,231 ports = 282,310 concurrent connections per destination
```

4. **Reduce TIME_WAIT accumulation:**
```bash
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30  # down from 120s
sysctl -w net.ipv4.tcp_tw_reuse=1  # allow reuse of TIME_WAIT sockets for outbound
```

5. **Structural: use persistent connections (HTTP/2, gRPC, connection pooling)** — reduces connection churn by reusing existing connections instead of creating new ones for each request.

**Follow-up Questions:**
- What is the difference between `tcp_tw_reuse` and `tcp_tw_recycle`, and why was recycle removed from the kernel?
- How does AWS NAT Gateway's 55K per-destination limit interact with RDS connections?
- What happens when the conntrack table is exhausted vs when the port range is exhausted?

**Connecting Points:**
- `10-interview-prep/02-linux-networking-qa.md` for conntrack table exhaustion
- `03-cloud-networking/` for AWS NAT Gateway limits

---

## Q: Explain DNS failure types. A client gets NXDOMAIN, SERVFAIL, and timeout — what causes each and how do you debug them differently?

**What the interviewer is testing:** Whether you treat DNS as a distributed system with failure modes, not a black box.

**Model Answer:**

The three DNS failure types have completely different root causes:

**NXDOMAIN (Name does not exist):**
- The authoritative server has responded definitively: this name does not exist in this zone.
- Could be: typo in the name, the record was deleted, the name was never created, or you're querying the wrong zone.
- Cached for the SOA minimum TTL (often 60-3600 seconds). A recently deleted record can cause NXDOMAIN to be cached.
```bash
dig +trace api.example.com    # see which authoritative server returns NXDOMAIN
dig SOA example.com           # check negative cache TTL (minimum field)
```

**SERVFAIL (Server failure):**
- The recursive resolver encountered an error: could not reach the authoritative server, DNSSEC validation failed, or the authoritative server returned SERVFAIL itself.
- Common causes: authoritative server down, DNSSEC chain broken (expired RRSIG), resolver configuration error.
```bash
dig api.example.com                    # returns SERVFAIL
dig +cd api.example.com                # +cd disables DNSSEC checking
# If +cd works but normal doesn't: DNSSEC validation is failing
dig +dnssec +trace api.example.com     # find where the DNSSEC chain breaks
```

**Timeout (No response):**
- The resolver sent a query and got no response within the timeout (typically 5 seconds, 3 retries).
- Causes: resolver unreachable, authoritative server firewalled, UDP 53 blocked, resolver overloaded.
- Unlike NXDOMAIN/SERVFAIL, timeouts are NOT cached (retried on next query).
```bash
# Test UDP connectivity to resolver
nc -u -z <resolver-ip> 53
# Test TCP fallback (large responses or DNSSEC force TCP)
dig +tcp api.example.com @<resolver-ip>
# Check if it's a specific resolver problem
dig api.example.com @8.8.8.8    # try alternative resolver
```

**In Kubernetes:** The conntrack UDP race condition causes silent query drops that manifest as timeouts, not NXDOMAIN or SERVFAIL:
```bash
conntrack -S | grep insert_failed    # incrementing = race condition
# Fix: NodeLocal DNSCache or single-request-reopen resolv.conf option
```

**Follow-up Questions:**
- What is the difference between an authoritative NXDOMAIN and a negative cache hit?
- Why does ndots:5 in Kubernetes amplify DNS failures?
- How does DNSSEC validation failure produce SERVFAIL?

**Connecting Points:**
- `04-kubernetes-networking/` for ndots:5 and CoreDNS
- `10-interview-prep/04-kubernetes-networking-qa.md` for DNS latency root causes

---

## Q: How does MTU black hole happen and how do you detect and fix it? A service has 2% of requests timing out only from certain client populations.

**What the interviewer is testing:** Whether you understand PMTUD and why blocking ICMP breaks TCP.

**Model Answer:**

Path MTU Discovery (PMTUD) allows TCP to discover the minimum MTU along the path. When a router needs to fragment a packet but the DF (Don't Fragment) bit is set, it should send ICMP Type 3, Code 4 ("Fragmentation Needed") back to the sender. The sender then reduces its packet size.

An MTU black hole occurs when firewalls block ICMP Type 3, Code 4. The sender never learns the path MTU, keeps sending large packets, and the intermediate router silently drops them.

The 2% pattern is the key clue — those clients traverse links with reduced MTU: PPPoE (1492 bytes), VPN tunnels (1400-1460), GTP on mobile (1440), or certain ISP links.

**Detection:**
```bash
# Test if PMTUD is working (send large packets with DF bit)
ping -M do -s 1472 <server-ip>    # 1472 + 28 header = 1500 bytes
ping -M do -s 1372 <server-ip>    # 1372 + 28 = 1400 bytes
# If 1500 fails but 1400 succeeds, the path MTU is somewhere between 1400-1500

# Check if server has any cached PMTU entries
ip route get <client-ip>
# "cache expires" with "mtu 1400" means PMTUD has learned the path MTU

# Capture ICMP Fragmentation Needed messages
tcpdump -ni eth0 'icmp and icmp[icmptype] == 3 and icmp[icmpcode] == 4'
# If you see these being received but ignored: PMTUD is working
# If you see none: black hole — the ICMP messages are blocked upstream

# Monitor PLPMTUD failures (kernel 5.1+)
nstat -a | grep -i "MTUPFail"
```

**Fix (in priority order):**

1. **MSS Clamping — most reliable:**
```bash
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
# Or set fixed value: MTU - 40 bytes for IPv4 TCP headers
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1360
```

2. **Allow ICMP Fragmentation Needed through all firewalls:**
```bash
iptables -A INPUT -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -A FORWARD -p icmp --icmp-type fragmentation-needed -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
```

3. **Enable PLPMTUD (Packetization Layer PMTU Discovery):**
```bash
sysctl -w net.ipv4.tcp_mtu_probing=1    # 0=off, 1=on only on ICMP black hole detection, 2=always
sysctl -w net.ipv4.tcp_base_mss=1024    # starting probe size
```

**Follow-up Questions:**
- How does this interact with VXLAN, which adds 50 bytes of overhead?
- Why does MSS clamping work for TCP but not for QUIC or UDP?
- How does `tcp_mtu_probing=2` differ from PMTUD?

**Connecting Points:**
- `01-networking-fundamentals/` for IP fragmentation and DF bit
- `10-interview-prep/09-exotic-deep-dive-qa.md` for TCP option negotiation details

---

## Q: Design subnets for a 3-tier VPC with Kubernetes pod CIDRs. What constraints do you have?

**What the interviewer is testing:** Whether you can do production subnetting with real constraints, not just binary math.

**Model Answer:**

A production 3-tier VPC for EKS with 3 AZs:

```
VPC: 10.0.0.0/16 (65,536 addresses)

Tier 1 — Public subnets (load balancers, NAT gateways):
  10.0.0.0/24  (AZ-a)  — 256 addresses, ~5 actually used
  10.0.1.0/24  (AZ-b)
  10.0.2.0/24  (AZ-c)

Tier 2 — Private subnets (EKS worker nodes, application):
  10.0.10.0/22 (AZ-a)  — 1,022 addresses for nodes
  10.0.14.0/22 (AZ-b)
  10.0.18.0/22 (AZ-c)

Tier 3 — Data subnets (RDS, ElastiCache):
  10.0.30.0/24 (AZ-a)  — 256 addresses, RDS is not pod-dense
  10.0.31.0/24 (AZ-b)
  10.0.32.0/24 (AZ-c)

Pod CIDR (separate from VPC in some CNI modes):
  100.64.0.0/16 — 65,536 addresses for pods (RFC 6598 shared address space)
  OR 10.1.0.0/16 within the VPC CIDR for EKS with AWS CNI
```

**Key constraints for EKS:**

With AWS VPC CNI (pods get VPC IPs):
- Each pod consumes a real VPC IP — a single t3.medium node can have ~17 pods, requiring 17 VPC IPs
- Subnet must be large enough: a /22 (1,022 usable) supports ~60 nodes × 17 pods = 1,020 pods
- Always add secondary CIDRs before running out: `aws ec2 associate-vpc-cidr-block --cidr 100.64.0.0/16`

With Cilium or Calico (pods get pod CIDR IPs, not VPC IPs):
- Pod IPs are internal to the cluster, not visible to AWS routing
- Pod CIDR must not overlap with VPC CIDR or any peered network
- Typical: `/16` for pods gives 65,536 pod IPs (sufficient for ~300 nodes × 110 pods/node theoretical max, but practical max is ~50)

**Avoid:**
- `/24` for node subnets in Kubernetes — runs out of IPs with AWS CNI at ~15 large nodes
- Overlapping with on-premises CIDRs (10.0.0.0/8 is heavily used; consider 172.16.0.0/12)
- Leaving no room for future peering (AWS peering requires non-overlapping CIDRs)

**Follow-up Questions:**
- What is WARM_IP_TARGET in AWS VPC CNI and how does it affect subnet size planning?
- How do you calculate maximum pods per node given the ENI limits?
- What is the trade-off between a large VPC CIDR and many small ones?

**Connecting Points:**
- `03-cloud-networking/` for EKS CNI modes and IP exhaustion
- `10-interview-prep/03-cloud-networking-qa.md` for EKS pod IP exhaustion Q&A

---

## Q: What is ECMP hash polarization and when does it cause problems in production?

**What the interviewer is testing:** Whether you understand that ECMP is not magic load balancing — it can create severe imbalances.

**Model Answer:**

ECMP (Equal-Cost Multi-Path) distributes traffic across multiple equal-cost paths by hashing the 5-tuple (src_ip, src_port, dst_ip, dst_port, protocol). The hash selects which path each flow uses.

**Hash polarization** occurs when multiple ECMP tiers all use the same hash algorithm with the same inputs. The first ECMP tier assigns flow X to link 1. The second ECMP tier also assigns flow X to link 1 (same hash, same inputs). All flows that take link 1 at tier 1 also take link 1 at tier 2. Result: some paths carry 0% of traffic, others carry 100%.

Concrete example in a spine-leaf topology:
```
Pod A → Leaf A → [ECMP: Spine 1, Spine 2, Spine 3] → Leaf B → Pod B

If Leaf A uses CRC32 hash and Spine 1 also uses CRC32 hash with same fields:
- Flows hashed to Spine 1 at Leaf A are ALSO hashed to Spine 1's single uplink at Spine 1
- Effective paths: Leaf A → Spine 1 → Leaf B (only this path used)
- Spine 2 and 3 see 0% of this traffic
```

**When it causes problems:**
- Large file transfers that dominate a single link (elephant flows)
- Kubernetes pod-to-pod traffic where many pods have the same dst_port (8080) causing most flows to the same hash bucket
- East-west traffic in microservices where connections are persistent (gRPC, connection pools) and the same 5-tuple lasts hours

**Detection:**
```bash
# On a router (FRRouting):
show ip route <prefix> longer-prefixes  # see ECMP paths
show interface <if> counters            # compare traffic on each ECMP member

# On Linux with ECMP routes:
ip route show 10.0.0.0/8   # should show multiple nexthops
# Monitor per-interface traffic with: ip -s link show
```

**Fixes:**
1. Different hash seeds per tier (randomize the hash function)
2. Use entropy fields: VXLAN inner packet hash, GRE key, or Geneve options to add entropy at encapsulation
3. Flowlet switching — rebalance flows on idle gaps rather than per-packet
4. For Kubernetes: Cilium uses Maglev consistent hashing which has better distribution than 5-tuple CRC

**Follow-up Questions:**
- How does VXLAN inner-packet hashing help resolve ECMP polarization?
- What is flowlet switching and which vendors support it?
- Why does Maglev hashing provide better distribution than CRC?

**Connecting Points:**
- `01-networking-fundamentals/` for ECMP configuration on Linux
- `04-kubernetes-networking/` for Cilium load balancing algorithms

---

## Q: How is Gratuitous ARP used in HA failover? What can go wrong?

**What the interviewer is testing:** Whether you understand L2 address learning and how it interacts with IP failover.

**Model Answer:**

Gratuitous ARP (GARP) is an ARP Reply sent without a corresponding ARP Request, used to update the ARP caches of all hosts in the broadcast domain. The sender announces: "IP address X is at MAC address Y" without being asked.

**HA failover use case:**

When a VIP (Virtual IP) moves from the primary to the backup node, all hosts on the network still have the primary's MAC in their ARP caches. Without GARP, they'll send traffic to the dead primary for up to the ARP cache TTL (typically 300-1800 seconds — catastrophic for HA).

```bash
# arping sends GARP: "VIP 10.0.1.100 is now at MAC aa:bb:cc:dd:ee:ff"
arping -A -I eth0 10.0.1.100     # -A sends gratuitous reply
arping -U -I eth0 10.0.1.100     # -U sends unsolicited request (broader compat)

# keepalived does this automatically on VRRP failover
# Check keepalived script:
grep "garp" /etc/keepalived/keepalived.conf
```

**What can go wrong:**

1. **GARP dropped by cloud provider:** AWS, Azure, and GCP block GARP in most configurations. VPC routing is controlled by route tables, not ARP. For cloud HA, update the route table to point to the new instance's ENI — GARP will be ignored.

2. **Switch port security rejecting GARP:** Cisco Dynamic ARP Inspection (DAI) blocks ARP packets that don't match the DHCP binding table. A VIP that wasn't assigned by DHCP gets its GARP dropped.

3. **ARP cache timeout race:** If the failover takes longer than the retry interval, some hosts will have already removed the primary's MAC entry and try ARP discovery — during which time they'll get no response and requests will fail. Send GARP on VRRP state change, not after.

4. **Gratuitous ARP storm:** Some misconfigured HA setups send GARP on every heartbeat. This floods the network and causes unnecessary ARP table churn on every host in the broadcast domain.

```bash
# Check if GARP is being received
tcpdump -i eth0 arp and arp[6:2] = 2    # ARP Reply (gratuitous)

# Check ARP cache on hosts to verify they updated
ip neigh show | grep <VIP>
arp -n | grep <VIP>
```

**Follow-up Questions:**
- How does keepalived's VRRP work with GARP on AWS vs on-premises?
- What is Proxy ARP and when is it useful?
- How does NDP (Neighbor Discovery Protocol) replace ARP in IPv6?

**Connecting Points:**
- `01-networking-fundamentals/` for ARP protocol details
- `05-network-security/` for ARP spoofing attacks and defenses

---

## Key Takeaways

- TCP state is held independently by each side — CLOSE_WAIT is ALWAYS an application bug (kernel can't call close() for you); TIME_WAIT is expected and correct
- BGP is a policy protocol: LOCAL_PREF (your policy) always beats AS_PATH (distance) in the tie-breaking order
- NAT exhaustion has a structural fix (persistent connections) and a scaling fix (multiple source IPs) — name both in interviews
- DNS failure types are diagnostic categories: NXDOMAIN = authoritative answer, SERVFAIL = resolver error, Timeout = network or overload
- ECMP hash polarization is a real production problem that requires entropy injection or different hash seeds per tier to fix
