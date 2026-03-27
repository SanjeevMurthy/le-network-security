# Lab 05: eBPF and XDP

## Learning Objectives

- Write, compile, and load a working XDP program from scratch in C
- Understand how XDP operates at the earliest point in the Linux network stack (before SKB allocation)
- Use eBPF maps to share state between kernel programs and userspace
- Use bpftrace for zero-overhead TCP observability in production
- Observe Kubernetes network flows and policy drops using Hubble (Cilium)

## Prerequisites

- Linux kernel 5.10+ (check: `uname -r`)
- Packages: `clang`, `llvm`, `libbpf-dev`, `bpftool`, `linux-headers-$(uname -r)`, `bpftrace`
- For Lab 5D: Kubernetes cluster with Cilium and Hubble enabled

```bash
# Install prerequisites (Ubuntu 22.04)
sudo apt-get update && sudo apt-get install -y \
  clang llvm libelf-dev libbpf-dev \
  linux-headers-$(uname -r) \
  bpftool \
  bpftrace \
  iproute2 iputils-ping netcat-openbsd

# Verify kernel version
uname -r
# Expected: 5.15.x or 6.x.x

# Verify bpf filesystem is mounted
mount | grep bpf
# Expected: bpffs on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
# If not mounted:
sudo mount -t bpf none /sys/fs/bpf

# Verify bpftool
sudo bpftool version
# Expected: bpftool v7.x.x

# Create working directory
mkdir -p /tmp/ebpf-lab && cd /tmp/ebpf-lab
export LAB_DIR=/tmp/ebpf-lab
```

## Debugging Methodology Alignment

eBPF/XDP sits at Layer 2 in the network stack — before netfilter, before routing, before any userspace process sees the packet. This makes it ideal for:
- Zero-overhead packet observability (no packet copies, kernel-native)
- High-performance packet drop (DDoS mitigation without iptables overhead)
- Production tracing without code instrumentation

These labs reinforce the understanding that the full packet path from `07-debugging-playbooks/00-debugging-methodology.md` has eBPF hooks at every layer.

---

## Lab 5A: Write a Minimal XDP Packet Counter

**Objective:** Write an XDP program in C that counts packets per source IP using a BPF hash map, compile it with clang, load it onto a veth interface, and read the counts from userspace.

---

### The XDP Program (C source)

```bash
cat > $LAB_DIR/xdp_counter.c << 'EOF'
// xdp_counter.c — counts packets per source IPv4 address
// Attach to an interface; read packet counts from BPF map

#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

// BPF hash map: key = source IP (u32), value = packet count (u64)
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, __u32);
    __type(value, __u64);
    __uint(max_entries, 1024);
} pkt_count SEC(".maps");

// XDP program entry point
SEC("xdp")
int xdp_packet_counter(struct xdp_md *ctx)
{
    // ctx->data and ctx->data_end define the packet boundaries
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    // Parse Ethernet header
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;  // malformed packet, let kernel handle

    // Only process IPv4 (ethertype 0x0800)
    if (eth->h_proto != bpf_htons(ETH_P_IP))
        return XDP_PASS;

    // Parse IPv4 header
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    // Look up or create counter for this source IP
    __u32 src_ip = ip->saddr;
    __u64 *count = bpf_map_lookup_elem(&pkt_count, &src_ip);

    if (count) {
        // Increment existing counter
        __sync_fetch_and_add(count, 1);
    } else {
        // Initialize counter at 1 for new source IP
        __u64 initial = 1;
        bpf_map_update_elem(&pkt_count, &src_ip, &initial, BPF_ANY);
    }

    return XDP_PASS;  // pass packet up the stack (don't drop)
}

// License is required — BPF programs must declare GPL-compatible license
char _license[] SEC("license") = "GPL";
EOF
```

### Compile the XDP Program

```bash
cd $LAB_DIR

# Compile to eBPF bytecode using clang
clang -O2 -g \
  -target bpf \
  -D__TARGET_ARCH_x86 \
  -I /usr/include/$(uname -m)-linux-gnu \
  -c xdp_counter.c \
  -o xdp_counter.o

# Verify the compiled object
file xdp_counter.o
# Expected: xdp_counter.o: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV)

# List the sections in the BPF object
llvm-objdump -h xdp_counter.o
# Expected sections:
# xdp         — the XDP program
# .maps       — the BPF map definitions
# license     — GPL license string
```

