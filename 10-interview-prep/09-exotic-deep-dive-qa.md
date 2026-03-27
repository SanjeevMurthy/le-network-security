# Exotic and Deep Dive Q&A — Kernel Internals and Advanced TCP/IP

## Table of Contents

- [Quick Reference](#quick-reference)
- [Q: Walk through the complete path of an incoming packet from NIC to userspace socket buffer. Include interrupt handling, NAPI, sk_buff, all kernel subsystems.](#q-walk-through-the-complete-path-of-an-incoming-packet-from-nic-to-userspace-socket-buffer-include-interrupt-handling-napi-sk_buff-all-kernel-subsystems)
- [Q: Explain NAPI and interrupt coalescing. A high-traffic server has high CPU from NIC interrupts despite low packet loss. How do you tune it?](#q-explain-napi-and-interrupt-coalescing-a-high-traffic-server-has-high-cpu-from-nic-interrupts-despite-low-packet-loss-how-do-you-tune-it)
- [Q: Compare TCP BBR and TCP CUBIC. A backend service on a WAN link (100ms RTT, 1% packet loss) achieves only 10% of available bandwidth. Would switching to BBR help?](#q-compare-tcp-bbr-and-tcp-cubic-a-backend-service-on-a-wan-link-100ms-rtt-1-packet-loss-achieves-only-10-of-available-bandwidth-would-switching-to-bbr-help)
- [Q: Explain conntrack table internals. Production is logging "nf_conntrack: table full." Walk through diagnosis, remediation, and when to bypass conntrack entirely.](#q-explain-conntrack-table-internals-production-is-logging-nf_conntrack-table-full-walk-through-diagnosis-remediation-and-when-to-bypass-conntrack-entirely)
- [Q: Explain AF_XDP (XDP Sockets) and zero-copy packet processing. When would you use AF_XDP instead of standard sockets?](#q-explain-af_xdp-xdp-sockets-and-zero-copy-packet-processing-when-would-you-use-af_xdp-instead-of-standard-sockets)
- [Q: What is the SO_REUSEPORT socket option, how does it work, and what problem does it solve for high-performance servers?](#q-what-is-the-so_reuseport-socket-option-how-does-it-work-and-what-problem-does-it-solve-for-high-performance-servers)
- [Q: What does the BPF verifier check and why does it reject programs? What are the most common reasons a production eBPF program fails to load?](#q-what-does-the-bpf-verifier-check-and-why-does-it-reject-programs-what-are-the-most-common-reasons-a-production-ebpf-program-fails-to-load)
- [Q: How does Cilium's eBPF datapath replace iptables? Walk through the actual BPF hook points and what they replace.](#q-how-does-ciliums-ebpf-datapath-replace-iptables-walk-through-the-actual-bpf-hook-points-and-what-they-replace)
- [Key Takeaways](#key-takeaways)

---

## Quick Reference

This section contains questions that appear in final-round interviews at infrastructure-focused companies (Cloudflare, Meta, Google, Fastly, kernel-focused startups). They test whether you understand how the system works at the source-code level, not just how to use it. If you can answer these credibly, you demonstrate that you have read kernel source, studied networking internals, or built systems at a level where these details mattered. Source material: `09_Exotic_and_Deep_Dive_Questions.md` (extended). See `09-advanced-topics/` for eBPF and kernel internals deep dives.

---

## Q: Walk through the complete path of an incoming packet from NIC to userspace socket buffer. Include interrupt handling, NAPI, sk_buff, all kernel subsystems.

**What the interviewer is testing:** Whether you understand the Linux networking stack at a level that enables precise performance tuning and debugging at any layer.

**Model Answer:**

The complete ingress packet path traverses hardware context, softirq context, and process context:

**Stage 1 — Hardware DMA:**
```
Wire → NIC PHY/MAC → FCS validation → DMA write to ring buffer
```
The NIC validates the frame checksum (FCS) and writes packet data to a pre-allocated DMA ring buffer in kernel memory — no CPU involvement. After writing, the NIC raises a hardware IRQ.

**Stage 2 — Hardware Interrupt → NAPI:**
The IRQ handler runs in hardirq context (all interrupts on this CPU disabled). It does minimal work:
```c
irqreturn_t driver_irq_handler(int irq, void *dev_id) {
    napi_schedule(&adapter->napi);    // schedule NAPI poll
    // disable further NIC interrupts (crucial for NAPI)
    return IRQ_HANDLED;
}
```
`napi_schedule()` raises `NET_RX_SOFTIRQ`. The softirq handler `net_rx_action()` runs shortly after in softirq context (interrupts re-enabled).

**Stage 3 — NAPI poll loop:**
```c
int driver_poll(struct napi_struct *napi, int budget) {
    int processed = 0;
    while (processed < budget) {
        skb = alloc_skb();          // allocate sk_buff
        napi_gro_receive(napi, skb); // pass to GRO then stack
        processed++;
    }
    if (processed < budget)
        napi_complete(napi);        // re-enable NIC interrupts
    return processed;
}
```
Budget default: 300 packets per poll cycle (`net.core.netdev_budget`). Adaptive behavior: high traffic = polling mode (interrupts disabled), low traffic = interrupt mode.

**Stage 4 — GRO (Generic Receive Offload):**
Before passing sk_buffs up the stack, GRO coalesces multiple TCP segments from the same flow into one large sk_buff:
```
4 × 1500-byte TCP segments → GRO → 1 × 6000-byte sk_buff
(4 stack traversals)                  (1 stack traversal)
```

**Stage 5 — Netdev dispatch (net/core/dev.c):**
```
netif_receive_skb()
  → [XDP hook] — native XDP runs before sk_buff allocation in the driver
                  (operates on raw xdp_buff, can XDP_DROP at NIC-speed)
  → [TC ingress hook] — tc-bpf programs run here with full sk_buff access
  → protocol dispatch — EtherType → ip_rcv()
```

**Stage 6 — IP layer:**
```
ip_rcv()
  → IP header validation (checksum, version, length)
  → [NF_INET_PRE_ROUTING] — conntrack lookup; DNAT rules
  → ip_route_input() — routing decision: local or forward?
  → [NF_INET_LOCAL_IN] — filter INPUT chain
  → ip_local_deliver()
```

**Stage 7 — TCP layer:**
```
tcp_v4_rcv()
  → socket lookup in inet established hash table
  → TCP state machine processing
  → sequence number validation, ACK processing, congestion window update
  → data copied to socket receive queue (sk->sk_receive_queue)
  → wake_up(sk->sk_wq) — wake blocked application
```

**Stage 8 — Application:**
```
Application: read(fd, buf, len)
  → syscall → tcp_recvmsg()
  → copy from sk_receive_queue to userspace buffer
  → return to application
```

**Complete path summary:**
```
Wire → DMA ring buffer → HW IRQ → NAPI schedule →
softirq → NAPI poll → [XDP] → GRO → sk_buff →
[TC ingress] → protocol dispatch →
ip_rcv → [NF_PRE_ROUTING] → route → [NF_INPUT] →
tcp_v4_rcv → socket lookup → receive queue →
wake_up → tcp_recvmsg → copy to userspace
```

**Debugging drops at each stage:**
```bash
ethtool -S eth0 | grep rx_dropped         # Stage 2: ring buffer overflow
cat /proc/net/softnet_stat                 # Stage 3: col 3 = time_squeezed (budget exhausted)
nstat -a | grep TcpExtRcvPruned           # Stage 8: socket buffer pruning
```

**Follow-up Questions:**
- What happens when the ring buffer is full (NIC drops, not kernel drops)?
- How does RSS distribute packets across multiple CPUs?
- What is the difference between XDP native mode and XDP generic mode?

**Connecting Points:**
- `02-linux-networking/` for NAPI tuning reference
- `10-interview-prep/02-linux-networking-qa.md` for packet path in K8s context

---

## Q: Explain NAPI and interrupt coalescing. A high-traffic server has high CPU from NIC interrupts despite low packet loss. How do you tune it?

**What the interviewer is testing:** Whether you understand the interrupt storm problem and can quantify the tuning parameters.

**Model Answer:**

**The interrupt storm problem (pre-NAPI):**
Before NAPI, each incoming packet triggered a hardware interrupt. At 1 million pps: 1M interrupts/sec = CPU spends 100% of its time in interrupt context, makes no actual progress processing packets ("livelock").

**NAPI's hybrid model:**
```
Low traffic:  packet → interrupt → process → wait for next
              (low latency, low CPU)

High traffic: first packet → interrupt → disable interrupts → poll loop
              → process budget packets → re-enable interrupts
              (higher per-packet latency, much lower CPU)
```

**Diagnosing excessive interrupt rates:**
```bash
# Check interrupt rate per NIC queue
cat /proc/interrupts | grep eth0
# Each column = a CPU; if values are > 100k/sec, interrupt rate is high

# Check NAPI budget exhaustion (time_squeezed = batch processing was deferred)
cat /proc/net/softnet_stat
# Columns: processed, dropped, time_squeezed
# time_squeezed > 0: budget exhausted, latency increasing

# Watch interrupt rate in real time
watch -n1 'cat /proc/interrupts | grep eth0'
```

**Hardware-level interrupt coalescing (ethtool):**
The NIC delays its interrupt until N packets accumulate OR T microseconds elapse:
```bash
ethtool -c eth0
# rx-usecs: 50     (wait up to 50μs before interrupting)
# rx-frames: 64    (or until 64 frames received)

# High-throughput, latency-tolerant workloads:
ethtool -C eth0 rx-usecs 100 rx-frames 128
# Halves interrupt rate; adds up to 100μs latency

# Enable adaptive coalescing (NIC auto-tunes):
ethtool -C eth0 adaptive-rx on

# Latency-sensitive (trading, real-time):
ethtool -C eth0 rx-usecs 0 rx-frames 1
# Interrupt on every packet — lowest latency, highest CPU
```

**Software-level NAPI tuning:**
```bash
# Increase NAPI budget for high-throughput servers
sysctl -w net.core.netdev_budget=600          # default 300
sysctl -w net.core.netdev_budget_usecs=8000   # max time per softirq run

# Busy polling: process poll the NIC directly (sub-μs latency)
sysctl net.core.busy_poll=50     # poll for 50μs before sleeping
sysctl net.core.busy_read=50
# Application must set SO_BUSY_POLL on the socket
# Trades CPU for latency

# Threaded NAPI (kernel 5.11+): move NAPI to dedicated kernel threads
echo 1 > /sys/class/net/eth0/threaded
# Prevents NAPI from starving timer/scheduler softirqs
```

**Tuning strategy for the scenario (high CPU, low loss):**
```bash
# Step 1: check if GRO is enabled (reduces packets entering stack)
ethtool -k eth0 | grep generic-receive-offload
ethtool -K eth0 gro on

# Step 2: enable adaptive coalescing
ethtool -C eth0 adaptive-rx on

# Step 3: verify RSS distributes load evenly across CPUs
ethtool -l eth0                              # how many queues?
cat /proc/interrupts | grep eth0             # even distribution?

# Step 4: if time_squeezed is nonzero, increase budget
sysctl -w net.core.netdev_budget=600
```

**Follow-up Questions:**
- What is the `softnet_stat` file and how do you read all three columns?
- How does busy polling differ from NAPI polling in terms of CPU context?
- What is interrupt affinity and why does it matter for cache efficiency?

**Connecting Points:**
- `02-linux-networking/` for NIC tuning reference
- `09-advanced-topics/` for RSS and NUMA-aware interrupt affinity

---

## Q: Compare TCP BBR and TCP CUBIC. A backend service on a WAN link (100ms RTT, 1% packet loss) achieves only 10% of available bandwidth. Would switching to BBR help?

**What the interviewer is testing:** Whether you understand congestion control algorithms at the algorithmic level, not just "BBR is newer."

**Model Answer:**

**CUBIC (loss-based, default since Linux 2.6.19):**
CUBIC grows `cwnd` as a cubic function of time since last loss. When loss is detected (3 duplicate ACKs or RTO): reduce `cwnd` by 30%. CUBIC equates ALL packet loss with congestion.

The Mathis equation gives the theoretical maximum throughput for loss-based CC:
```
throughput ≈ (MSS / RTT) × (1 / sqrt(loss_rate))
           = (1460 / 0.1s) × (1 / sqrt(0.01))
           = 14,600 × 10 = 146 Kbps
```

On a 100 Mbps link, this is 0.15% utilization. Real-world CUBIC on 100ms/1% loss: ~5-10% utilization. This explains the 10% figure in the question.

**BBR (model-based, Google, kernel 4.9+):**
BBR estimates two path parameters:
- **BtlBw**: bottleneck bandwidth (max observed delivery rate)
- **RTprop**: round-trip propagation delay (min observed RTT)

BBR sets sending rate to `BtlBw × RTprop` — keeps the bottleneck fully utilized without building queue. Critically: BBR ignores packet loss as a congestion signal.

```text
BBR on 100ms/1% random loss:
- Sees 1% loss → ignores it (not congestion signal)
- Estimates BtlBw from delivery rate measurements
- Achieves 80-95% of available bandwidth
- Google: 2,700x throughput improvement over CUBIC in similar conditions
```

**Enabling BBR:**
```bash
sysctl net.ipv4.tcp_available_congestion_control    # check if bbr is available
modprobe tcp_bbr                                    # load if needed
sysctl -w net.ipv4.tcp_congestion_control=bbr

# BBR requires fair queuing (fq) qdisc for pacing
tc qdisc replace dev eth0 root fq

# Verify BBR on a specific connection
ss -ti dst <remote-ip> | grep bbr
# Output: bbr:(bw:95Mbps,mrtt:100,pacing_rate:95Mbps)
```

**BBR risks:**
1. **Fairness with CUBIC:** BBR v1 can consume 40% of bandwidth vs 5 CUBIC flows sharing 60% — unfair in shared environments
2. **Buffer inflation:** BBR v1's ProbeBW phase can build persistent queues; BBR v2 (kernel 5.18+, `bbr2`) addresses this
3. **ProbeRTT latency spike:** BBR reduces cwnd to 4 packets for 200ms every 10 seconds to re-estimate RTprop — brief throughput drop
4. **Middlebox interference:** Some firewalls rate-limit flows that don't back off on loss — BBR can trigger these

**Per-service BBR (production approach):**
```c
// Set BBR only for WAN-facing sockets, not LAN sockets
setsockopt(sock, IPPROTO_TCP, TCP_CONGESTION, "bbr", 3);
```

**Follow-up Questions:**
- What is DCTCP and when would you use it in a data center instead of BBR?
- How does BBR v2 differ from BBR v1 in its buffer inflation behavior?
- What does the `fq` qdisc do and why does BBR require it?

**Connecting Points:**
- `02-linux-networking/` for TCP congestion control configuration
- `09-advanced-topics/` for congestion control algorithm comparison

---

## Q: Explain conntrack table internals. Production is logging "nf_conntrack: table full." Walk through diagnosis, remediation, and when to bypass conntrack entirely.

**What the interviewer is testing:** Whether you know conntrack architecture, memory costs, and structural alternatives.

**Model Answer:**

The conntrack table is a hash table indexed by 5-tuple (src_ip, dst_ip, src_port, dst_port, proto). When full, ALL new connections are dropped — not just connections to the overloaded service.

**Conntrack architecture:**
```
Hash function(5-tuple) → bucket → linked list of nf_conn entries
Per entry: original tuple + reply tuple + state + extensions (NAT, timeout)
Memory: ~300-400 bytes per entry (varies by kernel version and extensions)
```

**Diagnosis:**
```bash
# Current usage vs maximum
sysctl net.netfilter.nf_conntrack_count    # current
sysctl net.netfilter.nf_conntrack_max      # maximum
conntrack -S | grep drop                   # confirmed drops

# What's filling the table?
conntrack -L -o extended | awk '{print $4}' | sort | uniq -c | sort -rn
# High TIME_WAIT: short-lived connections not timing out fast enough
# High ESTABLISHED: too many long-lived connections or wrong timeout

# Which service is the offender?
conntrack -L | awk '{for(i=1;i<=NF;i++) if($i ~ /dport=/) print $i}' | \
  sort | uniq -c | sort -rn | head
```

**Immediate mitigation:**
```bash
# Increase table size
sysctl -w net.netfilter.nf_conntrack_max=1000000    # 1M entries
echo 250000 > /sys/module/nf_conntrack/parameters/hashsize  # 250K buckets
# Memory cost: 1M × 400 bytes + 250K × 8 bytes = ~402MB

# Reduce timeouts
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30    # default 120
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400  # default 5 days
sysctl -w net.netfilter.nf_conntrack_udp_timeout=15              # default 30
```

**Hash table sizing — the critical point interviewers test:**
```
nf_conntrack_max / nf_conntrack_buckets = entries per bucket
Default: 262144 / 16384 = 16 entries per bucket
At high fill: traversing 16-entry linked list per packet lookup = O(n) degradation
Rule: buckets = max / 4 for O(1) average lookup
```

**When to bypass conntrack (NOTRACK):**
Use NOTRACK for traffic that doesn't need stateful tracking:
- Health check endpoints that receive millions of requests
- DNS server traffic (stateless protocol)
- Traffic between trusted internal hosts

```bash
# NOTRACK for health checks (saves ~400 bytes per check)
iptables -t raw -A PREROUTING -p tcp --dport 8080 -j NOTRACK
iptables -t raw -A OUTPUT -p tcp --sport 8080 -j NOTRACK

# WARNING: NOTRACK traffic CANNOT use "-m conntrack --ctstate ESTABLISHED"
# If any iptables rules depend on connection state for this port, they will break
# Audit full ruleset before applying NOTRACK
```

**Structural fix — Cilium eBPF:**
Cilium replaces conntrack for pod-to-service traffic with its own eBPF-based connection tracking that scales to 100M+ entries with better cache efficiency and no hash table degradation.

**Follow-up Questions:**
- Why does the `insert_failed` counter in `conntrack -S` indicate a DNS problem?
- How do conntrack zones work and when do you use them?
- What is the conntrack "expectation" mechanism (for FTP RELATED connections)?

**Connecting Points:**
- `10-interview-prep/02-linux-networking-qa.md` for conntrack in K8s context
- `09-advanced-topics/` for Cilium eBPF connection tracking

---

## Q: Explain AF_XDP (XDP Sockets) and zero-copy packet processing. When would you use AF_XDP instead of standard sockets?

**What the interviewer is testing:** Whether you understand the bypass stack model and when kernel processing overhead is the bottleneck.

**Model Answer:**

AF_XDP is a socket family that allows userspace to directly access packet data from the NIC's DMA ring buffer — bypassing the kernel networking stack (sk_buff, routing, netfilter) entirely. It provides kernel-bypass performance with standard Linux socket APIs (unlike DPDK which requires privileged ring access).

**Architecture:**
```
Standard socket:
  NIC DMA → sk_buff allocation → NAPI → TCP/IP stack → socket → copy to userspace
  (multiple copies, kernel processing overhead)

AF_XDP:
  NIC DMA → shared memory (UMEM) → XDP program routes to XDP socket → userspace reads directly
  (zero copies, no kernel TCP/IP stack)
```

**Key data structures:**
```c
// UMEM: userspace memory registered with the kernel
// Packets land here via DMA; userspace reads them directly
struct xsk_umem *umem;
void *bufs;
posix_memalign(&bufs, getpagesize(), NUM_FRAMES * FRAME_SIZE);
xsk_umem__create(&umem, bufs, NUM_FRAMES * FRAME_SIZE, &fq, &cq, NULL);

// XDP socket: receives packets redirected by an XDP program
struct xsk_socket *xsk;
xsk_socket__create(&xsk, "eth0", queue_id, umem, &rx, &tx, NULL);
```

**The XDP program redirects specific packets to AF_XDP:**
```c
SEC("xdp")
int redirect_to_user(struct xdp_md *ctx) {
    // parse packet, check if it should go to userspace
    if (is_my_traffic(ctx))
        return bpf_redirect_map(&xsks_map, ctx->rx_queue_index, 0);
    return XDP_PASS;    // other traffic continues through kernel stack
}
```

**Zero-copy mode (requires NIC driver support):**
```bash
# Enable zero-copy (avoids even the DMA-to-UMEM copy on supported hardware)
# Supported by: i40e (Intel XL710), ice (Intel E810), mlx5 (Mellanox)
# Zero-copy means: NIC writes directly to the registered UMEM
# No copies at all: NIC → UMEM → application (same memory)
```

**Performance:**
- Standard socket throughput: ~2-5 Mpps per core
- AF_XDP copy mode: ~10-15 Mpps per core
- AF_XDP zero-copy: ~24-40 Mpps per core (limited by NIC hardware)

**When to use AF_XDP:**
1. Custom packet processing at > 10 Mpps (DDoS mitigation, packet generation, monitoring)
2. Network appliances where every microsecond of latency matters
3. When you want kernel-bypass performance without DPDK's operational complexity
4. Replacing specialized hardware (TAP/SPAN) with software packet capture

**When NOT to use AF_XDP:**
- Standard web servers, databases — the overhead of TCP/IP stack is negligible
- When you need OS networking features (routing, NAT, socket multiplexing)
- When you don't have NIC driver AF_XDP support

**Follow-up Questions:**
- How does AF_XDP interact with the NAPI poll loop?
- What is the FILL ring and COMPLETION ring in AF_XDP?
- How does AF_XDP compare to DPDK in terms of operational complexity?

**Connecting Points:**
- `09-advanced-topics/` for XDP and eBPF architecture
- `10-interview-prep/02-linux-networking-qa.md` for XDP in DDoS context

---

## Q: What is the SO_REUSEPORT socket option, how does it work, and what problem does it solve for high-performance servers?

**What the interviewer is testing:** Whether you understand kernel socket dispatching and the accept queue bottleneck.

**Model Answer:**

`SO_REUSEPORT` allows multiple sockets to bind to the same IP:port. The kernel distributes incoming connections across all bound sockets — each socket has its own accept queue, and each is typically handled by a separate application thread.

**The problem it solves — accept queue bottleneck:**

Without `SO_REUSEPORT`, all connections to a given port funnel through a single listening socket. A single-threaded event loop or a multi-threaded server with a shared accept queue faces:
1. Lock contention on the accept queue (threads compete for the queue lock)
2. CPU core scaling limitation (one socket = one queue = one CPU handles the accept path)
3. Spurious wakeups (all threads wake up for each connection, only one succeeds — "thundering herd")

**With `SO_REUSEPORT`:**
```c
// Server: bind N sockets to the same port
for (int i = 0; i < num_threads; i++) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
    bind(fd, &addr, sizeof(addr));    // multiple binds to same port succeed
    listen(fd, backlog);
    // Thread i handles fd i — no shared accept queue
}
```

**Kernel dispatching:**
The kernel hashes the 4-tuple (src_ip, src_port, dst_ip, dst_port) to select which socket receives the new connection. Same hash function as RSS — ensures consistent routing for a given flow.

```
Incoming SYN → hash(src_ip, src_port, dst_ip, dst_port) % num_sockets → socket[N]
Each socket has independent accept queue, handled by dedicated thread
```

**Performance impact:**
- Without SO_REUSEPORT: 300K connections/sec (limited by single accept queue lock)
- With SO_REUSEPORT (4 cores): ~1.2M connections/sec (4× scale)
- Used by: nginx, HAProxy, Redis, Envoy, Node.js cluster module

**Kubernetes use:**
Cilium and kube-proxy use SO_REUSEPORT for UDP load balancers (e.g., DNS). The kernel distributes incoming UDP packets across multiple socket instances without application-level dispatch.

**BPF extension (kernel 4.19+):**
```c
// Attach a BPF program to control socket selection
setsockopt(fd, SOL_SOCKET, SO_ATTACH_REUSEPORT_EBPF, &prog_fd, sizeof(prog_fd));
// The BPF program can make routing decisions based on packet content
// Used by: Cilium for identity-aware socket dispatch
```

**Follow-up Questions:**
- What is the thundering herd problem and how does SO_REUSEPORT eliminate it?
- How does SO_REUSEPORT interact with TCP_FASTOPEN?
- What happens to existing connections when a SO_REUSEPORT socket is closed?

**Connecting Points:**
- `02-linux-networking/` for socket option reference
- `09-advanced-topics/` for eBPF socket programs

---

## Q: What does the BPF verifier check and why does it reject programs? What are the most common reasons a production eBPF program fails to load?

**What the interviewer is testing:** Whether you have actually written or deployed eBPF programs and encountered verifier rejections.

**Model Answer:**

The BPF verifier is the kernel's safety net. Before loading any BPF program, the verifier statically analyzes every possible execution path to prove the program is safe to run in kernel context.

**What the verifier checks:**

1. **Termination (no loops without bounds):**
   ```c
   // This is REJECTED before kernel 5.3:
   for (int i = 0; i < n; i++) { ... }    // n is not provably bounded

   // This is ACCEPTED (bounded loop):
   for (int i = 0; i < 10; i++) { ... }    // bound is a compile-time constant
   // Kernel 5.3+: verifier supports bounded loops with proof obligations
   ```

2. **Memory safety (all packet data access is bounds-checked):**
   ```c
   // REJECTED: no bounds check before accessing packet data
   struct iphdr *iph = data + sizeof(struct ethhdr);
   if (iph->protocol == IPPROTO_TCP) { ... }    // verifier doesn't know data is valid

   // ACCEPTED: explicit bounds check
   if (data + sizeof(struct ethhdr) + sizeof(struct iphdr) > data_end)
       return XDP_PASS;
   struct iphdr *iph = data + sizeof(struct ethhdr);
   ```

3. **Register state tracking:**
   The verifier tracks the type and valid range of every register at every program point. A NULL pointer dereference anywhere in the program causes rejection.

4. **Instruction count limit:**
   ```
   Default limit: 1 million instructions (raised from 4096 in kernel 5.2)
   Verified instructions (accounting for loop unrolling) can exceed source count
   Large programs may hit this limit; split into multiple programs with tail calls
   ```

5. **Privileged operations:**
   - `bpf_trace_printk()` requires `CAP_SYS_ADMIN` (or `CAP_BPF` in kernel 5.8+)
   - Unrestricted packet writes require XDP or TC hook (not raw socket programs)

**Common rejection reasons in production:**
```bash
# Debug verifier output
bpftool prog load myprog.o /sys/fs/bpf/myprog 2>&1 | head -50
# Verifier output shows which instruction failed and why

# Most common errors:
# "R2 invalid mem access 'inv'" — uninitialized pointer dereference
# "invalid indirect read from stack" — reading uninitialized stack memory
# "Unreachable insn 42" — dead code that references invalid registers
# "jump out of range" — jump target out of program bounds
# "cannot write into packet" — packet write without verifying write capability
```

**CO-RE (Compile Once, Run Everywhere):**
CO-RE allows an eBPF program compiled on one kernel version to run correctly on different kernel versions where struct layouts differ.

```c
// Without CO-RE: struct field offset hardcoded
// Breaks if kernel changes struct layout
__u32 pid = *((__u32 *)(task + 1234));  // offset 1234 only valid on this kernel

// With CO-RE: BTF-based relocation
struct task_struct *task = bpf_get_current_task_btf();
__u32 pid = BPF_CORE_READ(task, pid);   // offset resolved at load time via BTF
```

CO-RE requires:
- Kernel compiled with `CONFIG_DEBUG_INFO_BTF=y` (all kernels since 5.5 with debug info)
- libbpf 0.4+ for loading
- Clang/LLVM for compilation (`-g -target bpf`)

**Why CO-RE matters for production deployment:**
Without CO-RE, you need to either:
1. Compile the program on the target kernel (requires build toolchain on prod)
2. Distribute different binaries for each kernel version (maintenance nightmare)
3. Use BPF maps to pass in offsets at runtime (fragile)

With CO-RE: compile once, ship a single binary, it works on kernel 5.10 through 6.12+.

**Follow-up Questions:**
- What is BTF (BPF Type Format) and how does it enable CO-RE?
- How do you profile an eBPF program's performance impact?
- What is the difference between BPF tail calls and BPF subprograms?

**Connecting Points:**
- `09-advanced-topics/` for eBPF program types and hooks
- `10-interview-prep/02-linux-networking-qa.md` for eBPF vs iptables comparison

---

## Q: How does Cilium's eBPF datapath replace iptables? Walk through the actual BPF hook points and what they replace.

**What the interviewer is testing:** Whether you understand Cilium as a kernel-level networking system, not just a "iptables replacement."

**Model Answer:**

Cilium uses eBPF programs attached to multiple kernel hook points to replace kube-proxy (iptables service routing), NetworkPolicy (iptables filter rules), and conntrack (for pod-to-service traffic).

**Hook point mapping:**

**1. cgroup-bpf hooks (replace kube-proxy ClusterIP DNAT):**
```
Application: connect(sockfd, {dst=ClusterIP:80})
  → cgroup_sock_ops BPF program intercepts at socket layer (BEFORE packet is created)
  → BPF map lookup: key=ClusterIP:80 → value=endpoint_list
  → Select endpoint (Maglev consistent hashing)
  → Rewrite socket destination: dst=PodIP:8080
  → TCP handshake goes directly to pod (no DNAT, no conntrack entry)
```

This is the key insight: service load balancing happens at socket() level, not packet level. No DNAT, no conntrack DNAT entry.

**2. XDP hook (replace iptables DROP for external traffic):**
```
Packet arrives at NIC → XDP program (native mode)
  → BPF map lookup: source IP in block list?
  → If yes: XDP_DROP (before sk_buff allocation — minimal CPU)
  → If no: XDP_PASS (continue to network stack)
```

**3. TC (Traffic Control) BPF hooks (replace iptables NetworkPolicy):**
```
Pod egress packet:
  → TC egress BPF on pod's veth pair
  → Identity lookup: what is this pod's identity? (based on labels, stored in BPF map)
  → Policy lookup: is this source identity allowed to reach this destination identity?
  → If allowed: TC_ACT_OK (pass)
  → If denied: TC_ACT_DROP
  → No iptables traversal, no conntrack lookup

Pod ingress packet:
  → TC ingress BPF on pod's veth pair
  → Same identity-based policy check
```

**4. BPF LB (external load balancing via XDP):**
```
Traffic arrives from internet:
  → XDP BPF program on external NIC
  → Maglev hash → select backend
  → bpf_redirect_map() to backend's NIC
  → Backend processes packet directly (DSR mode: no return through LB)
```

**What Cilium eliminates:**
- kube-proxy: service DNAT via cgroup-bpf socket hooks
- conntrack: for pod-to-service traffic (socket-level rewrite, no NAT)
- iptables FORWARD chain rules: replaced by TC-BPF on each veth
- iptables NetworkPolicy: replaced by identity-based BPF maps

**What still uses iptables/conntrack:**
- Host-level traffic (kubelets, node agents)
- Traffic from outside the cluster to NodePort services
- Some masquerade (SNAT) rules — though Cilium can replace these with eBPF masquerading

```bash
# Verify Cilium's BPF programs are loaded
bpftool prog list | grep cilium
# Should show: cgroup_connect4, cgroup_connect6, xdp, tc_ingress, tc_egress

# Check service endpoint map
cilium bpf lb list
# Shows all ClusterIPs and their backend endpoints in BPF maps

# Monitor policy decisions in real time
cilium monitor --type drop    # shows dropped packets with policy reason
```

**Follow-up Questions:**
- What is Cilium's Maglev consistent hashing and how does it improve on random selection?
- How does Cilium handle traffic between nodes in native routing mode?
- What is Cilium Hubble and how does it get flow visibility without packet copies?

**Connecting Points:**
- `10-interview-prep/04-kubernetes-networking-qa.md` for Cilium vs kube-proxy comparison
- `09-advanced-topics/` for Cilium architecture

---

## Key Takeaways

- The complete NIC-to-socket packet path has 8 distinct stages — know where each major kernel subsystem attaches (XDP at Stage 2, TC at Stage 5, netfilter at Stage 6) and how to diagnose drops at each
- NAPI solves interrupt storms with adaptive interrupt/polling — tune with `ethtool -C` for hardware coalescing and `netdev_budget` for software budget; measure with `softnet_stat` time_squeezed
- BBR is dramatically better than CUBIC on lossy WAN links (100ms/1% loss: CUBIC ~5%, BBR ~85% utilization) but has known fairness and buffer inflation issues — use per-socket setsockopt for targeted deployment
- The BPF verifier rejects programs on three main grounds: unbounded loops, missing packet bounds checks, and instruction count exceeding 1M — CO-RE solves the portability problem across kernel versions
- Cilium replaces kube-proxy by intercepting at the socket layer (cgroup-bpf) before packets are created — eliminating DNAT, conntrack entries, and O(n) iptables traversal simultaneously
