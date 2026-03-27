# Interview Prep — Study Guide and Methodology

## Table of Contents

- [Quick Reference](#quick-reference)
- [Q: How should I structure 4 weeks of interview prep for a Staff SRE networking role?](#q-how-should-i-structure-4-weeks-of-interview-prep-for-a-staff-sre-networking-role)
  - [Week 1: Foundations and Internals (Sections 01, 02)](#week-1-foundations-and-internals-sections-01-02)
  - [Week 2: Cloud and Kubernetes Networking (Sections 03, 04)](#week-2-cloud-and-kubernetes-networking-sections-03-04)
  - [Week 3: Security, Deep Dives, and Q&A Practice (Sections 05, 09, 10)](#week-3-security-deep-dives-and-qa-practice-sections-05-09-10)
  - [Week 4: Integration, System Design, and Polish (Sections 06, 07, 08)](#week-4-integration-system-design-and-polish-sections-06-07-08)
- [Q: What prerequisite knowledge do I need before this prep is useful?](#q-what-prerequisite-knowledge-do-i-need-before-this-prep-is-useful)
  - [Must-Know Cold (No Reference)](#must-know-cold-no-reference)
- [Q: How do you answer networking questions at staff level? Give me the methodology.](#q-how-do-you-answer-networking-questions-at-staff-level-give-me-the-methodology)
- [Critical Commands Cheat Sheet (30 Most Important)](#critical-commands-cheat-sheet-30-most-important)
- [Q: What do interviewers actually evaluate — what separates Senior from Staff from Principal?](#q-what-do-interviewers-actually-evaluate-what-separates-senior-from-staff-from-principal)
  - [Senior SRE (L5 equivalent)](#senior-sre-l5-equivalent)
  - [Staff SRE (L6 equivalent)](#staff-sre-l6-equivalent)
  - [Principal SRE (L7 equivalent)](#principal-sre-l7-equivalent)
- [Key Takeaways](#key-takeaways)

---

## Quick Reference

This section is the synthesis layer for the entire `le-network-security` knowledge base. It does not teach concepts — it teaches you how to demonstrate mastery under interview conditions. Senior SRE interviews test mental models, debugging methodology, and trade-off reasoning, not encyclopedic recall. This guide maps 4 weeks of prep to the sections in this repo, defines prerequisite knowledge, and explains the evaluation rubric interviewers actually use at the Senior, Staff, and Principal levels.

---

## Q: How should I structure 4 weeks of interview prep for a Staff SRE networking role?

**What the interviewer is testing:** Not applicable — this is your study plan. But interviewers evaluate whether you have breadth (know all the layers) plus depth (can go to kernel internals on at least 2-3 topics).

**Model Answer (Study Plan):**

### Week 1: Foundations and Internals (Sections 01, 02)

**Goal:** Build a packet-path mental model you can recite from memory in any direction — from application to wire, from L7 down to L1.

| Day | Focus | Sections | Time |
|-----|-------|----------|------|
| Mon | TCP/IP fundamentals, OSI as debugging framework | `01-networking-fundamentals` | 3h |
| Tue | Linux network stack internals, sk_buff lifecycle | `02-linux-networking` | 3h |
| Wed | Routing, BGP basics, ECMP | `01-networking-fundamentals` | 2h |
| Thu | DNS architecture, conntrack internals | `02-linux-networking` | 3h |
| Fri | Draw the packet path from memory. No notes. | Both | 1h |
| Sat | Critical commands — run all 30 on a test system | `11-hands-on-labs` | 3h |
| Sun | Review gaps; start flash cards | — | 2h |

### Week 2: Cloud and Kubernetes Networking (Sections 03, 04)

**Goal:** Map Linux fundamentals to cloud abstractions. Know that an AWS Security Group is a stateful connection tracker, a VPC route table is a kernel routing table, and a kube-proxy iptables rule is just conntrack DNAT.

| Day | Focus | Sections | Time |
|-----|-------|----------|------|
| Mon | AWS VPC, subnets, route tables, Security Groups | `03-cloud-networking` | 3h |
| Tue | EKS/GKE networking, CNI plugins | `04-kubernetes-networking` | 3h |
| Wed | Kubernetes Services: ClusterIP, NodePort, LoadBalancer | `04-kubernetes-networking` | 2h |
| Thu | Cloud security: NACLs, PrivateLink, Transit Gateway | `03-cloud-networking` | 2h |
| Fri | Cross-cloud scenarios: Azure CNI, ExpressRoute | `03-cloud-networking` | 2h |
| Sat | Hands-on: deploy EKS, trace pod-to-pod traffic | `11-hands-on-labs` | 3h |
| Sun | Review; practice one system design question aloud | `08-system-design` | 2h |

### Week 3: Security, Deep Dives, and Q&A Practice (Sections 05, 09, 10)

**Goal:** Demonstrate depth on 3-4 topics where you go to source-code level. Pick topics you have real experience with — BBR vs CUBIC, eBPF XDP, Cilium, TLS 1.3 internals.

| Day | Focus | Sections | Time |
|-----|-------|----------|------|
| Mon | TLS handshake, PKI, mTLS, certificate lifecycle | `05-network-security` | 3h |
| Tue | DDoS mitigation, WAF, network segmentation | `05-network-security` | 2h |
| Wed | eBPF, XDP, AF_XDP, Cilium architecture | `09-advanced-topics` | 3h |
| Thu | TCP internals: BBR, CUBIC, BBR v2, congestion window | `09-advanced-topics` | 2h |
| Fri | Q&A practice: conntrack, CLOSE_WAIT, MTU black holes | `10-interview-prep` (this section) | 3h |
| Sat | Q&A practice: cross-domain scenarios | `10-interview-prep` | 3h |
| Sun | Mock interview: 2 scenario questions, timed | — | 2h |

### Week 4: Integration, System Design, and Polish (Sections 06, 07, 08)

**Goal:** Connect everything. Practice talking through system designs for 20 minutes. Remove filler words from your debugging narration.

| Day | Focus | Sections | Time |
|-----|-------|----------|------|
| Mon | System design: multi-region active-active | `08-system-design` | 3h |
| Tue | System design: EKS cluster networking end-to-end | `08-system-design` | 2h |
| Wed | Debugging playbooks: practice from symptoms | `07-debugging-playbooks` | 2h |
| Thu | Interview patterns and pitfalls — read it all | `10-interview-prep/08-interview-patterns-pitfalls.md` | 2h |
| Fri | Full mock interview: 90 minutes, all question types | — | 2h |
| Sat | Review weak areas from mock | — | 2h |
| Sun | Light review; rest | — | 1h |

**Connecting Points:**
- See `10-interview-prep/08-interview-patterns-pitfalls.md` for the evaluation rubric by level
- See `10-interview-prep/06-system-design-qa.md` for system design frameworks

---

## Q: What prerequisite knowledge do I need before this prep is useful?

**What the interviewer is testing:** Self-awareness about your current level. Interviewers can tell when you are bullshitting by the precision of your answers.

**Model Answer:**

Before this knowledge base adds value, be comfortable with these without looking them up:

### Must-Know Cold (No Reference)

**TCP/IP fundamentals:**
- Three-way handshake sequence (SYN, SYN-ACK, ACK with sequence numbers)
- What ESTABLISHED, TIME_WAIT, CLOSE_WAIT, FIN_WAIT states mean
- Why TIME_WAIT exists (prevent old packets from corrupting new connections, 2×MSL)
- IPv4 header fields: TTL, DSCP, DF bit, protocol field
- Subnet math: /24 = 256 hosts, /25 = 128, /26 = 64, /27 = 32

**Linux networking commands:**
- `ss -tnp` — list TCP connections with process
- `ip route get <IP>` — show which route the kernel selects
- `ip netns exec <ns> <cmd>` — run command in network namespace
- `tcpdump -i eth0 -nn port 443` — basic capture
- `ethtool -S eth0` — NIC statistics

**Basic socket programming mental model:**
- `socket()` → `bind()` → `listen()` → `accept()` lifecycle (server)
- `socket()` → `connect()` lifecycle (client)
- What a file descriptor is and why fd leaks cause CLOSE_WAIT

**Kubernetes basics:**
- Pod, Service, Deployment, Namespace definitions
- What a CNI plugin does (assigns IPs, configures routes)
- What kube-proxy does (programs iptables/IPVS for ClusterIP)

**Follow-up Questions:**
- What is the sequence number in a TCP SYN packet?
- Why does a client go to TIME_WAIT, not the server?
- What does `ip_forward=0` break and why?

**Connecting Points:**
- `01-networking-fundamentals` for TCP deep dives
- `02-linux-networking` for Linux-specific commands

---

## Q: How do you answer networking questions at staff level? Give me the methodology.

**What the interviewer is testing:** Whether you have a framework for thinking under pressure, or whether you memorize answers.

**Model Answer:**

The Q&A methodology for staff-level networking questions follows four phases. Practice each phase explicitly until it becomes unconscious.

**Phase 1: Restate and bound (15 seconds)**

Before answering, restate what you heard and clarify scope. "You're asking about conntrack table exhaustion — are we talking about a Kubernetes cluster, bare metal proxy, or cloud VPC? I'll cover the general case and call out Kubernetes specifics."

This signals: you think before you talk, and you know context matters.

**Phase 2: Layered diagnosis (your core answer, 2-3 minutes)**

Structure every answer as a diagnostic sequence:
1. What does the symptom tell us (what layer is failing)?
2. What is the first command to run and why?
3. What are the 3 most likely causes, in probability order?
4. What is the fix for each cause?
5. What is the long-term structural fix (not just the immediate workaround)?

Example for "connection drops under load":
```
L1/L2: ethtool -S for NIC drops
L3: ip route, conntrack -S for drop counters
L4: ss -s for socket state counts, nstat for retransmissions
App: accept queue depth, fd limits, thread pool saturation
```

**Phase 3: Trade-offs (30 seconds)**

Every answer must include "this trades X for Y." If you cannot name the trade-off, you do not understand the solution deeply enough.

Examples of required trade-off statements:
- "Increasing conntrack_max reduces drops but costs 320 bytes per entry — at 2M entries, that is 640MB of kernel memory."
- "MSS clamping fixes PMTUD black holes but only works for TCP. QUIC and UDP still need separate handling."
- "NodeLocal DNSCache eliminates conntrack race conditions for DNS but adds a DaemonSet to manage and can serve stale records."

**Phase 4: Connecting points (15 seconds)**

End with a pointer to depth: "If you want to go deeper on this, I can walk through how Cilium's eBPF connection tracking eliminates the conntrack table entirely, which removes this class of problem." This shows you have depth available and lets the interviewer steer.

**Follow-up Questions:**
- How do you handle "debug this" questions where you can't ask clarifying questions?
- What if your first hypothesis is wrong — how do you pivot?
- How do you communicate uncertainty to an interviewer?

**Connecting Points:**
- `10-interview-prep/08-interview-patterns-pitfalls.md` for the full live debugging methodology
- `10-interview-prep/07-cross-domain-scenarios.md` for practicing the layered approach

---

## Critical Commands Cheat Sheet (30 Most Important)

| # | Command | Purpose | When to Use |
|---|---------|---------|-------------|
| 1 | `ss -tnp` | List TCP connections with PIDs | First command for any connectivity issue |
| 2 | `ss -s` | Socket state summary | COUNT: how many ESTABLISHED, TIME_WAIT, etc. |
| 3 | `ss -tlnp` | Show listening ports with PIDs | Verify service is bound to expected port/interface |
| 4 | `ip route get <IP>` | Show kernel's route selection | Debug routing: which interface, gateway, src IP |
| 5 | `ip route show table all` | Show all routing tables | Policy routing, VPN routes, Kubernetes routes |
| 6 | `conntrack -S` | Conntrack per-CPU stats | Look for `drop`, `insert_failed` counters |
| 7 | `conntrack -L \| wc -l` | Current conntrack count | Compare to `nf_conntrack_max` |
| 8 | `tcpdump -i eth0 -nn -s0 port 443` | Packet capture | Any network-level debugging |
| 9 | `tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'` | SYN only | DDoS, connection establishment debugging |
| 10 | `ethtool -S eth0 \| grep -i drop` | NIC drop counters | Ring buffer full, driver drops |
| 11 | `ethtool -c eth0` | Interrupt coalescing settings | High CPU from NIC interrupts |
| 12 | `nstat -a \| grep -i retrans` | TCP retransmission counters | Packet loss, congestion |
| 13 | `cat /proc/net/softnet_stat` | NAPI budget exhaustion | Columns: processed, dropped, time_squeezed |
| 14 | `bpftool prog show` | List loaded eBPF programs | XDP/TC debugging |
| 15 | `nft list ruleset` | All nftables rules | Modern kernel firewall audit |
| 16 | `iptables -L FORWARD -n -v --line-numbers` | iptables forward rules | Container networking, K8s pod-to-pod |
| 17 | `ip netns exec <ns> <cmd>` | Run command in network namespace | Container/pod network debugging |
| 18 | `dig +trace <name>` | Full DNS resolution chain | DNS delegation failures |
| 19 | `dig @<dns-ip> <name>` | Query specific DNS server | Isolate stub vs recursive vs authoritative |
| 20 | `curl -w "dns:%{time_namelookup} connect:%{time_connect} ttfb:%{time_starttransfer}"` | HTTP timing breakdown | Latency diagnosis by phase |
| 21 | `openssl s_client -connect host:443 -tls1_3` | TLS handshake | Certificate validation, TLS version |
| 22 | `ping -M do -s 1472 <IP>` | MTU path test | PMTUD black hole detection |
| 23 | `traceroute -T -p 443 <host>` | TCP traceroute | Firewall detection, path analysis |
| 24 | `sysctl net.ipv4.tcp_retries2` | Max TCP retransmissions | Connection timeout behavior |
| 25 | `cat /proc/net/sockstat` | System socket memory usage | TCP memory pressure diagnosis |
| 26 | `bpftrace -e 'kprobe:tcp_drop { @[kstack] = count(); }'` | Kernel TCP drop trace | Advanced: find where drops happen in kernel |
| 27 | `kubectl get endpoints <svc>` | K8s service endpoints | Service not routing: check endpoint health |
| 28 | `kubectl exec <pod> -- ss -tnp` | Pod socket state | Kubernetes networking debugging |
| 29 | `bridge fdb show dev <vxlan-if>` | VXLAN forwarding database | VXLAN overlay debugging |
| 30 | `ip -s link show eth0` | Interface statistics | TX/RX errors, drops, overruns |

---

## Q: What do interviewers actually evaluate — what separates Senior from Staff from Principal?

**What the interviewer is testing:** This is meta — but understanding the rubric helps you pitch your answers at the right level.

**Model Answer:**

### Senior SRE (L5 equivalent)
**Evaluating:** Can you diagnose a production incident systematically? Do you know the right commands? Can you articulate why a fix works?

What they want to see:
- Correct diagnosis sequence (bottom-up, measurement before action)
- Specific commands with correct flags
- Awareness of common failure modes for each technology
- Can explain one thing deeply (your area of experience)

What kills Senior candidates:
- "Let me restart the service and see if that helps"
- Commands without explaining why
- No awareness of trade-offs

### Staff SRE (L6 equivalent)
**Evaluating:** Can you reason about systems you haven't directly operated? Do your answers include second-order effects? Can you design at scale?

What they want to see:
- Trade-offs stated explicitly, with numbers (memory cost, latency impact, failure modes)
- Cross-domain connections (this iptables rule affects K8s, which affects this SLO)
- Structural fix vs immediate workaround, and why you'd do both
- Can connect to security and compliance implications unprompted

What kills Staff candidates:
- Correct diagnosis but no awareness of "what breaks at 10x scale"
- "It depends" without specifying what it depends on
- No opinion — hedging every answer

### Principal SRE (L7 equivalent)
**Evaluating:** Can you set technical direction? Do you know what you don't know? Can you estimate when to stop investing in a solution?

What they want to see:
- When to use a simple solution vs invest in a complex one (iptables vs eBPF — the tradeoff isn't just performance)
- Awareness of industry direction (nftables replacing iptables, kube-proxy's future, eBPF ubiquity)
- Failure modes of your own recommendations
- Economic awareness: "this adds $X/month, here's when that matters"

What kills Principal candidates:
- Overengineering every answer ("let's use eBPF for this")
- Not knowing when to say "I'd call the vendor" or "I'd read the kernel source"

**Follow-up Questions:**
- What is the most important networking topic for a Staff SRE in 2026?
- How do you stay current on kernel changes that affect production?
- When would you escalate to a kernel networking expert?

**Connecting Points:**
- `10-interview-prep/08-interview-patterns-pitfalls.md` for the full evaluation breakdown
- `10-interview-prep/09-exotic-deep-dive-qa.md` for Principal-level depth topics

---

## Key Takeaways

- The 4-week plan front-loads fundamentals because everything else builds on packet-path mental models
- Prerequisite knowledge is binary — if you can't subnet and read `ss` output, stop here and build that first
- Staff-level answers always include a trade-off with specific numbers (bytes, milliseconds, connections/sec)
- The 30 commands are not a memorization list — they are a diagnostic protocol organized by layer
- Principal candidates differentiate by knowing when NOT to use advanced solutions and by citing industry direction