### Setup: Create a veth Interface for Testing

```bash
# Create a veth pair: veth-test <-> veth-peer
sudo ip link add veth-test type veth peer name veth-peer
sudo ip link set veth-test up
sudo ip link set veth-peer up

# Assign an IP to veth-peer so we can send traffic through the interface
sudo ip addr add 192.168.99.1/24 dev veth-peer
sudo ip addr add 192.168.99.2/24 dev veth-test

echo "veth pair created: veth-test <-> veth-peer"
ip link show veth-test
```

### Load the XDP Program

```bash
cd $LAB_DIR

# Load XDP program onto veth-test (native mode for veth = skb mode)
sudo bpftool prog load xdp_counter.o /sys/fs/bpf/xdp_counter

# Get the program ID
PROG_ID=$(sudo bpftool prog show pinned /sys/fs/bpf/xdp_counter | head -1 | awk '{print $1}' | tr -d ':')
echo "Program ID: $PROG_ID"

# Attach the XDP program to veth-test in skb mode
# (skb mode = compatible with veth; native mode requires driver support)
sudo ip link set veth-test xdpgeneric obj xdp_counter.o sec xdp

# Verify attachment
ip link show veth-test
# Expected output includes:
# xdpgeneric  id <N>  tag <hash>
```

### Generate Test Traffic

```bash
# Send some ping packets through the interface (source IP = 192.168.99.2)
ping -I veth-peer -c 10 192.168.99.1 &

# Also send traffic with a different source to populate multiple map entries
# (We'll use a second IP on the veth-peer)
sudo ip addr add 192.168.99.3/24 dev veth-peer 2>/dev/null
ping -I veth-peer -c 5 192.168.99.1 -s 64 &

wait
```

### Read the BPF Map (Packet Counts)

```bash
# Find the map ID
sudo bpftool map list
# Expected:
# 42: hash  name pkt_count  flags 0x0
#   key 4B  value 8B  max_entries 1024  memlock 8192B

MAP_ID=$(sudo bpftool map list | grep pkt_count | awk '{print $1}' | tr -d ':')
echo "Map ID: $MAP_ID"

# Dump all entries in the map
sudo bpftool map dump id $MAP_ID
# Expected (raw hex — we'll decode below):
# key: c0 a8 63 02  value: 0a 00 00 00 00 00 00 00
# key: c0 a8 63 03  value: 05 00 00 00 00 00 00 00

# Decode: key c0 a8 63 02 = 192.168.99.2, value 0a = 10 packets
# (network byte order: c0=192, a8=168, 63=99, 02=2)

# For human-readable output, write a simple decoder:
sudo bpftool map dump id $MAP_ID | \
  awk '/key/{
    split($2, a, " ")
    printf "src: %d.%d.%d.%d -> ", strtonum("0x"a[1]), strtonum("0x"a[2]),
           strtonum("0x"a[3]), strtonum("0x"a[4])
  } /value/{
    # value is little-endian u64
    printf "packets: %d\n", strtonum("0x"$2)
  }'
# Expected:
# src: 192.168.99.2 -> packets: 10
# src: 192.168.99.3 -> packets: 5
```

### Cleanup Lab 5A

```bash
sudo ip link set veth-test xdpgeneric off
sudo ip link del veth-test   # also removes veth-peer
sudo rm -f /sys/fs/bpf/xdp_counter
```

**Key Takeaway:** XDP programs run at the earliest point in the receive path — before SKB allocation, before netfilter. A BPF `HASH` map with key=src_ip stores per-IP state accessible from both kernel (XDP program) and userspace (`bpftool map dump`). The `__sync_fetch_and_add` is required because XDP runs per-CPU in parallel.

---

## Lab 5B: XDP Packet Drop (DDoS Simulation)

**Objective:** Extend the counter to selectively DROP packets from a blocked source IP, measure the performance advantage of XDP drop vs iptables DROP.

---

### The XDP Drop Program (C source)

```bash
cat > $LAB_DIR/xdp_drop.c << 'EOF'
// xdp_drop.c — drops packets from source IPs in a blocklist
// Uses BPF_MAP_TYPE_HASH as an IP blocklist

#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

// Blocklist map: key = source IP (u32), value = 1 (blocked)
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, __u32);
    __type(value, __u32);
    __uint(max_entries, 10000);
} blocklist SEC(".maps");

// Counter map: key = 0 (dropped), 1 (passed)
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, __u32);
    __type(value, __u64);
    __uint(max_entries, 2);
} drop_stats SEC(".maps");

SEC("xdp")
int xdp_drop_blocked(struct xdp_md *ctx)
{
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    if (eth->h_proto != bpf_htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    __u32 src_ip = ip->saddr;

    // Check if source IP is in blocklist
    __u32 *blocked = bpf_map_lookup_elem(&blocklist, &src_ip);
    if (blocked && *blocked == 1) {
        // Increment drop counter (key=0)
        __u32 key = 0;
        __u64 *dropped = bpf_map_lookup_elem(&drop_stats, &key);
        if (dropped)
            __sync_fetch_and_add(dropped, 1);
        return XDP_DROP;  // Drop before skb allocation — fastest possible drop
    }

    // Increment pass counter (key=1)
    __u32 key = 1;
    __u64 *passed = bpf_map_lookup_elem(&drop_stats, &key);
    if (passed)
        __sync_fetch_and_add(passed, 1);

    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
EOF

# Compile
clang -O2 -g \
  -target bpf \
  -D__TARGET_ARCH_x86 \
  -I /usr/include/$(uname -m)-linux-gnu \
  -c xdp_drop.c \
  -o xdp_drop.o

echo "Compiled xdp_drop.o"
```

### Setup: Load Program and Populate Blocklist

```bash
cd $LAB_DIR

# Recreate veth pair
sudo ip link add veth-test type veth peer name veth-peer
sudo ip link set veth-test up
sudo ip link set veth-peer up
sudo ip addr add 192.168.99.1/24 dev veth-peer
sudo ip addr add 192.168.99.2/24 dev veth-test

# Load and attach XDP drop program
sudo ip link set veth-test xdpgeneric obj xdp_drop.o sec xdp

# Find the blocklist map
sudo bpftool map list | grep blocklist
BLOCKLIST_ID=$(sudo bpftool map list | grep blocklist | awk '{print $1}' | tr -d ':')
echo "Blocklist map ID: $BLOCKLIST_ID"

# Add 192.168.99.3 to blocklist
# Key: IP in network byte order (192.168.99.3 = 0x03 0x63 0xa8 0xc0 in little-endian)
# Actually for little-endian arch: 192=0xc0, 168=0xa8, 99=0x63, 3=0x03
# In hex: c0 a8 63 03
BLOCKED_IP_HEX="c0 a8 63 03"
sudo bpftool map update id $BLOCKLIST_ID \
  key hex c0 a8 63 03 \
  value hex 01 00 00 00

echo "IP 192.168.99.3 added to blocklist"
```

### Simulate Attack Traffic

```bash
# Add the blocked IP to our interface
sudo ip addr add 192.168.99.3/24 dev veth-peer 2>/dev/null || true

# Send traffic FROM blocked IP (192.168.99.3)
# We simulate this by using hping3 or changing source with ip rule
# For simulation, we can use raw socket or just observe the map counter mechanism

# Send some traffic from the allowed IP (192.168.99.2) — should pass
ping -I veth-peer -c 5 192.168.99.2 -q

# The XDP program uses ip->saddr to check the blocklist
# Any packet with saddr=192.168.99.3 will be XDP_DROP'd

# Show drop statistics
STATS_ID=$(sudo bpftool map list | grep drop_stats | awk '{print $1}' | tr -d ':')
echo "=== Drop Statistics ==="
sudo bpftool map dump id $STATS_ID
# Expected:
# key: 00 00 00 00  value: <N> 00 00 00 00 00 00 00  (dropped packets)
# key: 01 00 00 00  value: <M> 00 00 00 00 00 00 00  (passed packets)
```

### Inspect Loaded Programs

```bash
# List all loaded BPF programs
sudo bpftool prog show
# Expected:
# 42: xdp  name xdp_drop_blocked  tag <hash>  gpl
#     loaded_at 2024-01-01T00:00:00+0000  uid 0
#     xlated 256B  jited 192B  memlock 4096B  map_ids 43,44

# Disassemble the JIT-compiled code (shows actual machine instructions)
sudo bpftool prog dump xlated id 42
# Expected: BPF bytecode instructions

# Show XDP attachment on the interface
ip link show veth-test
# Expected:
# veth-test: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
#     link/ether ... brd ...
#     xdpgeneric  id 42
```

### Performance Comparison: XDP Drop vs iptables DROP

```bash
# Measure baseline (no filtering)
sudo ip link set veth-test xdpgeneric off
# Time sending 1000 pings
time ping -I veth-peer -c 1000 192.168.99.1 -f -q
# Record: packets/second

# iptables DROP (re-attach XDP pass-through, add iptables rule)
sudo iptables -I INPUT -s 192.168.99.3 -j DROP
time ping -I veth-peer -c 1000 192.168.99.1 -f -q
sudo iptables -D INPUT -s 192.168.99.3 -j DROP

# XDP DROP (reattach XDP drop program)
sudo ip link set veth-test xdpgeneric obj xdp_drop.o sec xdp
time ping -I veth-peer -c 1000 192.168.99.1 -f -q

# Expected result:
# iptables DROP: ~500K pps, measurable CPU in softirq
# XDP DROP:      ~10M+ pps, minimal CPU (no SKB allocation, no netfilter traversal)
# XDP is 10-20x faster because it drops before the kernel allocates sk_buff

echo "XDP operates before SKB allocation — drops at NIC driver level"
echo "iptables operates after netfilter — requires full packet processing"
```

### Cleanup Lab 5B

```bash
sudo ip link set veth-test xdpgeneric off
sudo ip link del veth-test
```

**Key Takeaway:** `XDP_DROP` returns the packet buffer to the NIC before the kernel allocates an `sk_buff`. This means near-zero CPU cost per dropped packet. iptables DROP requires full netfilter traversal. For DDoS mitigation at scale (>1Mpps), XDP is the correct tool. For complex stateful filtering, iptables is more expressive.

---

## Lab 5C: bpftrace TCP Connection Tracing

**Objective:** Use bpftrace one-liners to trace TCP connection behavior in production without modifying application code or installing agents.

---

### Background

bpftrace is a high-level language for eBPF tracing. It attaches to kernel tracepoints and kprobes, executes BPF programs on each event, and aggregates data in maps. Zero application modification required.

### One-Liner 1: Trace New TCP Connections

```bash
# Trace all new outbound TCP connections with process name, src, dst
sudo bpftrace -e '
kprobe:tcp_v4_connect
{
    printf("%-16s %-6d -> ", comm, pid);
}

kretprobe:tcp_v4_connect
/ retval == 0 /
{
    $sk = (struct sock *)curtask->files->fdt->fd[0];
    printf("connected\n");
}
'
```

```bash
# Simpler and more practical — trace connect() syscall
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_connect
{
    printf("%-10s %-6d connect() called\n", comm, pid);
}
'
# In another terminal: curl http://example.com
# Expected output:
# curl       12345  connect() called
```

```bash
# Full version: show source IP, dest IP, dest port for TCP connections
sudo bpftrace -e '
#include <linux/socket.h>
#include <linux/in.h>

tracepoint:syscalls:sys_enter_connect
/ args->uservaddr->sa_family == AF_INET /
{
    $addr = (struct sockaddr_in *)args->uservaddr;
    printf("%-12s %-6d -> %s:%d\n",
        comm,
        pid,
        ntop(AF_INET, $addr->sin_addr.s_addr),
        (uint16)bswap($addr->sin_port));
}
'
# In another terminal: curl http://1.1.1.1
# Expected output:
# curl         12345  -> 1.1.1.1:80
```

### One-Liner 2: Count TCP Retransmits by Process

```bash
# This is invaluable for identifying which service is experiencing packet loss
sudo bpftrace -e '
kprobe:tcp_retransmit_skb
{
    @retransmits[comm] = count();
}

END
{
    printf("\nTCP Retransmits by process:\n");
    print(@retransmits);
    clear(@retransmits);
}
' &
BPFTRACE_PID=$!

# Generate some traffic that might cause retransmits
# (Normally these appear during network congestion or packet loss)
sleep 15

kill $BPFTRACE_PID 2>/dev/null

# Expected output:
# @retransmits[curl]: 3
# @retransmits[nginx]: 1
# @retransmits[prometheus]: 7
```

### One-Liner 3: DNS Query Tracer

```bash
# Trace processes making DNS queries (sending to port 53)
sudo bpftrace -e '
kprobe:udp_sendmsg
{
    $sk = (struct sock *)arg0;
    $dport = $sk->__sk_common.skc_dport;
    // bswap because port is stored in network byte order
    if ((uint16)bswap($dport) == 53) {
        printf("%-12s %-6d DNS query to port 53\n", comm, pid);
        @dns_queries[comm] = count();
    }
}
'
# In another terminal: dig google.com
# Expected output:
# dig          12345  DNS query to port 53
# systemd-r... 678    DNS query to port 53

# Also shows if unexpected processes are doing DNS lookups (e.g., malware)
```

### One-Liner 4: TCP Connection Latency Histogram

```bash
# Measure time from SYN sent to connection established (TCP handshake latency)
sudo bpftrace -e '
kprobe:tcp_v4_connect
{
    @start[tid] = nsecs;
}

kretprobe:tcp_v4_connect
/ @start[tid] /
{
    @conn_latency_us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}
'
# In another terminal, run several curl requests:
# for i in $(seq 1 20); do curl -s --max-time 2 http://1.1.1.1 > /dev/null; done
# Ctrl+C the bpftrace

# Expected output:
# @conn_latency_us:
# [1]                    1 |                                                    |
# [2, 4)                 2 |@                                                   |
# [4, 8)                12 |@@@@@@@@                                            |
# [8, 16)                3 |@@                                                  |
# [16, 32)               2 |@                                                   |
# Histogram shows RTT in microseconds — useful for detecting network degradation
```

### One-Liner 5: Short-Lived TCP Connections

```bash
# Find processes making many short-lived connections (connection storms, misconfigured retry)
sudo bpftrace -e '
kprobe:tcp_v4_connect        { @start[tid] = nsecs; }
kprobe:tcp_close
{
    if (@start[tid]) {
        $duration_ms = (nsecs - @start[tid]) / 1000000;
        if ($duration_ms < 100) {
            // Connection lasted less than 100ms — short-lived
            printf("SHORT-LIVED: %-12s %-6d lasted %dms\n",
                   comm, pid, $duration_ms);
        }
        delete(@start[tid]);
    }
}
'
# Expected output highlights processes with connection churn:
# SHORT-LIVED: curl         12345  lasted 45ms
# SHORT-LIVED: prometheus   678    lasted 23ms
```

### Summary: bpftrace Quick Reference

```bash
# Attach to tracepoints (preferred — stable ABI)
sudo bpftrace -e 'tracepoint:net:net_dev_xmit { @[comm] = count(); }'

# Attach to kprobes (less stable — kernel version dependent)
sudo bpftrace -e 'kprobe:tcp_sendmsg { @bytes[comm] = sum(arg2); }'

# List available tracepoints for networking
sudo bpftrace -l 'tracepoint:net:*'
sudo bpftrace -l 'tracepoint:sock:*'
sudo bpftrace -l 'tracepoint:tcp:*'

# List available kprobes (filter for tcp)
sudo bpftrace -l 'kprobe:tcp_*' | head -20
```

**Key Takeaway:** bpftrace is the best tool for answering "which process is doing X?" without modifying code. Key patterns: `kprobe:function { ... }` (on entry), `kretprobe:function / retval != 0 / { ... }` (on return, only errors), `@map[key] = count()` (aggregate), `hist(expr)` (histogram). Use tracepoints over kprobes when possible — they're stable across kernel versions.

---

## Lab 5D: Observe Kubernetes Traffic with Hubble

**Objective:** Use Hubble (Cilium's observability layer) to watch real-time Kubernetes network flows, observe NetworkPolicy drops, and identify blocked traffic.

---

### Prerequisites: Install Cilium + Hubble

```bash
# If you don't have a Cilium cluster, create one with kind + Cilium:
# Reference: https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/

# Install cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz"
sudo tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin
rm cilium-linux-amd64.tar.gz

# Install Hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all \
  "https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz"
sudo tar xzvf hubble-linux-amd64.tar.gz -C /usr/local/bin
rm hubble-linux-amd64.tar.gz

# Install Cilium in kind cluster (create cluster first)
cilium install --version 1.14.0

# Wait for Cilium to be ready
cilium status --wait
# Expected:
#     /¯¯\
#  /¯¯\__/¯¯\    Cilium:             OK
#  \__/¯¯\__/    Operator:           OK
#  /¯¯\__/¯¯\    Envoy DaemonSet:    disabled
#  \__/¯¯\__/    Hubble Relay:       OK
#     \__/        ClusterMesh:        disabled

# Enable Hubble with relay and UI
cilium hubble enable --ui
```

### Setup: Deploy Test Pods

```bash
# Create test namespace
kubectl create namespace hubble-lab

# Deploy two pods
kubectl apply -n hubble-lab -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: client
  labels:
    app: client
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Pod
metadata:
  name: server
  labels:
    app: server
spec:
  containers:
  - name: echo
    image: hashicorp/http-echo
    args: ["-text=hello-from-server"]
    ports:
    - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: server-svc
spec:
  selector:
    app: server
  ports:
  - port: 80
    targetPort: 5678
EOF

kubectl wait -n hubble-lab --for=condition=Ready pod/client pod/server --timeout=60s

# Port-forward Hubble relay for local CLI access
kubectl port-forward -n kube-system svc/hubble-relay 4245:80 &
HUBBLE_PF_PID=$!
sleep 2
```

### Observe Live Flows

```bash
# Watch all flows in real-time
hubble observe --namespace hubble-lab --follow

# In another terminal, generate some traffic
kubectl exec -n hubble-lab client -- curl -s http://server-svc/

# Expected hubble observe output:
# Feb  1 12:00:00.000 hubble-lab/client:54321 -> hubble-lab/server-svc:80  to-endpoint FORWARDED (TCP Flags: SYN)
# Feb  1 12:00:00.001 hubble-lab/server:5678  -> hubble-lab/client:54321   to-endpoint FORWARDED (TCP Flags: SYN, ACK)
# Feb  1 12:00:00.002 hubble-lab/client:54321 -> hubble-lab/server-svc:80  to-endpoint FORWARDED (TCP Flags: ACK)
# Feb  1 12:00:00.003 hubble-lab/client:54321 -> hubble-lab/server-svc:80  to-endpoint FORWARDED (HTTP/1.1 GET http://server-svc/)
# Feb  1 12:00:00.004 hubble-lab/server:5678  -> hubble-lab/client:54321   to-endpoint FORWARDED (HTTP/1.1 200 10ms)

# Useful filters:
# By source pod:
hubble observe --from-pod hubble-lab/client --follow

# By destination service:
hubble observe --to-service server-svc --namespace hubble-lab --follow

# By verdict:
hubble observe --verdict FORWARDED --namespace hubble-lab
hubble observe --verdict DROPPED --namespace hubble-lab

# By protocol:
hubble observe --protocol tcp --namespace hubble-lab
hubble observe --protocol dns --namespace hubble-lab  # see all DNS lookups!
```

### Apply a NetworkPolicy and Observe Drops

```bash
# Apply a deny-all policy for the server pod
kubectl apply -n hubble-lab -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-server
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
    - Ingress
  # No ingress rules = deny all ingress
EOF

# Attempt traffic (will be blocked)
kubectl exec -n hubble-lab client -- curl --max-time 3 http://server-svc/ || echo "BLOCKED (expected)"

# Observe the drop in Hubble
hubble observe --type drop --namespace hubble-lab
# Expected:
# Feb  1 12:01:00.000 hubble-lab/client:54322 -> hubble-lab/server:5678  to-endpoint DROPPED
#   (cilium_call_policy DROP NetworkPolicy)

# The key field: "NetworkPolicy" tells you EXACTLY which mechanism dropped it
# Compare to "XDP_DROP" which shows XDP drops, "NodeFirewall DROP" for node-level policy

# Get more detail on dropped flows
hubble observe --type drop --namespace hubble-lab --output json | \
  python3 -m json.tool | grep -E "drop_reason|source|destination"
# Expected:
# "drop_reason": "Policy denied",
# "source": "hubble-lab/client",
# "destination": "hubble-lab/server"
```

### Hubble CLI Reference

```bash
# Show Hubble status and connected nodes
hubble status
# Expected:
# Healthcheck (via localhost:4245): Ok
# Current/Max Flows: 4096/4096 (100.00%)
# Flows/s: 10.84
# Connected Nodes: 3/3

# Show flows for a specific pod
hubble observe --pod hubble-lab/client --follow

# Show flows with full L7 info (HTTP, DNS, Kafka)
hubble observe --protocol http --namespace hubble-lab
hubble observe --protocol dns --namespace hubble-lab
# Expected DNS flows:
# client -> kube-dns UDP FORWARDED (DNS Query A server-svc.hubble-lab.svc.cluster.local.)
# kube-dns -> client UDP FORWARDED (DNS Answer A 10.96.x.x)

# Count flows per source
hubble observe --namespace hubble-lab --output json | \
  python3 -c "
import sys, json
from collections import Counter
flows = [json.loads(l) for l in sys.stdin if l.strip()]
srcs = Counter(f.get('source','?') for f in flows)
for src, cnt in srcs.most_common(10):
    print(f'{cnt:5d}  {src}')
"

# Run connectivity test (Cilium built-in)
cilium connectivity test
# Tests pod-to-pod, pod-to-service, egress, ingress across the full cluster
```

### Cleanup

```bash
kill $HUBBLE_PF_PID 2>/dev/null
kubectl delete namespace hubble-lab
```

**Key Takeaway:** Hubble provides L3-L7 observability for every packet in a Cilium cluster. The key use case: "Traffic is being dropped — is it NetworkPolicy, XDP, or something else?" Hubble's `--type drop` with `--output json` shows the exact drop reason. For DNS debugging, `--protocol dns` shows all DNS queries cluster-wide — invaluable for ndots latency analysis.

---

## Summary: eBPF/XDP Lab Checklist

| Lab  | Core Skill | Production Scenario |
|------|-----------|---------------------|
| 5A   | XDP program + BPF map | Per-IP traffic accounting without iptables |
| 5B   | XDP DROP + performance | DDoS mitigation — drop at line rate before kernel processing |
| 5C   | bpftrace one-liners | "Which process is causing TCP retransmits?" "Who is hammering DNS?" |
| 5D   | Hubble observability | "Why is pod-A dropping traffic?" in Kubernetes |

## Interview Discussion Points

1. "What is XDP and how does it differ from iptables?" — XDP runs before SKB allocation (NIC driver level), iptables runs after netfilter (kernel stack). XDP is 10-20x faster for drop rules, but cannot do complex stateful decisions.
2. "How would you trace which process is making the most DNS queries without modifying code?" — `bpftrace -e 'kprobe:udp_sendmsg { if port==53 { @[comm]=count(); } }'`
3. "A NetworkPolicy is applied but you're not sure if it's working — how do you verify?" — `hubble observe --type drop --namespace X` shows exactly which flows are being dropped and why.
4. "What's a BPF map?" — A kernel data structure shared between BPF programs and userspace. Types: HASH (arbitrary key-value), ARRAY (fixed-index), LRU_HASH (auto-evicts LRU entries), PERF_EVENT_ARRAY (streaming events to userspace), RINGBUF (preferred over PERF for modern kernels).

## eBPF Architecture Reference

```
 Userspace          |  Kernel
 ---------------    |  ----------------------------------
 bpftool            |
 bpftrace           |  tracepoints / kprobes / XDP hooks
 libbpf             |       |
       |            |  eBPF verifier (safety check)
       |            |       |
   BPF syscall <----|--  JIT compiler (arch-specific code)
       |            |       |
   BPF maps  <------|--  BPF program (runs at hook point)
                    |
```

The verifier ensures: no unbounded loops, no invalid memory accesses, no kernel crashes. This is why eBPF programs are "safe" to load into the kernel at runtime.
