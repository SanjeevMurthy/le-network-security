# Network & Security Command Cheatsheet

Production-grade command reference for Senior SREs. Extracted and synthesized from the full knowledge base (sections 00–11), covering 51 tools across Linux networking, packet analysis, DNS, TLS, firewalls, eBPF, Kubernetes, and cloud CLIs.

Each part is also available as a standalone file under [`12-command-cheatsheet/`](./12-command-cheatsheet/):

| Part | File | Tools |
|------|------|-------|
| Part 1 — Linux Network Core | [01-linux-network-core.md](./12-command-cheatsheet/01-linux-network-core.md) | `ip`, `ss`, `tc`, `ethtool`, `bridge`, `conntrack`, `nstat`, `sysctl`, `ping`, `mtr`, `traceroute`, `arp`, `nc`, `socat`, `nmap`, `iperf3`, `nsenter` |
| Part 2 — Capture / DNS / HTTP / TLS / Firewall | [02-capture-dns-http-tls-firewall.md](./12-command-cheatsheet/02-capture-dns-http-tls-firewall.md) | `tcpdump`, `tshark`, `dig`, `nslookup`, `host`, `resolvectl`, `curl`, `wget`, `openssl`, `iptables`, `ip6tables`, `nft`, `ipset` |
| Part 3 — eBPF / Kubernetes / Cloud / Container | [03-ebpf-kubernetes-cloud-container.md](./12-command-cheatsheet/03-ebpf-kubernetes-cloud-container.md) | `bpftool`, `bpftrace`, `perf`, `strace`, `lsof`, `kubectl`, `istioctl`, `cilium`, `hubble`, `aws`, `az`, `docker`, `crictl`, `dmesg`, `journalctl`, `systemctl`, `vtysh`, `trivy`, `vault` |

---

## How to Use This Cheatsheet

- **Finding a command**: Use your editor's search (`/tool-name`) or jump via the TOC below
- **Quick copy**: All code blocks are self-contained and runnable
- **Flag tables**: Each tool lists the most important options — not exhaustive, but production-relevant
- **Expected output**: Abbreviated realistic output shown under each example

---

## Table of Contents

### Part 1 — Linux Network Core

| Tool | Purpose |
|------|---------|
| [`ip`](#ip--network-interface-routing-and-policy-management) | Interface, address, route, neigh, rule, netns, tunnel, VRF |
| [`ss`](#ss--socket-statistics) | Active connections, listening ports, TCP internals |
| [`tc`](#tc--traffic-control) | Queueing disciplines, rate limiting, netem, BPF filters |
| [`ethtool`](#ethtool--nic-configuration-and-statistics) | NIC settings, ring buffers, offloads, RSS |
| [`bridge`](#bridge--bridge-management) | FDB, VLAN, STP state, VXLAN VTEPs |
| [`conntrack`](#conntrack--connection-tracking) | NAT table, flow stats, insert failures |
| [`nstat` / `netstat`](#nstat--netstat--network-statistics) | Kernel counters, retransmits, listen overflow |
| [`sysctl`](#sysctl--kernel-network-parameters) | Kernel tuning: buffers, conntrack, routing |
| [`ping` / `ping6`](#ping--ping6--icmp-connectivity-test) | Reachability, PMTU probing |
| [`mtr`](#mtr--combined-traceroute--ping) | Per-hop loss and latency |
| [`traceroute` / `tracepath`](#traceroute--tracepath--path-tracing) | UDP/TCP/ICMP path trace, MTU discovery |
| [`arp` / `arping`](#arp--arping--arp-table-management) | ARP table, gratuitous ARP, duplicate detection |
| [`nc`](#nc--netcat---network-swiss-army-knife) | Port scan, listener, banner grab, file transfer |
| [`socat`](#socat--advanced-socket-relay) | Port forward, TLS probe, UNIX socket relay |
| [`nmap`](#nmap--network-scanner) | Host discovery, port scan, service/TLS enumeration |
| [`iperf3`](#iperf3--bandwidth-testing) | TCP/UDP throughput, parallel streams |
| [`nsenter`](#nsenter--enter-process-namespaces) | Enter container network namespace |
| [`ip netns exec`](#ip-netns-exec--network-namespace-execution) | Execute commands in named namespaces |

### Part 2 — Packet Capture, DNS, HTTP, TLS & Firewall

| Tool | Purpose |
|------|---------|
| [`tcpdump`](#tcpdump--packet-capture-and-analysis) | BPF filter capture, pcap write/read |
| [`tshark`](#tshark--cli-wireshark) | Protocol dissection, field extraction, statistics |
| [`dig`](#dig--dns-lookup) | DNS queries, trace, DNSSEC, reverse lookup |
| [`nslookup`](#nslookup--dns-query-tool) | Interactive DNS queries, debug mode |
| [`host`](#host--simple-dns-lookup) | Quick DNS resolution, all record types |
| [`resolvectl`](#resolvectl--systemd-resolved-query-tool) | systemd-resolved status, flush, monitor |
| [`curl`](#curl--http--https-client) | HTTP testing, TLS debug, timing, mTLS |
| [`wget`](#wget--file-download-tool) | Download, spider, resume, rate-limit |
| [`openssl`](#openssl--tls--certificate-toolkit) | s_client, x509, req, verify, genrsa |
| [`iptables`](#iptables--ipv4-packet-filtering) | Filter, NAT, conntrack states, DNAT/SNAT |
| [`ip6tables`](#ip6tables--ipv6-packet-filtering) | IPv6 filtering, ICMPv6, NDP |
| [`nft`](#nft--nftables---modern-netfilter) | Tables, sets, NAT, atomic ruleset load |
| [`ipset`](#ipset--ip-set-management) | IP/net sets, timed entries, blocklist swap |

### Part 3 — eBPF, Kubernetes, Cloud & Container

| Tool | Purpose |
|------|---------|
| [`bpftool`](#bpftool--bpf-program--map-inspection) | Prog/map list, BTF, CO-RE, net attach |
| [`bpftrace`](#bpftrace--dynamic-tracing) | TCP retransmit, DNS latency, drop reasons |
| [`perf`](#perf--linux-performance-profiling) | CPU perf, network events, flamegraphs |
| [`strace`](#strace--system-call-tracer) | Syscall trace, port exhaustion, BPF debug |
| [`lsof`](#lsof--list-open-filessockets) | Listening sockets, per-process connections |
| [`kubectl`](#kubectl--kubernetes-cli-networking) | Pods, services, endpoints, NetworkPolicy, exec |
| [`istioctl`](#istioctl--istio-service-mesh-cli) | proxy-status, proxy-config, authz, analyze |
| [`cilium`](#cilium--cilium-cni-cli) | Status, monitor, BPF policy/lb/ct, connectivity |
| [`hubble`](#hubble--cilium-observability) | Flow observation, drops, DNS, HTTP flows |
| [`aws`](#aws--aws-cli-networking) | VPC, SG, ELB, Route53, TGW, EKS |
| [`az`](#az--azure-cli-networking) | VNet, NSG, Network Watcher, AKS |
| [`docker`](#docker--container-networking) | Network ls/inspect/create, exec, PID |
| [`crictl`](#crictl--container-runtime-cli) | ps, inspect, exec, logs for containerd/CRI-O |
| [`dmesg`](#dmesg--kernel-ring-buffer) | Conntrack overflow, SYN flood, NIC errors |
| [`journalctl`](#journalctl--systemd-log-query) | Kubelet CNI errors, kernel messages, time range |
| [`systemctl`](#systemctl--service-management) | Network service status, restart, failed units |
| [`vtysh`](#vtysh--frrouting-cli) | BGP summary/neighbors, OSPF, RPKI, BFD |
| [`trivy`](#trivy--container-vulnerability-scanner) | Image, fs, k8s, config scan, secret detection |
| [`vault`](#vault--hashicorp-vault-cli) | PKI, KV, Kubernetes auth, lease management |

---


---

# Part 1 — Linux Network Core

## `ip` — Swiss army knife for network interfaces, routing, and namespaces

**When to use:** Primary tool for all interface, address, route, neighbor, and namespace operations on modern Linux. Replaces `ifconfig`, `route`, `arp`, and related legacy tools.

---

### `ip link` — Interface management

#### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `show [dev DEV]` | Show interface state, MTU, flags, and driver details |
| `-s link show DEV` | Show TX/RX byte and error counters |
| `-d link show DEV` | Show detailed/driver-level info (VXLAN VNI, mode, etc.) |
| `set DEV up\|down` | Bring interface up or down |
| `set DEV mtu N` | Set MTU on interface |
| `set DEV master BR` | Attach interface to a bridge |
| `set DEV xdp obj F.o sec xdp` | Attach XDP program to interface |
| `set DEV xdp off` | Detach XDP program |
| `add NAME type TYPE` | Create virtual interface (veth, vxlan, bridge, etc.) |
| `del NAME` | Delete a virtual interface |
| `show type TYPE` | Filter by type: vrf, bridge, vxlan, geneve, macvlan, etc. |

#### Common Usage

```bash
# Show all interfaces
ip link show

# Show interface with counters
ip -s link show eth0
```

**Output:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP ...
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped  missed  mcast
    1234567890 9876543  0       0        0       0
    TX: bytes  packets  errors  dropped  carrier collsns
    987654321  7654321  0       0        0       0
```

#### Important Variants

```bash
# Create veth pair for container/namespace networking
ip link add veth0 type veth peer name veth1

# Create VXLAN tunnel interface
ip link add vxlan10 type vxlan id 10 dstport 4789 dev eth0

# Create TUN (L3) interface for VPN
ip tuntap add dev tun0 mode tun user $(whoami)

# Create TAP (L2) interface for VM bridging
ip tuntap add dev tap0 mode tap

# Create macvlan in bridge mode
ip link add macvlan0 link eth0 type macvlan mode bridge

# Create ipvlan in L3 mode (no ARP)
ip link add ipvlan0 link eth0 type ipvlan mode l3

# Attach XDP program in native mode (fails if driver unsupported)
ip link set eth0 xdp obj xdp_drop.o sec xdp mode native

# Attach XDP program in generic (skb) mode — works on any NIC
ip link set eth0 xdp obj xdp_drop.o sec xdp mode skbmode

# Check XDP attachment status
ip link show eth0

# Show VXLAN interfaces with VNI and remote details
ip -d link show type vxlan

# Show interfaces attached to bridge br0
ip link show master br0

# Show all bridge interfaces on system
ip link show type bridge
```

---

### `ip addr` — Address management

#### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `show [dev DEV]` | Show addresses on all or one interface |
| `add CIDR dev DEV` | Add IPv4/IPv6 address |
| `del CIDR dev DEV` | Remove address |
| `flush dev DEV` | Remove all addresses from an interface |

#### Common Usage

```bash
# Show all addresses
ip addr show

# Add address to interface
ip addr add 10.0.0.10/24 dev eth1
```

**Output:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 10.0.0.10/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::1/64 scope link
```

#### Important Variants

```bash
# Assign /31 point-to-point address
ip addr add 10.0.0.0/31 dev eth1

# Show address inside a namespace
ip netns exec myns ip addr
```

---

### `ip route` — Route management

#### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `show` | Show main routing table |
| `show table TABLE` | Show a specific table (main, local, default, or number) |
| `show table all` | Show routes in all tables |
| `show root PREFIX` | Show routes matching a prefix |
| `get DEST` | Show which route/interface will be used for DEST |
| `get DEST from SRC` | Route lookup with source IP (RPDB-aware) |
| `add PREFIX via GW` | Add route with nexthop |
| `add PREFIX dev DEV` | Add on-link route |
| `add PREFIX blackhole` | Drop traffic silently |
| `add PREFIX via GW metric N` | Add route with explicit metric |
| `del PREFIX` | Delete route |
| `add default via GW` | Add default route |
| `monitor` | Watch route changes in real time |
| `add ... table T` | Add route to a custom policy table |

#### Common Usage

```bash
# Show main routing table
ip route show

# Check which route is selected for a destination
ip route get 8.8.8.8
```

**Output:**
```
default via 10.0.0.1 dev eth0 proto dhcp src 10.0.0.42 metric 100
10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.42
10.244.0.0/16 via 10.0.0.1 dev eth0  # pod CIDR route

# ip route get 8.8.8.8
8.8.8.8 via 10.0.0.1 dev eth0 src 10.0.0.42 uid 0
    cache
```

#### Important Variants

```bash
# RPDB-aware route lookup with source IP
ip route get 8.8.8.8 from 10.0.0.5

# ECMP route lookup simulating a specific 5-tuple
ip route get 10.5.5.5 from 192.168.1.100 sport 45678 dport 80 proto tcp

# Add ECMP route with two equal-weight nexthops
ip route add 10.0.0.0/8 nexthop via 192.168.1.1 dev eth0 weight 1 nexthop via 192.168.2.1 dev eth1 weight 1

# Add route to a custom VRF table
ip route add 198.51.100.0/24 dev eth0 table isp_a
ip route add default via 198.51.100.1 table isp_a

# Add default route within a VRF
ip route add 0.0.0.0/0 via 10.0.0.254 vrf vrf-mgmt

# Show routes in a VRF
ip route show vrf vrf-mgmt

# Watch route changes in real time
ip route monitor

# Add blackhole route to discard an entire prefix
ip route add 192.168.0.0/16 blackhole

# Check pod CIDR routes (Kubernetes)
ip route show table main | grep 10.244
```

---

### `ip neigh` — Neighbor (ARP/NDP) cache

#### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `show` | Show ARP/NDP neighbor table |
| `-6 neigh show` | Show IPv6 NDP neighbors |
| `add IP lladdr MAC nud permanent dev DEV` | Add static permanent neighbor |
| `del IP dev DEV` | Delete a neighbor entry |
| `flush all` | Flush entire neighbor cache |
| `flush dev DEV` | Flush neighbors on a specific interface |

#### Common Usage

```bash
# Show ARP table
ip neigh show

# Monitor ARP table changes live
watch -n 1 'ip neigh show'
```

**Output:**
```
10.0.0.1 dev eth0 lladdr 02:42:ac:11:00:01 REACHABLE
10.0.0.50 dev eth0 lladdr 02:42:ac:11:00:32 STALE
10.0.0.99 dev eth0  FAILED
```

```bash
# Show IPv6 NDP cache
ip -6 neigh show

# Add static ARP entry
ip neigh add 10.0.0.100 lladdr 02:00:00:00:00:01 nud permanent dev eth0

# Check ARP entry for a VIP
ip neigh show | grep 10.0.1.100

# Flush cache after failover
ip neigh flush all
```

---

### `ip rule` — Routing Policy Database (RPDB)

**When to use:** Inspect or manage policy rules that select which routing table to use, based on source IP, destination, firewall mark, or interface.

#### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `show` | List RPDB rules in priority order |
| `add from SRC table T` | Route traffic from source IP via table T |
| `add fwmark MARK table T` | Route by firewall mark |
| `add to DST table T` | Route by destination prefix |
| `del ...` | Delete a rule |
| `priority N` | Explicit priority (lower = evaluated first) |

#### Common Usage

```bash
# Show all RPDB rules
ip rule show
```

**Output:**
```
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default
```

```bash
# Source-based routing: route ISP-A source IPs via ISP-A table
ip rule add from 198.51.100.10 table isp_a

# Route by firewall mark (more robust than source IP)
ip rule add fwmark 0x10 table monitoring

# Verify the routing result
ip route get 8.8.8.8 from 198.51.100.10

# Fix overly broad rule (replace with /32)
ip rule del from 10.0.0.0/8 table monitoring
ip rule add from 10.0.5.10/32 table monitoring
```

---

### `ip netns` — Network namespace lifecycle

**When to use:** Create, list, delete, and execute commands in named network namespaces.

#### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `add NAME` | Create named namespace |
| `list` | List named namespaces |
| `del NAME` | Delete namespace |
| `exec NAME CMD` | Execute command inside namespace |
| `identify PID` | Show namespace name for a PID |

#### Common Usage

```bash
# Create and wire up a namespace
ip netns add myns
ip link add veth-host type veth peer name veth-ns
ip link set veth-ns netns myns
ip netns exec myns ip addr add 10.100.0.2/24 dev veth-ns
ip netns exec myns ip link set veth-ns up
ip netns exec myns ip link set lo up
ip netns exec myns ip route add default via 10.100.0.1
```

```bash
# List namespaces
ip netns list

# Execute command inside namespace
ip netns exec myns ip link show

# Inspect pod CNI namespace (Kubernetes)
ip netns exec cni-abc12345 ip addr
ip netns exec cni-abc12345 ss -tlnp

# Delete namespace
ip netns del myns
```

---

### `ip tunnel` — GRE/IPIP/SIT tunnel management

**When to use:** Create and manage L3 tunnels (GRE, IPIP, SIT, ip6tnl).

```bash
# Create GRE tunnel
ip tunnel add gre1 mode gre local 192.168.1.1 remote 192.168.2.1 ttl 64

# Bring up the tunnel
ip link set gre1 up
ip addr add 10.10.10.1/30 dev gre1

# Show all tunnels
ip tunnel show

# Delete tunnel
ip tunnel del gre1
```

---

### `ip vrf` — VRF management

**When to use:** Isolate routing domains on a single host (management plane separation, multi-tenant).

```bash
# Create VRF and assign interface
ip link add vrf-mgmt type vrf table 100
ip link set vrf-mgmt up
ip link set eth0 master vrf-mgmt
ip route add 0.0.0.0/0 via 10.0.0.254 vrf vrf-mgmt

# Show all VRF devices
ip link show type vrf

# Show routes in VRF
ip route show vrf vrf-mgmt

# Run command in VRF context
ip vrf exec vrf-mgmt ping 10.0.0.254
```

---

### `ip nexthop` — Resilient ECMP nexthop groups (kernel 5.12+)

**When to use:** Create named nexthop objects and resilient ECMP groups that survive single nexthop failures without disrupting existing flows.

```bash
# Create individual nexthop objects
ip nexthop add id 1 via 192.168.1.1 dev eth0
ip nexthop add id 2 via 192.168.2.1 dev eth1

# Create resilient group with 128 buckets
ip nexthop add id 10 group 1/2 type resilient buckets 128 idle_timer 60 unbalanced_timer 300

# Assign to route
ip route add 10.0.0.0/8 nhid 10

# Inspect group
ip nexthop show id 10
ip nexthop bucket show id 10

# Monitor nexthop changes (BFD/failover environments)
ip nexthop monitor
```

---

### `ip mroute` — Multicast routing

```bash
# Show multicast routing cache
ip mroute show

# Show multicast routing statistics
ip mroute show table all

# Flush multicast route cache
ip mroute flush
```

---

## `ss` — Socket statistics

**When to use:** Inspect listening and connected sockets, TCP state distribution, internal TCP metrics (RTT, cwnd, retransmits). Faster than `netstat` and reads directly from kernel.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-l` | Listening sockets only |
| `-n` | No name resolution (faster) |
| `-p` | Show process name and PID |
| `-s` | Summary statistics by socket family and state |
| `-i` | Show internal TCP info (RTT, cwnd, ssthresh) |
| `-m` | Show socket memory usage (Recv-Q, Send-Q) |
| `-e` | Extended socket info (uid, inode, cookie) |
| `-a` | All sockets (listening + non-listening) |
| `state STATE` | Filter by TCP state (e.g., `established`, `time-wait`, `close-wait`) |
| `dst IP` | Filter by destination IP |
| `src IP:PORT` | Filter by source |
| `dport :PORT` | Filter by destination port |

### Common Usage

```bash
# All listening TCP sockets with process info
ss -tnlp
```

**Output:**
```
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
LISTEN  0       128     0.0.0.0:8080        0.0.0.0:*          users:(("nginx",pid=1234,fd=6))
LISTEN  0       4096    0.0.0.0:443         0.0.0.0:*          users:(("nginx",pid=1234,fd=7))
```

```bash
# Socket summary (great first-look in incident triage)
ss -s
```

**Output:**
```
Total: 512
TCP:   498 (estab 450, closed 30, orphaned 2, timewait 16)

Transport  Total  IP   IPv6
RAW        0      0    0
UDP        8      5    3
TCP        468    380  88
INET       476    385  91
```

### Important Variants

```bash
# TCP connection internals: RTT, cwnd, retransmits
ss -ti

# Show TCP internals to a specific destination
ss -ti dst 10.0.1.100

# Count active connections to a port (detect exhaustion)
ss -tnp | grep 8080 | wc -l

# Show state distribution (detect TIME_WAIT / CLOSE_WAIT accumulation)
ss -tnp | awk '{print $2}' | sort | uniq -c | sort -rn | head

# List all CLOSE_WAIT sockets with process
ss -tanp state close-wait

# Count CLOSE_WAIT by process
ss -tanp state close-wait | awk '{print $NF}' | sort | uniq -c | sort -rn

# List TIME_WAIT connections
ss -tn state time-wait | head

# Count TIME_WAIT
ss -s | grep "time wait"

# Sockets with non-empty receive queues (app not reading fast enough)
ss -tnmp | grep -v "Recv-Q:0\|Recv-Q 0"

# Check listen queue depth vs backlog (accept overflow)
ss -ltn | grep 8080

# Show UDP sockets
ss -u -a

# Check connections to specific destination
ss -tn dst 10.0.1.100 | wc -l

# Ephemeral port usage distribution
ss -tn | awk '{print $4}' | awk -F: '{print $NF}' | sort -n | uniq -c | sort -rn | head

# Show SYN_RECV count (confirm SYN flood)
ss -s | grep -i "SYN_RECV"
```

---

## `tc` — Traffic control (qdisc, class, filter)

**When to use:** Shape, schedule, and police traffic on egress (and ingress via IFB). Essential for bandwidth management, AQM, netem testing, and attaching TC-BPF programs.

### Key Options

| Subcommand | Description |
|------------|-------------|
| `qdisc show dev DEV` | Show active qdisc on interface |
| `-s qdisc show dev DEV` | Show qdisc with drop/byte statistics |
| `qdisc replace dev DEV root TYPE` | Set or replace root qdisc |
| `qdisc add dev DEV root handle H: TYPE` | Add root qdisc with handle |
| `qdisc del dev DEV root` | Remove root qdisc (resets to pfifo_fast) |
| `class show dev DEV` | Show all HTB/HFSC classes |
| `class add dev DEV parent P classid C htb rate R ceil C` | Add HTB class |
| `filter show dev DEV` | Show all filters |
| `filter add dev DEV parent P protocol ip u32 match ...` | Add u32 classifier |
| `monitor` | Watch qdisc/class/filter changes in real time |

### Common Usage

```bash
# Show current qdisc on interface
tc qdisc show dev eth0
```

**Output:**
```
qdisc mq 0: root
qdisc fq_codel 0: parent :1 limit 10240p flows 1024 quantum 1514 target 5ms interval 100ms memory_limit 32Mb ecn drop_batch 64
```

```bash
# Show qdisc stats (drops, bytes processed)
tc -s qdisc show dev eth0
```

### Important Variants

```bash
# AQM: fq_codel with tuned target latency
tc qdisc replace dev eth0 root fq_codel target 5ms interval 100ms quantum 1514 limit 10240

# Simple rate limit with Token Bucket Filter
tc qdisc add dev eth0 root tbf rate 10mbit burst 32kbit latency 400ms

# Fair queuing (required for BBR TCP pacing)
tc qdisc replace dev eth0 root fq

# HTB hierarchical bandwidth control
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: classid 1:1 htb rate 1gbit
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 500mbit ceil 800mbit burst 15k
tc qdisc add dev eth0 parent 1:10 handle 10: fq_codel
tc filter add dev eth0 parent 1: protocol ip u32 match ip src 10.244.1.0/24 flowid 1:10

# netem: network impairment simulation
tc qdisc add dev eth0 root netem delay 100ms
tc qdisc add dev eth0 root netem delay 100ms 20ms distribution normal  # with jitter
tc qdisc add dev eth0 root netem loss 1%                                 # packet loss
tc qdisc add dev eth0 root netem delay 50ms 10ms loss 0.5%              # combined
tc qdisc add dev eth0 root netem corrupt 0.1%                            # corruption
tc qdisc add dev eth0 root netem delay 100ms reorder 25% gap 5          # reordering

# Ingress shaping via IFB (Intermediate Functional Block)
ip link add ifb0 type ifb && ip link set ifb0 up
tc qdisc add dev eth0 handle ffff: ingress
tc filter add dev eth0 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0
# Now apply shaping qdiscs to ifb0

# TC-BPF program attachment
tc qdisc add dev eth0 clsact
tc filter add dev eth0 ingress bpf da obj my_tc.o sec tc_ingress
tc filter add dev eth0 egress bpf da obj my_tc.o sec tc_egress
tc filter show dev eth0 ingress
tc -s filter show dev eth0 ingress

# Remove TC-BPF
tc filter del dev eth0 ingress
tc qdisc del dev eth0 clsact

# BBR setup: fair queue + BBR congestion control
tc qdisc replace dev eth0 root fq
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

---

## `ethtool` — NIC configuration and statistics

**When to use:** Inspect NIC hardware state, tune ring buffers, interrupt coalescing, RSS queues, and offload features. Essential for diagnosing L1/L2 drops.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `ethtool DEV` | Show link speed, duplex, auto-negotiation, and port type |
| `-i DEV` | Show driver name and firmware version |
| `-S DEV` | Show all driver statistics counters |
| `-g DEV` | Show current and maximum ring buffer sizes |
| `-G DEV rx N tx N` | Set ring buffer sizes |
| `-l DEV` | Show current and maximum number of channels (queues) |
| `-L DEV combined N` | Set number of combined channels for RSS |
| `-x DEV` | Show RSS hash indirection table |
| `-X DEV equal N` | Distribute RSS evenly across N queues |
| `-n DEV rx-flow-hash tcp4` | Show hash fields used for TCP/IPv4 RSS |
| `-k DEV` | Show offload feature state (gro, gso, tso, etc.) |
| `-K DEV FEATURE on\|off` | Enable or disable offload feature |
| `-c DEV` | Show interrupt coalescing settings |
| `-C DEV rx-usecs N rx-frames N` | Set coalescing parameters |
| `-A DEV autoneg on\|off` | Set auto-negotiation |
| `-s DEV speed N duplex full` | Force link speed and duplex |

### Common Usage

```bash
# Basic NIC state
ethtool eth0
```

**Output:**
```
Settings for eth0:
    Speed: 10000Mb/s
    Duplex: Full
    Auto-negotiation: on
    Port: Twisted Pair
    Link detected: yes
```

```bash
# Check for NIC-level packet drops
ethtool -S eth0 | grep -E -i '(drop|miss|error|discard|overflow|lost)'
```

### Important Variants

```bash
# Identify driver (ixgbe/mlx5 = AF_XDP zero-copy capable)
ethtool -i eth0 | grep driver

# Find veth peer interface index
ethtool -S veth0 | grep peer_ifindex

# Show ring buffer sizes
ethtool -g eth0

# Increase ring buffers to absorb traffic bursts
ethtool -G eth0 rx 4096 tx 4096

# Show number of queues
ethtool -l eth0

# Set 16 queues for RSS parallelism
ethtool -L eth0 combined 16

# Show RSS indirection table
ethtool -x eth0

# Evenly distribute RSS across 16 queues
ethtool -X eth0 equal 16

# Check what fields are hashed for RSS
ethtool -n eth0 rx-flow-hash tcp4

# View interrupt coalescing settings
ethtool -c eth0

# High-throughput coalescing (batch more packets per interrupt)
ethtool -C eth0 rx-usecs 100 rx-frames 128

# Low-latency coalescing (interrupt on fewer packets)
ethtool -C eth0 rx-usecs 10 rx-frames 8

# Adaptive coalescing (NIC adjusts automatically)
ethtool -C eth0 adaptive-rx on adaptive-tx on

# Ultra-low latency: interrupt on every single packet (high CPU)
ethtool -C eth0 rx-usecs 0 rx-frames 1

# Enable/disable GRO
ethtool -K eth0 gro on
ethtool -K eth0 gro off

# Live monitor of NIC drop counters
watch -n 1 "ethtool -S eth0 | grep -E '(miss|drop|overflow)'"
```

---

## `bridge` — Bridge management

**When to use:** Inspect Linux software bridges — MAC forwarding tables, VLAN assignments, port states, and STP status.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `fdb show` | Show forwarding database (all learned MACs) |
| `fdb show dev DEV` | Filter FDB entries to a specific port/interface |
| `fdb add MAC dev DEV` | Add static FDB entry |
| `fdb del MAC dev DEV` | Delete FDB entry |
| `vlan show` | Show VLAN membership on bridge ports |
| `vlan add vid VID dev DEV` | Add VLAN to bridge port |
| `link show` | Show bridge port states and STP role/state |
| `monitor` | Watch FDB and VLAN changes in real time |

### Common Usage

```bash
# Show MAC table (FDB) for all bridge interfaces
bridge fdb show
```

**Output:**
```
02:42:ac:11:00:02 dev veth0 master docker0
33:33:00:00:00:01 dev docker0 self permanent
```

```bash
# Show VLAN assignments on bridge ports
bridge vlan show

# Show bridge port state (STP roles)
bridge link show
```

### Important Variants

```bash
# Show MAC addresses on Docker bridge
bridge fdb show dev docker0

# Show VTEP MAC/IP entries for Flannel VXLAN
bridge fdb show dev flannel.1

# Show L2 forwarding database for a lab bridge
bridge fdb show dev br0

# Add a static FDB entry (e.g., for VTEP)
bridge fdb add 00:11:22:33:44:55 dev eth0 dst 10.0.0.50

# Watch for FDB changes (useful during VXLAN failover)
bridge monitor fdb
```

---

## `conntrack` — Connection tracking

**When to use:** Inspect and manipulate the netfilter connection tracking table. Critical for diagnosing NAT issues, conntrack exhaustion, DNS timeout bugs (insert_failed), and TIME_WAIT buildup.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-L` | List all conntrack entries |
| `-L -o extended` | List with extended (state-level) fields |
| `-S` | Show per-CPU statistics (drops, insert_failed) |
| `-C` | Print total count of tracked connections |
| `-E` | Watch conntrack events in real time |
| `-D -p proto --src IP --dst IP` | Delete a specific entry |
| `-F` | Flush the entire conntrack table |

### Common Usage

```bash
# List all tracked connections
conntrack -L
```

**Output:**
```
tcp      6 86399 ESTABLISHED src=10.0.0.5 dst=8.8.8.8 sport=54321 dport=443 ...
udp      17 29 src=10.0.0.5 dst=10.0.0.1 sport=53201 dport=53 ...
```

```bash
# Check table statistics — key for diagnosing drops
conntrack -S
```

**Output:**
```
cpu=0 found=12 invalid=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0
cpu=1 found=8  invalid=0 insert=0 insert_failed=3 drop=0 early_drop=0 error=0 ...
```

### Important Variants

```bash
# Count ESTABLISHED entries (detect leaks)
conntrack -L | grep ESTABLISHED | wc -l

# Count TIME_WAIT entries
conntrack -L 2>/dev/null | grep TIME_WAIT | wc -l

# Check insert_failed (smoking gun for Kubernetes DNS 5-second timeout)
conntrack -S | grep insert_failed

# Watch new connections and deletions in real time
conntrack -E

# Check for specific service connections
conntrack -L | grep "10.96.100.50"

# Break down entries by TCP state
conntrack -L -o extended | awk '{print $4}' | sort | uniq -c | sort -rn

# Break down entries by protocol
conntrack -L -o extended | awk '{print $3}' | sort | uniq -c | sort -rn

# Show current table count
conntrack -C

# Delete a specific stale entry
conntrack -D -p tcp --src 10.0.0.5 --dst 10.0.1.100 --sport 12345 --dport 8080
```

---

## `nstat` / `netstat` — Network statistics

**When to use:** `nstat` reads kernel SNMP counters with delta support (compare two readings). `netstat -s` shows cumulative protocol stats since boot. Both are invaluable for detecting drops, retransmits, and buffer pressure without packet captures.

### `nstat` — Kernel network counters (preferred)

#### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-a` | Show all counters including zero-value ones |
| `-z` | Show zero-value counters |
| `-d` | Show delta since last reset (default behavior) |
| (no flag) | Show only changed counters since last read |

#### Common Usage

```bash
# Quick overview at start of incident triage
nstat -az 2>/dev/null | head -20

# Check all drop counters
nstat -az | grep -i drop
```

#### Important Variants

```bash
# TCP retransmit counters (packet loss evidence)
nstat -az | grep -i retrans

# Listen queue overflow and SYN retransmits
nstat -az | grep -E 'TcpExt(Listen|Backlog|Prune|RcvPruned|OFO|TCPSchedulerFailed)'
nstat -az | grep -E "TcpExtListenOverflow|TcpExtTCPSynRetrans"

# Socket buffer pruning (socket too small for traffic)
nstat -az | grep -E '(PruneCalled|RcvPruned|OfoPruned)'
nstat -a | grep TcpExtRcvPruned

# NfConntrack full (new connections being dropped)
nstat -az | grep NfConntrackFull

# UDP receive buffer overflow (DNS, syslog, metrics)
nstat -az | grep UdpInErrors

# SYN cookie activation rate (SYN flood indicator)
nstat -az | grep TcpExtSyncookiesSent

# TCP memory pressure
nstat -z | grep TcpExtTCPMemoryPressures

# PMTUD failure counters (kernel 5.1+)
nstat -a | grep -i "MTUPFail"

# TIME_WAIT recycling and PAWS
nstat -a | grep -E "TcpExt(TWRecycled|PAWSPassive|SynRetrans)"

# Accept queue drops
nstat -z | grep TcpExtListenDrops
nstat -z | grep ListenOverflows

# Watch retransmit counters in real time
watch -n 1 'nstat -az | grep -E "TcpRetransSegs|TcpExtTCPFastRetrans|TcpExtTCPSlowStartRetrans"'

# Watch all drop counters in real time
watch -n 1 'nstat -az | grep -i drop'
```

### `netstat` — Legacy protocol stats

```bash
# Protocol statistics since boot (TCP, UDP, IP)
netstat -s | head -30

# Check for receive errors and drops
netstat -s | grep -E "(receive errors|failed|overflow|dropped)"

# TCP retransmit counters
netstat -s | grep -i "retransmit"
```

---

## `sysctl` — Kernel network parameters

**When to use:** Read and tune kernel network parameters at runtime. Use `-w` to set a value, persist via `/etc/sysctl.d/*.conf` and `sysctl --system`.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `KEY` | Read a single parameter |
| `-w KEY=VALUE` | Set a parameter (takes effect immediately) |
| `-a` | Show all kernel parameters |
| `-p FILE` | Load parameters from file |
| `--system` | Load all `/etc/sysctl.d/` and `/etc/sysctl.conf` files |

### Important Network Parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `net.ipv4.ip_forward` | `0` | Must be `1` for routing, NAT, containers |
| `net.core.netdev_max_backlog` | `1000` | Per-CPU queue for packets from NIC to kernel; increase to `10000`+ at high pps |
| `net.core.netdev_budget` | `300` | Max packets per softirq poll cycle |
| `net.core.netdev_budget_usecs` | `2000` | Max time (µs) per softirq cycle |
| `net.core.somaxconn` | `128` | System-wide listen backlog ceiling; set to `65536` for busy servers |
| `net.ipv4.tcp_max_syn_backlog` | `1024` | SYN queue size (half-open connections); set to `65536` under flood |
| `net.ipv4.tcp_syncookies` | `1` | SYN cookies for SYN flood defense |
| `net.ipv4.tcp_syn_retries` | `6` | SYN retransmit count; reduce to `2` under flood |
| `net.ipv4.tcp_synack_retries` | `5` | SYN-ACK retransmit count; reduce to `2` under flood |
| `net.ipv4.tcp_rmem` | `4096 87380 6291456` | TCP recv buffer: min/default/max (bytes) |
| `net.ipv4.tcp_wmem` | `4096 16384 4194304` | TCP send buffer: min/default/max (bytes) |
| `net.core.rmem_max` | `212992` | Ceiling for SO_RCVBUF; set to `67108864` for 10G+ |
| `net.core.wmem_max` | `212992` | Ceiling for SO_SNDBUF |
| `net.ipv4.tcp_mem` | `auto` | Global TCP memory thresholds (pages) |
| `net.ipv4.tcp_window_scaling` | `1` | TCP window scaling (must be `1` for BDP > 64KB) |
| `net.ipv4.tcp_tw_reuse` | `0` | Reuse TIME_WAIT sockets for new outbound connections |
| `net.ipv4.tcp_fin_timeout` | `60` | FIN_WAIT_2 timeout; reduce to `15` to reclaim sockets faster |
| `net.ipv4.ip_local_port_range` | `32768 60999` | Ephemeral port range; expand to `1024 65535` |
| `net.ipv4.tcp_keepalive_time` | `7200` | Idle time before first keepalive probe (seconds) |
| `net.ipv4.tcp_keepalive_intvl` | `75` | Interval between keepalive probes |
| `net.ipv4.tcp_keepalive_probes` | `9` | Probes before RST |
| `net.ipv4.tcp_congestion_control` | `cubic` | TCP congestion algorithm; `bbr` for high-BDP links |
| `net.core.default_qdisc` | `pfifo_fast` | Default qdisc; use `fq` when enabling BBR |
| `net.ipv4.fib_multipath_hash_policy` | `0` | ECMP hash: `0`=L3, `1`=L3+L4, `2`=L3+L4+inner |
| `net.netfilter.nf_conntrack_max` | `65536` | Max conntrack entries; increase to `262144`–`1048576` on busy NAT |
| `net.netfilter.nf_conntrack_count` | (read-only) | Current conntrack entries |
| `net.netfilter.nf_conntrack_tcp_timeout_time_wait` | `120` | Conntrack TIME_WAIT timeout; reduce to `30` |
| `net.netfilter.nf_conntrack_udp_timeout` | `30` | UDP conntrack timeout |
| `net.ipv4.tcp_mtu_probing` | `0` | Enable PLPMTUD (`1`=enable when ICMP blocked) |
| `net.ipv4.ip_no_pmtu_disc` | `0` | Disable PMTUD (`0`=enabled) |
| `net.core.busy_poll` | `0` | Busy poll period in µs for SO_BUSY_POLL sockets |

### Common Usage

```bash
# Check if forwarding is enabled
sysctl net.ipv4.ip_forward

# Enable IP forwarding immediately
sysctl -w net.ipv4.ip_forward=1

# Check conntrack capacity
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count
```

### Important Variants

```bash
# SYN flood defense
sysctl -w net.ipv4.tcp_syncookies=1
sysctl -w net.ipv4.tcp_syn_retries=2
sysctl -w net.ipv4.tcp_synack_retries=2
sysctl -w net.ipv4.tcp_max_syn_backlog=65536
sysctl -w net.core.somaxconn=65536

# High-throughput socket buffers
sysctl -w net.ipv4.tcp_rmem="4096 131072 67108864"
sysctl -w net.ipv4.tcp_wmem="4096 16384 67108864"
sysctl -w net.core.rmem_max=67108864
sysctl -w net.core.wmem_max=67108864
sysctl -w net.ipv4.tcp_mem="4000000 6000000 8000000"

# Conntrack tuning for NAT-heavy workloads
sysctl -w net.netfilter.nf_conntrack_max=524288
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
sysctl -w net.netfilter.nf_conntrack_udp_timeout=10

# Port exhaustion remediation
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
sysctl -w net.ipv4.tcp_tw_reuse=1

# Keepalive tuning (detect dead connections faster)
sysctl -w net.ipv4.tcp_keepalive_time=60
sysctl -w net.ipv4.tcp_keepalive_intvl=10
sysctl -w net.ipv4.tcp_keepalive_probes=6

# Packet drop reduction
sysctl -w net.core.netdev_max_backlog=10000
sysctl -w net.core.netdev_budget=600
sysctl -w net.core.netdev_budget_usecs=8000
sysctl -w net.core.rmem_default=26214400

# BBR TCP congestion control
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr

# ECMP L3+L4 hash
sysctl -w net.ipv4.fib_multipath_hash_policy=1

# Enable PLPMTUD (when ICMP is blocked)
sysctl -w net.ipv4.tcp_mtu_probing=1
sysctl -w net.ipv4.tcp_base_mss=1024

# Persist settings
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/99-network.conf
sysctl --system
```

---

## `ping` / `ping6` — ICMP connectivity test

**When to use:** Verify L3 reachability, measure round-trip latency, detect packet loss, and probe path MTU with DF bit.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-c N` | Send N packets then stop |
| `-i SEC` | Interval between packets (default 1s; use 0.2 for faster probing) |
| `-s BYTES` | Payload size in bytes |
| `-M do` | Set DF bit — essential for MTU black hole detection |
| `-M dont` | Unset DF bit (allow fragmentation) |
| `-t TTL` | Set IP TTL |
| `-W SEC` | Timeout for response |
| `-I DEV\|IP` | Bind to specific interface or source IP |
| `-q` | Quiet mode (summary only) |
| `-f` | Flood ping (requires root; use carefully) |
| `-4` / `-6` | Force IPv4 or IPv6 |

### Common Usage

```bash
# Basic reachability: 3 packets
ping -c 3 10.0.1.1
```

**Output:**
```
PING 10.0.1.1 (10.0.1.1) 56(84) bytes of data.
64 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=0.412 ms
64 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=0.389 ms
64 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=0.401 ms

--- 10.0.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2043ms
rtt min/avg/max/mdev = 0.389/0.400/0.412/0.009 ms
```

### Important Variants

```bash
# Extended test (30 packets) to detect intermittent loss
ping -c 30 10.0.1.100

# MTU black hole detection: DF bit set, 1472-byte payload = 1500-byte packet
ping -M do -s 1472 10.0.1.100

# Test with smaller payload after initial failure (binary search path MTU)
ping -M do -s 1372 10.0.1.100
ping -M do -s 1452 10.0.1.100

# VXLAN/tunnel path: test inner payload MTU (1450-byte payload = 1478-byte inner)
ping -M do -s 1400 10.0.1.100

# IPv6 ping
ping6 -c 3 fe80::1%eth0

# Bind to specific source interface
ping -I eth1 -c 3 8.8.8.8

# Slow ping to detect periodic loss (load-correlated)
ping -i 0.2 -c 100 10.0.1.100
```

---

## `mtr` — Combined traceroute + ping

**When to use:** Continuous per-hop latency and loss measurement in a single tool. More informative than traceroute for diagnosing where loss or high latency is introduced along a path.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-n` | No DNS resolution (faster output) |
| `-r` / `--report` | Run in report mode (non-interactive, prints summary) |
| `--report-cycles N` | Number of pings per hop in report mode (default 10) |
| `-T` / `--tcp` | Use TCP SYN probes (bypasses ICMP filtering) |
| `-u` / `--udp` | Use UDP probes |
| `-P PORT` | Port for TCP/UDP probes |
| `-s BYTES` | Probe packet size |
| `-i SEC` | Interval between probes |
| `-m N` | Maximum number of hops |
| `-b` | Show both hostnames and IPs |

### Common Usage

```bash
# Continuous path analysis with 100 cycles per hop
mtr -n --report --report-cycles 100 10.0.1.100
```

**Output:**
```
Host                          Loss%   Snt   Last   Avg  Best  Wrst StDev
1. 10.0.0.1                   0.0%   100    0.3   0.3   0.2   0.6   0.1
2. 172.16.0.1                 0.0%   100    1.2   1.3   1.0   2.1   0.2
3. 203.0.113.1                0.0%   100   12.4  12.6  12.0  14.5   0.4
4. 10.0.1.100                 2.0%   100   12.8  13.0  12.0  45.3   3.2
```

### Important Variants

```bash
# TCP-based probe on port 443 (penetrates TCP-only firewalls)
mtr -n -T -P 443 --report --report-cycles 50 10.0.1.100

# Live interactive view
mtr -n 10.0.1.100

# IPv6 path analysis
mtr -n -6 --report --report-cycles 50 2001:db8::1
```

---

## `traceroute` / `tracepath` — Path tracing

**When to use:** `traceroute` maps the L3 path to a destination using TTL-expiry; use TCP mode to penetrate firewalls that block UDP/ICMP. `tracepath` performs path tracing with automatic PMTUD discovery.

### `traceroute`

#### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-n` | No DNS resolution |
| `-T` | Use TCP SYN probes |
| `-p PORT` | Port for TCP/UDP probes |
| `-I` | Use ICMP ECHO probes (like Windows `tracert`) |
| `-U` | Use UDP probes (default on Linux) |
| `-m N` | Max hops (TTL) |
| `-w SEC` | Timeout per probe |
| `--mtu` | Report MTU at each hop (TCP mode) |

#### Common Usage

```bash
# Standard UDP traceroute without DNS
traceroute -n 8.8.8.8
```

**Output:**
```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max
 1  10.0.0.1  0.412 ms  0.389 ms  0.401 ms
 2  172.16.0.1  1.234 ms  1.201 ms  1.198 ms
 3  * * *
 4  8.8.8.8  12.456 ms  12.412 ms  12.389 ms
```

#### Important Variants

```bash
# TCP traceroute: use this in cloud environments (ICMP often blocked)
traceroute -T -p 443 10.0.1.100 -n

# TCP traceroute with MTU discovery (detect MTU black holes)
traceroute -T -p 8080 10.0.1.100 -n --mtu

# ICMP-based traceroute (equivalent to Windows tracert)
traceroute -n -I 8.8.8.8

# IPv6 traceroute
traceroute6 -n 2001:db8::1
```

### `tracepath`

**When to use:** Simpler than `traceroute`, does not need root, and automatically performs MTU discovery along the path.

```bash
# Trace path and discover per-hop MTU
tracepath -n 10.0.1.100
```

**Output:**
```
 1?: [LOCALHOST]     pmtu 1500
 1:  10.0.0.1        0.412ms
 2:  172.16.0.1      1.234ms
 3:  10.0.1.100      1.801ms reached
     Resume: pmtu 1500 hops 3 back 3
```

---

## `arp` / `arping` — ARP table management and probing

### `arp` — ARP table (legacy)

**When to use:** Quick ARP table inspection on older systems. Prefer `ip neigh` on modern Linux.

```bash
# Show ARP table
arp -n

# Delete a stale entry
arp -d 10.0.0.50

# Add static entry
arp -s 10.0.0.100 02:00:00:00:00:01
```

### `arping` — ARP-level reachability probe

**When to use:** Test L2 reachability when ICMP is blocked. Detect duplicate IPs. Send gratuitous ARP after failover/VIP migration.

#### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-I DEV` | Interface to use |
| `-c N` | Send N ARP packets |
| `-A` | Gratuitous ARP reply (ARP reply with src=dst) |
| `-U` | Gratuitous ARP request (broader compatibility) |
| `-D` | Duplicate address detection mode (exit 1 if duplicate found) |
| `-s SRC` | Set source IP address |
| `-w SEC` | Timeout |

#### Common Usage

```bash
# L2 reachability test
arping -I eth0 -c 3 10.0.1.50
```

**Output:**
```
ARPING 10.0.1.50 from 10.0.1.10 eth0
Unicast reply from 10.0.1.50 [02:42:ac:11:00:32]  0.743ms
Unicast reply from 10.0.1.50 [02:42:ac:11:00:32]  0.681ms
Unicast reply from 10.0.1.50 [02:42:ac:11:00:32]  0.694ms
Sent 3 probes (1 broadcast(s))
Received 3 response(s)
```

#### Important Variants

```bash
# Duplicate IP detection (exits non-zero if duplicate found)
arping -D -I eth0 -c 2 10.0.0.100

# Gratuitous ARP reply — update neighbor caches after VIP failover
arping -A -I eth0 -c 3 10.0.1.100

# Gratuitous ARP request — broader compatibility (updates most implementations)
arping -U -I eth0 -c 3 10.0.1.100

# Send gratuitous ARP to announce IP ownership after failover
arping -I eth0 -A -c 3 10.0.0.100

# Verify receipt of gratuitous ARP (in parallel on another host)
tcpdump -i eth0 arp and 'arp[6:2] = 2'
```

---

## `nc` (netcat) — Network Swiss army knife

**When to use:** Quickly test TCP/UDP connectivity, create ad-hoc listeners, transfer files, proxy connections, and banner-grab services. `nc` (ncat/netcat) is available in most distributions.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-z` | Zero-I/O mode — scan/test without sending data |
| `-v` | Verbose (show connection result) |
| `-w SEC` | Timeout in seconds |
| `-u` | UDP mode |
| `-l` | Listen mode |
| `-p PORT` | Source port (listen) or local port |
| `-k` | Keep listening after connection closes |
| `-e CMD` | Execute command on connection |
| `-n` | No DNS resolution |

### Common Usage

```bash
# Test TCP port connectivity (triage first step)
nc -zv -w5 10.0.1.100 8080
```

**Output (success):**
```
Connection to 10.0.1.100 8080 port [tcp/http-alt] succeeded!
```

**Output (refused):**
```
nc: connect to 10.0.1.100 port 8080 (tcp) failed: Connection refused
```

**Output (filtered/timeout):**
```
(hangs until -w timeout expires, then exits)
```

### Important Variants

```bash
# Test UDP port (DNS nameserver reachability)
nc -uzv -w3 10.0.0.53 53

# PCI segmentation test: verify CDE isolation across common ports
for port in 5432 3306 6379 8443 22 3389; do
  result=$(nc -zv -w3 10.2.0.100 $port 2>&1)
  echo "Port $port: $result"
done

# Ad-hoc TCP listener (test from client side)
nc -l -p 9999

# Transfer a file over TCP
# Receiver:
nc -l -p 9999 > output.bin
# Sender:
nc 10.0.1.100 9999 < input.bin

# Banner grab / HTTP probe
echo -e "HEAD / HTTP/1.0\r\n\r\n" | nc -w3 example.com 80

# Test UDP DNS resolver connectivity
nc -u -z 10.0.0.53 53

# Force TCP for DNS query test (RFC 7766 TCP fallback)
nc -w3 10.0.0.53 53
```

---

## `socat` — Advanced socket relay

**When to use:** Bidirectional data relay between arbitrary socket types (TCP, UDP, UNIX, TLS, serial, files). Ideal for forwarding ports, debugging protocol interactions, and creating test harnesses not possible with `nc`.

### Key Options

| Syntax | Description |
|--------|-------------|
| `socat A B` | Relay data between address A and address B |
| `TCP:HOST:PORT` | Connect TCP to HOST:PORT |
| `TCP-LISTEN:PORT` | Listen on TCP PORT |
| `TCP-LISTEN:PORT,reuseaddr,fork` | Accept multiple connections |
| `UDP:HOST:PORT` | UDP send/receive |
| `UNIX-CONNECT:PATH` | Connect to UNIX socket |
| `UNIX-LISTEN:PATH` | Create UNIX socket listener |
| `OPENSSL:HOST:PORT` | TLS client connection |
| `EXEC:CMD` | Pipe to/from a subprocess |
| `STDOUT` | Standard output |
| `-v` | Verbose data logging |
| `-x` | Hex dump of transferred data |

### Common Usage

```bash
# Forward local port to remote host (useful from inside containers)
socat TCP-LISTEN:8080,reuseaddr,fork TCP:backend-host:80
```

### Important Variants

```bash
# TCP port forward with multiple client support
socat TCP-LISTEN:2222,reuseaddr,fork TCP:internal-host:22

# Forward TCP to a UNIX domain socket
socat TCP-LISTEN:8080,reuseaddr,fork UNIX-CONNECT:/var/run/app.sock

# Connect to a UNIX socket interactively
socat STDIO UNIX-CONNECT:/var/run/docker.sock

# Send a raw HTTP request to a UNIX socket
echo -e "GET /info HTTP/1.0\r\n\r\n" | socat - UNIX-CONNECT:/var/run/docker.sock

# TLS probe: verify certificate and connection
socat - OPENSSL:example.com:443,verify=0

# UDP relay
socat UDP-LISTEN:5514,reuseaddr UDP:10.0.0.50:514

# Create bidirectional pipe between two TCP hosts
socat TCP:host1:8080 TCP:host2:9090

# Debug protocol: show hex dump of transferred bytes
socat -x TCP-LISTEN:8080,reuseaddr,fork TCP:backend:80

# Replay a file to a service (HTTP test)
socat TCP:example.com:80 FILE:request.http
```

---

## `nmap` — Network scanner

**When to use:** Port scanning, service fingerprinting, OS detection, firewall behavior assessment, and TLS cipher enumeration. Use in authorized environments only.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-p PORT` | Scan specific port(s) |
| `-p-` | Scan all 65535 ports |
| `-sS` | TCP SYN scan (stealth, requires root) |
| `-sT` | TCP connect scan (no root needed) |
| `-sU` | UDP scan |
| `-sn` | Ping scan (host discovery, no port scan) |
| `-Pn` | Skip host discovery (treat all hosts as up) |
| `-n` | No DNS resolution |
| `-O` | OS detection |
| `-sV` | Service version detection |
| `-A` | Aggressive: OS + version + traceroute + scripts |
| `-T4` | Timing template 4 (fast) |
| `--script SCRIPT` | Run NSE script |
| `-oN FILE` | Normal output to file |
| `-oG FILE` | Grepable output |

### Common Usage

```bash
# Quick port probe — distinguish open vs closed vs filtered
nmap -p 8080 10.0.1.100
```

**Output:**
```
PORT     STATE    SERVICE
8080/tcp open     http-proxy      # firewall allows, process listening
8080/tcp closed   http-proxy      # firewall allows, no process
8080/tcp filtered http-proxy      # firewall blocking (no RST received)
```

### Important Variants

```bash
# Test TCP port from client (bypass ICMP-block assumptions)
nmap -Pn -n -p 8080 10.0.1.100

# Scan multiple ports
nmap -Pn -n -p 22,80,443,8080,8443 10.0.1.100

# UDP port scan
nmap -sU -p 53,161,514 10.0.1.100

# PCI requirement 4: verify no TLS 1.0/1.1 on payment API
nmap --script ssl-enum-ciphers -p 443 payment-api.mycompany.com \
  | grep -E "TLSv1\.[01]"

# Check cipher suite strength and TLS version support
nmap --script ssl-cert,ssl-enum-ciphers -p 443 example.com

# Host discovery scan across a subnet
nmap -sn 10.0.0.0/24

# Full TCP SYN scan with service detection
nmap -sS -sV -T4 -p- 10.0.1.100

# OS and service fingerprint
nmap -A -p 22,80,443 10.0.1.100
```

---

## `iperf3` — Bandwidth testing

**When to use:** Measure TCP/UDP throughput between two hosts; validate network capacity, detect RSS imbalance, test bufferbloat, and baseline post-change performance.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-s` | Server mode |
| `-c HOST` | Client mode: connect to HOST |
| `-p PORT` | Port (default 5201) |
| `-t SEC` | Test duration in seconds (default 10) |
| `-P N` | Number of parallel streams |
| `-R` | Reverse mode (server-to-client) |
| `-u` | UDP mode |
| `-b RATE` | Target bandwidth for UDP test (e.g., `1G`) |
| `-l BYTES` | Buffer length / datagram size |
| `-w BYTES` | Socket buffer size |
| `-J` | JSON output (for scripting) |
| `--logfile FILE` | Log output to file |
| `-Z` | Use sendfile() instead of send() |
| `-4` / `-6` | Force IPv4 or IPv6 |

### Common Usage

```bash
# Start server
iperf3 -s -p 5201

# Single-stream TCP test for 30 seconds
iperf3 -c 10.0.1.100 -t 30
```

**Output (client):**
```
Connecting to host 10.0.1.100, port 5201
[  5] local 10.0.0.5 port 54321 connected to 10.0.1.100 port 5201
[ ID] Interval       Transfer     Bitrate         Retr  Cwnd
[  5] 0.00-30.00 sec  33.8 GBytes  9.68 Gbits/sec    0   1.77 MBytes
```

### Important Variants

```bash
# 8 parallel streams — test RSS distribution and aggregate throughput
iperf3 -c 10.0.1.100 -t 30 -P 8

# Reverse direction: server-to-client (test downstream path)
iperf3 -c 10.0.1.100 -t 30 -R

# UDP test targeting 1Gbps (measures jitter and loss)
iperf3 -c 10.0.1.100 -u -b 1G -t 30

# Background load generation for bufferbloat testing
iperf3 -c 10.0.1.100 -t 60 &
ping -c 60 10.0.1.100  # measure latency under load

# JSON output for scripting
iperf3 -c 10.0.1.100 -t 30 -J | jq '.end.sum_received.bits_per_second'

# Multiple runs with increasing parallel streams
for P in 1 2 4 8 16; do
  echo "=== $P streams ==="
  iperf3 -c 10.0.1.100 -t 10 -P $P | grep -E "SUM|receiver"
done
```

---

## `nsenter` — Enter process namespaces

**When to use:** Debug a running container or process by entering its namespaces directly from the host, using host tools (tcpdump, strace, ip) against the container's network, mount, or PID namespace.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-t PID` | Target process PID |
| `-n` | Enter network namespace |
| `-m` | Enter mount namespace |
| `-p` | Enter PID namespace |
| `-u` | Enter UTS (hostname) namespace |
| `-i` | Enter IPC namespace |
| `--all` | Enter all namespaces of the target process |

### Common Usage

```bash
# Get container PID (using crictl or docker)
CONTAINER_ID=$(crictl ps | grep mypod | awk '{print $1}')
PID=$(crictl inspect $CONTAINER_ID | jq '.info.pid')

# Run ip addr inside container's network namespace using host tools
nsenter -t $PID -n ip addr
```

**Output:**
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
    inet 127.0.0.1/8 scope host lo
3: eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450
    inet 10.244.1.23/24 brd 10.244.1.255 scope global eth0
```

### Important Variants

```bash
# Show routes inside container
nsenter -t $PID -n ip route

# Show listening sockets inside container
nsenter -t $PID -n ss -tlnp

# Run tcpdump inside container's namespace (uses host's binary)
nsenter -t $PID -n tcpdump -i eth0 -w /tmp/capture.pcap

# Enter ALL namespaces (network + mount + PID + UTS)
nsenter -t $PID --all

# Enter network + UTS namespace (for hostname-aware debugging)
nsenter -t $PID -n -u

# Read pod resolv.conf (NodeLocal DNSCache verification)
nsenter -t $POD_PID -n cat /etc/resolv.conf

# Get container's veth index on host side (for tc/eBPF correlation)
nsenter -t $CTR_PID -n ip link show eth0 | grep -oP 'if\K[0-9]+'

# Alternative: create a named netns symlink and use ip netns exec
ln -s /proc/$PID/ns/net /run/netns/debug-container
ip netns exec debug-container ip addr
```

---

## `ip netns exec` — Execute in named network namespace

**When to use:** Run commands inside a named network namespace (created with `ip netns add` or symlinked from `/run/netns/`). Used for container debugging when you have a named namespace reference.

### Common Usage

```bash
# Execute a command in a named namespace
ip netns exec myns ip addr
ip netns exec myns ip route
ip netns exec myns ss -tlnp

# Check pod CNI namespace
ip netns exec cni-abc12345 ip addr
ip netns exec cni-abc12345 ss -tlnp

# Full namespace setup sequence
ip netns add web
ip link add veth-host type veth peer name veth-web
ip link set veth-web netns web
ip addr add 10.100.0.1/24 dev veth-host
ip netns exec web ip addr add 10.100.0.2/24 dev veth-web
ip netns exec web ip link set veth-web up
ip netns exec web ip link set lo up
ip netns exec web ip route add default via 10.100.0.1
ip link set veth-host up

# Test connectivity from namespace to host
ip netns exec web ping -c 3 10.100.0.1

# Test internet from inside namespace (requires host NAT)
ip netns exec web curl -s --max-time 5 http://1.1.1.1

# Run VRF-scoped command (alternative to ip netns exec)
ip vrf exec vrf-mgmt ping 10.0.0.254
ip vrf exec vrf-mgmt ssh admin@10.0.0.10
```

---

---

# Part 2 — Packet Capture, DNS, HTTP, TLS & Firewall

## `tcpdump` — Packet capture and real-time traffic analysis

**When to use:** Capture live traffic or write pcap files for offline analysis. Essential for diagnosing packet drops, retransmissions, SYN floods, malformed packets, and verifying that traffic actually reaches or leaves an interface.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-i <iface>` | Interface to capture on (`any` for all interfaces) |
| `-n` | Do not resolve hostnames |
| `-nn` | Do not resolve hostnames or port names |
| `-v` / `-vv` / `-vvv` | Increase verbosity (IP TTL, checksum, protocol details) |
| `-e` | Print Ethernet (link-layer) headers |
| `-w <file>` | Write raw packets to a pcap file |
| `-r <file>` | Read packets from a pcap file instead of live interface |
| `-s <snaplen>` | Capture only first N bytes per packet (0 = full packet) |
| `-c <count>` | Stop after capturing N packets |
| `-A` | Print packet payload as ASCII |
| `-X` | Print payload in hex and ASCII |
| `-q` | Quiet mode: less protocol information |
| `-tttt` | Print absolute human-readable timestamps |
| `'<filter>'` | BPF filter expression (see below) |

### Common Usage

```bash
# Capture all traffic on eth0, no name resolution
tcpdump -i eth0 -nn

# Capture traffic to/from a host on a specific port, write to file
tcpdump -i eth0 -nn -w /tmp/capture.pcap host 10.0.1.100 and port 5432

# Read a previously saved pcap file
tcpdump -r /tmp/capture.pcap -nn
```

**Output:**
```
14:22:03.112451 IP 10.0.1.50.54231 > 10.0.1.100.5432: Flags [S], seq 1483920211, win 64240, options [mss 1460,sackOK,TS val 3789 ecr 0,nop,wscale 7], length 0
14:22:03.112612 IP 10.0.1.100.5432 > 10.0.1.50.54231: Flags [S.], seq 3012744890, ack 1483920212, win 65160, options [mss 1460,sackOK,TS val 4421 ecr 3789,nop,wscale 7], length 0
14:22:03.112690 IP 10.0.1.50.54231 > 10.0.1.100.5432: Flags [.], ack 1, win 502, length 0
```

### BPF Filter Syntax

```bash
# Filter by host
tcpdump -i eth0 -nn host 192.168.1.1

# Filter by source or destination
tcpdump -i eth0 -nn src 10.0.0.5
tcpdump -i eth0 -nn dst 10.0.0.5

# Filter by port
tcpdump -i eth0 -nn port 443
tcpdump -i eth0 -nn portrange 8000-8080

# Filter by protocol
tcpdump -i eth0 -nn icmp
tcpdump -i eth0 -nn udp port 53

# Combine filters with and / or / not
tcpdump -i eth0 -nn 'host 10.0.1.100 and port 5432'
tcpdump -i eth0 -nn 'not port 22 and not arp'

# TCP flag filters (BPF byte offset syntax)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack|tcp-fin) != 0'
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack = 0'

# ICMP Fragmentation Needed (type 3, code 4) for PMTUD debugging
tcpdump -ni eth0 'icmp and icmp[icmptype] == 3 and icmp[icmpcode] == 4'

# IPv6 NDP Neighbor Solicitation (135) and Advertisement (136)
tcpdump -i eth0 -n 'icmp6 and (ip6[40] == 135 or ip6[40] == 136)'

# IPv6 Router Advertisement (type 134)
tcpdump -i eth0 -n 'icmp6 and ip6[40] == 134' -v

# ARP packets with link-layer headers
tcpdump -e -i eth0 arp -n

# Capture ARP replies only (arp-reply type = 2)
tcpdump -i eth0 'arp and arp[6:2] = 2'

# VXLAN encapsulated traffic
tcpdump -i eth0 -w /tmp/vxlan.pcap udp port 4789

# Flannel VXLAN (some clusters use 8472)
tcpdump -i eth0 -nn udp port 8472
```

### Important Variants

```bash
# Capture SYN packets only, write to pcap (SYN flood evidence)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack = 0' \
  -w /tmp/syn_flood.pcap -c 10000

# Capture with 96-byte snap length (headers only, smaller files)
tcpdump -i eth0 -w /tmp/headers.pcap -s 96 host 10.0.1.100 and port 5432

# Capture inside a container network namespace via nsenter
nsenter -t $CONTAINER_PID -n tcpdump -i eth0 -w /tmp/container.pcap

# DNS query capture (spot ndots search domain amplification)
tcpdump -i any -nn port 53

# Capture on all interfaces, find retransmissions (duplicate seqs)
tcpdump -i eth0 -nn -w /tmp/cap.pcap host 10.0.2.50 and port 443
tcpdump -r /tmp/cap.pcap -nn | grep -E "\[P\.\]|seq" | head -50

# Capture with full timestamps
tcpdump -i eth0 -tttt -nn port 443
```

---

## `tshark` — CLI Wireshark for pcap analysis and field extraction

**When to use:** Parse pcap files with Wireshark dissectors from the command line. Extract specific protocol fields, apply display filters, or convert pcap to JSON/CSV for downstream analysis.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-r <file>` | Read from pcap file |
| `-i <iface>` | Live capture interface |
| `-Y '<filter>'` | Display filter (Wireshark syntax, applied after capture) |
| `-f '<filter>'` | Capture filter (BPF syntax, applied during capture) |
| `-T fields` | Output mode: print specific fields |
| `-T json` | Output as JSON |
| `-T pdml` | Output as XML |
| `-e <field>` | Field to extract (used with `-T fields`) |
| `-E header=y` | Print field header row |
| `-E separator=,` | Set field separator (default tab) |
| `-R '<filter>'` | Read filter (legacy; prefer `-Y`) |
| `-n` | Disable name resolution |
| `-q` | Quiet — suppress packet summary lines |
| `-z <stat>` | Statistics: `io,stat,1`, `conv,tcp`, `endpoints,ip`, etc. |
| `-2` | Two-pass analysis (enables some advanced filters) |

### Common Usage

```bash
# Read pcap and apply a display filter
tshark -r /tmp/capture.pcap -Y 'tcp.flags.syn == 1 and tcp.flags.ack == 0'

# Extract specific fields from SYN packets
tshark -r /tmp/syn.pcap -Y 'tcp.flags.syn == 1' \
  -T fields -e ip.src -e ip.dst -e tcp.srcport -e tcp.dstport \
  -e tcp.options.wscale -E header=y
```

**Output:**
```
ip.src          ip.dst          tcp.srcport  tcp.dstport  tcp.options.wscale
10.0.1.50       10.0.1.100      54231        5432         7
10.0.1.51       10.0.1.100      54232        5432         7
```

### Important Variants

```bash
# Live capture with display filter, output JSON
tshark -i eth0 -Y 'http.response.code >= 500' -T json

# Extract TLS SNI from ClientHello packets
tshark -r /tmp/tls.pcap -Y 'tls.handshake.type == 1' \
  -T fields -e ip.src -e tls.handshake.extensions_server_name

# HTTP request/response summary
tshark -r /tmp/http.pcap -Y 'http' \
  -T fields -e http.request.method -e http.request.uri -e http.response.code

# TCP conversation statistics
tshark -r /tmp/capture.pcap -q -z conv,tcp

# Endpoint statistics
tshark -r /tmp/capture.pcap -q -z endpoints,ip

# IO statistics in 1-second buckets
tshark -r /tmp/capture.pcap -q -z io,stat,1

# Extract DNS queries and responses
tshark -r /tmp/dns.pcap -Y 'dns' \
  -T fields -e dns.qry.name -e dns.resp.addr -e dns.flags.response
```

---

## `dig` — DNS lookup and DNSSEC debugging tool

**When to use:** The primary tool for DNS interrogation. Query any record type, trace the full delegation path, test DNSSEC validation, compare resolvers, and measure lookup times.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `@<server>` | Use this resolver instead of system default |
| `<name>` | DNS name to query |
| `<type>` | Record type: `A`, `AAAA`, `MX`, `NS`, `TXT`, `SOA`, `PTR`, `SRV`, `CAA`, `DNSKEY`, `DS`, `RRSIG` |
| `+short` | One-line answer only |
| `+noall +answer` | Print only the answer section |
| `+trace` | Trace full delegation from root servers |
| `+dnssec` | Request DNSSEC records (sets DO bit) |
| `+cd` | Checking disabled — bypass DNSSEC validation |
| `+tcp` | Force query over TCP |
| `+stats` | Show query time and server stats |
| `+time=<n>` | Set query timeout in seconds |
| `+retry=<n>` | Number of retries |
| `-x <ip>` | Reverse (PTR) lookup |
| `-4` / `-6` | Force IPv4 / IPv6 transport |
| `+multiline` | Pretty-print SOA and DNSKEY records |

### Common Usage

```bash
# Basic A record query
dig example.com A

# Query using a specific resolver
dig @8.8.8.8 example.com A +stats
```

**Output:**
```
; <<>> DiG 9.18.1 <<>> @8.8.8.8 example.com A +stats
;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            300     IN      A       93.184.216.34

;; Query time: 12 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Mar 27 14:00:00 UTC 2026
;; MSG SIZE  rcvd: 56
```

### Important Variants

```bash
# Trace full delegation chain from root
dig example.com A +trace

# Short answer only — great for scripting
dig @8.8.8.8 api.company.com A +short

# Compare public vs private resolver (split-horizon debugging)
dig @8.8.8.8 api.company.com A +short       # public view
dig @10.0.0.2 api.company.com A +short      # internal view

# Query AWS VPC resolver directly
dig @169.254.169.253 api.example.com

# Query CoreDNS directly (Kubernetes)
dig @10.96.0.10 service-name.namespace.svc.cluster.local

# Kubernetes service DNS FQDN
dig +short service-name.namespace.svc.cluster.local

# Reverse lookup
dig -x 93.184.216.34

# SOA record — shows serial, refresh, and negative cache TTL (minimum)
dig example.com SOA +multiline

# Authoritative nameservers
dig NS example.com +short

# Query authoritative NS directly (bypass recursive cache)
dig @ns1.example.com service.example.com +short

# DNSSEC validation check — look for 'ad' (authenticated data) flag
dig +dnssec example.com A

# DNSSEC with checking disabled — diagnose DNSSEC failures
dig +cd example.com A

# Trace DNSSEC chain — find broken or expired RRSIG
dig +dnssec +trace service.example.com

# Check DNSKEY records
dig +dnssec example.com DNSKEY

# Check DS (delegation signer) record in parent zone
dig DS example.com

# Check CAA — restricts which CAs can issue certs
dig example.com CAA

# Force TCP transport (test TCP fallback, large responses)
dig +tcp api.example.com @10.0.0.53

# Measure DNS latency
time dig +short service.namespace.svc.cluster.local

# Check negative TTL in SOA MINIMUM field
dig service.example.com | grep -A5 "AUTHORITY SECTION"

# DNS-over-TLS test via TCP (port 853)
dig @1.1.1.1 -p 853 example.com +tcp
```

---

## `nslookup` — Interactive and non-interactive DNS queries

**When to use:** Quick DNS lookups when `dig` is unavailable (common on Windows and older Linux). Useful for interactive DNS debugging sessions and verifying search domain expansion inside containers.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `<name>` | Name to resolve (positional argument) |
| `<server>` | Optional second argument: resolver to use |
| `-type=<type>` | Record type: `A`, `AAAA`, `MX`, `NS`, `TXT`, `SOA`, `PTR` |
| `-debug` | Show full DNS message including query/response details |
| `-timeout=<n>` | Seconds before retrying |
| `-port=<n>` | Query on non-standard port |

### Common Usage

```bash
# Simple forward lookup
nslookup example.com

# Query using a specific server
nslookup example.com 8.8.8.8
```

**Output:**
```
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name:   example.com
Address: 93.184.216.34
```

### Important Variants

```bash
# Check a specific record type
nslookup -type=MX example.com
nslookup -type=NS example.com
nslookup -type=TXT example.com

# Reverse lookup
nslookup 93.184.216.34

# Debug mode — shows full query/response packets
nslookup -debug example.com

# Verify RDS endpoint resolves after failover
nslookup mydb.cluster-abcdefg.us-east-1.rds.amazonaws.com

# Test from inside a pod (check search domain expansion)
nslookup service-name           # should expand to service-name.namespace.svc.cluster.local
nslookup service-name.namespace.svc.cluster.local  # FQDN, no search expansion
```

---

## `host` — Simple, human-readable DNS lookups

**When to use:** Fast one-liner DNS queries in scripts or when you want clean output without `dig`'s verbosity. Supports all record types and reverse lookups.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-t <type>` | Record type: `A`, `AAAA`, `MX`, `NS`, `TXT`, `SOA`, `PTR`, `ANY` |
| `-v` | Verbose — show full query and response |
| `-a` | Equivalent to `-v -t ANY` |
| `-4` / `-6` | Force IPv4 / IPv6 transport |
| `-W <secs>` | Timeout in seconds |
| `-r` | Disable recursive queries (non-recursive lookup) |

### Common Usage

```bash
# Forward lookup
host example.com
```

**Output:**
```
example.com has address 93.184.216.34
example.com has IPv6 address 2606:2800:220:1:248:1893:25c8:1946
example.com mail is handled by 0 .
```

### Important Variants

```bash
# Reverse lookup
host 93.184.216.34

# MX records
host -t MX example.com

# NS records
host -t NS example.com

# TXT records (SPF, DKIM, DMARC)
host -t TXT example.com
host -t TXT _dmarc.example.com

# SOA record
host -t SOA example.com

# Query a specific resolver
host example.com 8.8.8.8

# Verbose output showing full request/response
host -v example.com
```

---

## `resolvectl` — Query and manage systemd-resolved

**When to use:** On systemd-based Linux hosts, `resolvectl` is the interface to `systemd-resolved`. Use it to check per-interface DNS settings, test resolution, flush the cache, and inspect DNSSEC/LLMNR/mDNS configuration.

### Key Options

| Subcommand | Description |
|------------|-------------|
| `status` | Show DNS servers, search domains, protocols per interface |
| `query <name>` | Perform a DNS lookup through systemd-resolved |
| `query -t <type> <name>` | Query a specific record type |
| `flush-caches` | Flush the DNS cache |
| `statistics` | Show resolver cache hit/miss statistics |
| `reset-statistics` | Reset resolver statistics counters |
| `dns [iface] [servers]` | Show or set DNS servers for an interface |
| `domain [iface] [domains]` | Show or set search domains for an interface |
| `log-level <level>` | Set logging verbosity (`debug`, `info`, `warning`) |
| `monitor` | Live stream of all DNS queries and responses |

### Common Usage

```bash
# Show full resolver status: DNS servers, protocols, search domains
resolvectl status
```

**Output:**
```
Global
       LLMNR setting: no
MulticastDNS setting: no
  DNSOverTLS setting: no
      DNSSEC setting: allow-downgrade
    DNSSEC supported: yes
          DNS Servers: 10.0.0.53
           DNS Domain: ~.

Link 2 (eth0)
      Current Scopes: DNS
       LLMNR setting: yes
          DNS Servers: 10.0.0.53
           DNS Domain: us-east-1.compute.internal
```

### Important Variants

```bash
# Perform a DNS query through systemd-resolved
resolvectl query example.com
resolvectl query -t MX example.com
resolvectl query -t AAAA example.com

# Flush DNS cache after record change or propagation wait
resolvectl flush-caches
# (legacy alias: systemd-resolve --flush-caches)

# Show resolver cache statistics
resolvectl statistics

# Monitor all DNS queries in real time (debugging ndots, search domain amplification)
resolvectl monitor

# Check DNSSEC status for a domain
resolvectl query --legend=yes example.com

# Set a specific DNS server for eth0 (temporary, survives until NetworkManager resets)
resolvectl dns eth0 1.1.1.1 8.8.8.8

# Set search domain for eth0
resolvectl domain eth0 example.internal corp.local
```

---

## `curl` — HTTP/HTTPS client for testing, debugging, and automation

**When to use:** The go-to tool for HTTP/HTTPS connectivity testing, TLS debugging, header inspection, API calls, authentication flows, and measuring per-phase request latency. Covers HTTP/1.1, HTTP/2, and HTTP/3.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-v` | Verbose: show TLS handshake, request headers, response headers |
| `-s` | Silent: suppress progress meter and error messages |
| `-S` | With `-s`, still show errors |
| `-o <file>` | Write response body to file (`/dev/null` to discard) |
| `-O` | Write to file named after remote file |
| `-I` | HEAD request — show response headers only |
| `-i` | Include response headers in stdout output |
| `-L` | Follow HTTP redirects |
| `-X <method>` | HTTP method: `GET`, `POST`, `PUT`, `DELETE`, `PATCH` |
| `-d '<data>'` | POST body (string) |
| `--data-binary @file` | POST body from file (preserves newlines) |
| `-H '<header>'` | Add or override a request header |
| `-u user:pass` | HTTP Basic authentication |
| `-b '<cookies>'` | Send cookies |
| `-c <file>` | Save response cookies to file |
| `--cookie-jar <file>` | Alias for `-c` |
| `-k` | Skip TLS certificate verification (insecure) |
| `--cacert <file>` | Use custom CA bundle for TLS verification |
| `--cert <file>` | Client certificate for mTLS |
| `--key <file>` | Client private key for mTLS |
| `--tls-max <ver>` | Maximum TLS version: `1.0`, `1.1`, `1.2`, `1.3` |
| `--tlsv1.2` | Require at least TLS 1.2 |
| `--tlsv1.3` | Require TLS 1.3 only |
| `--http2` | Force HTTP/2 (ALPN negotiation) |
| `--http3` | Force HTTP/3 over QUIC |
| `--resolve <host:port:ip>` | Override DNS for a specific host:port |
| `--connect-timeout <n>` | TCP connection timeout in seconds |
| `--max-time <n>` | Total request timeout in seconds |
| `-w '<format>'` | Write-out: print timing/status variables after response |
| `-A '<agent>'` | Set User-Agent header |
| `--compressed` | Request gzip/deflate compression |
| `-4` / `-6` | Force IPv4 / IPv6 |
| `--noproxy '*'` | Bypass all proxies |
| `--proxy <url>` | Use an HTTP/SOCKS proxy |

### Common Usage

```bash
# Verbose HTTPS request — shows TLS handshake, all headers
curl -v https://example.com

# Check HTTP status code only (silent, discard body)
curl -s -o /dev/null -w "%{http_code}\n" https://example.com
```

**Output:**
```
* Trying 93.184.216.34:443...
* Connected to example.com (93.184.216.34) port 443
* ALPN: offering h2,http/1.1
* TLS 1.3, TLS handshake, Client hello (1):
* TLS 1.3, TLS handshake, Server hello (2):
* TLS 1.3, TLS handshake, Encrypted Extensions (8):
* TLS 1.3, TLS handshake, Certificate (11):
* TLS 1.3, TLS handshake, CERT Verify (15):
* TLS 1.3, TLS handshake, Finished (20):
> GET / HTTP/2
> Host: example.com
...
< HTTP/2 200
```

### Important Variants

```bash
# HEAD request — check status and headers without downloading body
curl -I https://example.com

# Check public IP of current host
curl -s ifconfig.me

# Per-phase latency breakdown (DNS / TCP / TLS / TTFB / total)
curl -s -o /dev/null \
  -w "dns_lookup:     %{time_namelookup}s\ntcp_connect:    %{time_connect}s\ntls_handshake:  %{time_appconnect}s\nttfb:           %{time_starttransfer}s\ntotal:          %{time_total}s\n" \
  https://service:443/health

# Compact one-liner timing (run 10x to get distribution)
curl -w "dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" \
  -o /dev/null -s https://service:443/health

# Timed connectivity test with connection and total timeouts
curl -v --connect-timeout 5 --max-time 10 http://service:8080/health

# Force HTTP/2 and show ALPN negotiation
curl --http2 -v https://example.com

# Test with custom Host header (K8s ingress routing)
curl -H "Host: hostname.example.com" http://10.0.1.5/path

# Override DNS — test a specific IP without changing /etc/hosts
curl -v https://api.example.com/health --resolve api.example.com:443:54.1.2.3

# POST with JSON body
curl -X POST https://api.example.com/v1/resource \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"key": "value"}'

# POST with file body
curl -X POST https://api.example.com/upload \
  -H "Content-Type: application/octet-stream" \
  --data-binary @/path/to/file

# mTLS client certificate authentication
curl --cert client.crt --key client.key --cacert ca.crt https://service:443/

# Follow redirects (up to 30 by default)
curl -L https://example.com

# Check HTTP security headers on API endpoint
curl -I https://api.example.com/v1/health | \
  grep -E "Strict-Transport|X-Content-Type|X-Frame|Content-Security|Access-Control"

# Extract TLS error details from verbose output
curl -v https://service:443/health 2>&1 | grep -E "SSL|error|certificate|expired|verify"

# Bypass TLS verification (testing self-signed certs; never in production)
curl -sk https://internal-service:8443/health

# Test with custom CA bundle
curl --cacert /etc/ssl/custom/ca-bundle.crt https://internal-service/

# DNS-over-HTTPS (DoH) query via Cloudflare JSON API
curl -H "accept: application/dns-json" \
  "https://cloudflare-dns.com/dns-query?name=example.com&type=A"

# Check certificate transparency logs for a domain
curl -s "https://crt.sh/?q=api.example.com&output=json" | \
  jq '.[] | {cn: .common_name, issuer: .issuer_name, expiry: .not_after}'

# Test internet connectivity through NAT Gateway
curl -s --connect-timeout 5 http://checkip.amazonaws.com

# Test EC2 IMDS (should be blocked from workload subnets)
curl -s --connect-timeout 3 http://169.254.169.254/latest/meta-data/

# Envoy admin API — dump full config
curl -s localhost:15000/config_dump | jq .

# Envoy cluster live stats
curl -s localhost:15000/clusters | grep "backend::" | \
  grep -E "(healthy|cx_active|rq_active|rq_timeout)"
```

---

## `wget` — Non-interactive file downloader and HTTP client

**When to use:** Download files from HTTP/FTP servers in scripts. Supports recursive downloads, resuming interrupted transfers, mirroring, and authenticated downloads. Useful when `curl` is unavailable or when you need built-in retry logic.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-O <file>` | Save output to a named file (`-` for stdout) |
| `-o <logfile>` | Log messages to a file instead of stderr |
| `-q` | Quiet: suppress output except errors |
| `-v` | Verbose output |
| `-c` | Continue/resume a partially downloaded file |
| `--tries=<n>` | Number of retries (0 = unlimited) |
| `--timeout=<n>` | Timeout for connect, DNS, and read in seconds |
| `--connect-timeout=<n>` | Timeout for connection establishment only |
| `--read-timeout=<n>` | Timeout for data read stalls |
| `-r` | Recursive download |
| `-l <n>` | Maximum recursion depth |
| `-np` | Do not ascend to parent directory during recursion |
| `-p` | Download all page dependencies (images, CSS, JS) |
| `--no-check-certificate` | Skip TLS verification |
| `--ca-certificate=<file>` | Custom CA bundle |
| `--certificate=<file>` | Client certificate for mTLS |
| `--private-key=<file>` | Client private key for mTLS |
| `--header='<H>'` | Add a request header |
| `--post-data='<data>'` | Send POST request with data |
| `--post-file=<file>` | Send POST request with file body |
| `--user=<u>` | HTTP Basic auth username |
| `--password=<p>` | HTTP Basic auth password |
| `--spider` | Do not download — just check URL exists (HTTP 200 vs error) |
| `-S` | Print server response headers |
| `--limit-rate=<rate>` | Throttle download speed (e.g., `1m` = 1 MB/s) |
| `-b` | Run in background |
| `-i <file>` | Read URLs from a file |
| `--mirror` | Enable mirroring (`-r -N -l inf --no-remove-listing`) |

### Common Usage

```bash
# Download a file to current directory
wget https://example.com/file.tar.gz

# Download to stdout (pipe to another command)
wget -qO- https://example.com/script.sh | bash
```

**Output:**
```
--2026-03-27 14:00:00--  https://example.com/file.tar.gz
Resolving example.com... 93.184.216.34
Connecting to example.com|93.184.216.34|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5242880 (5.0M) [application/gzip]
Saving to: 'file.tar.gz'
file.tar.gz     100%[===================>]   5.00M  2.35MB/s    in 2.1s
```

### Important Variants

```bash
# Resume a partial download
wget -c https://example.com/large-file.iso

# Check if a URL is reachable without downloading (spider mode)
wget --spider https://service:8080/health

# Show response headers
wget -S -q -O /dev/null https://example.com

# Download with 3 retries and 10 second timeout
wget --tries=3 --timeout=10 https://example.com/file.tar.gz

# Download with Authorization header
wget --header="Authorization: Bearer $TOKEN" https://api.example.com/export

# POST request
wget --post-data='{"key":"value"}' --header="Content-Type: application/json" \
  -O - https://api.example.com/v1/resource

# Download and extract in one pipeline
wget -qO- https://releases.example.com/app-v1.2.tar.gz | tar xz

# Recursive download with depth limit (mirror a doc site)
wget -r -l 2 -np -p https://docs.example.com/

# Throttle download to avoid saturating links
wget --limit-rate=5m https://example.com/bigfile.bin

# Skip TLS verification (insecure — testing only)
wget --no-check-certificate https://internal-dev.example.com/
```

---

## `openssl` — TLS debugging, certificate inspection, and PKI toolkit

**When to use:** The essential Swiss Army knife for anything TLS-related. Use `s_client` to test TLS connections, debug handshake failures, inspect certificates, and verify mTLS. Use `x509`, `req`, `verify`, and `genrsa` for PKI operations and certificate lifecycle management.

### Key Options — `s_client`

| Flag/Option | Description |
|-------------|-------------|
| `-connect <host:port>` | Target server |
| `-servername <name>` | SNI hostname (required if IP ≠ hostname) |
| `-tls1_2` / `-tls1_3` | Force a specific TLS version |
| `-tls1_1` | Force TLS 1.1 (use to confirm it is rejected) |
| `-showcerts` | Show full certificate chain, not just leaf |
| `-cert <file>` | Client certificate for mTLS |
| `-key <file>` | Client private key for mTLS |
| `-CAfile <file>` | CA bundle for server certificate verification |
| `-verify <depth>` | Enable certificate verification with chain depth |
| `-alpn <proto>` | Offer ALPN protocol (e.g., `h2`, `http/1.1`) |
| `-starttls <proto>` | STARTTLS upgrade: `smtp`, `pop3`, `imap`, `ftp` |
| `-msg` | Show all protocol messages |
| `-debug` | Full hex dump of protocol messages |
| `-state` | Print SSL state machine transitions |
| `-status` | Request OCSP stapling response |

### Common Usage

```bash
# Connect to HTTPS server and inspect TLS handshake
openssl s_client -connect example.com:443 -servername example.com </dev/null
```

**Output:**
```
CONNECTED(00000003)
depth=2 C=US, O=DigiCert Inc, CN=DigiCert Global Root CA
verify return:1
depth=1 C=US, O=DigiCert Inc, CN=DigiCert TLS RSA SHA256 2020 CA1
verify return:1
depth=0 CN=www.example.com
verify return:1
---
Certificate chain
 0 s:CN=www.example.com
   i:C=US, O=DigiCert Inc, CN=DigiCert TLS RSA SHA256 2020 CA1
...
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: ...
---
```

### Important Variants

```bash
# Test TLS 1.3 specifically
openssl s_client -connect example.com:443 -tls1_3 -servername example.com </dev/null

# Test with SNI on an ingress IP
openssl s_client -connect 10.0.1.5:443 -servername hostname.example.com </dev/null

# Get full cert chain (check for missing intermediates)
openssl s_client -connect host:443 -servername host -showcerts </dev/null 2>/dev/null

# Extract cert from live server and inspect it
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null \
  | openssl x509 -noout -text

# Check expiry dates from live server
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null \
  | openssl x509 -noout -dates

# Script-friendly expiry check (exits 0 = valid, 1 = expired)
echo | openssl s_client -connect host:443 -servername host 2>/dev/null \
  | openssl x509 -noout -checkend 0 && echo "cert valid" || echo "cert EXPIRED"

# Check if cert expires within 24 hours
openssl x509 -noout -checkend 86400 -in cert.pem && echo "OK" || echo "expires within 24h"

# List SANs from live server (hostname mismatch debugging)
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName

# mTLS — present client cert
openssl s_client -connect host:443 -cert client.crt -key client.key \
  -servername host </dev/null

# PCI/TLS version compliance: confirm TLS 1.1 is rejected
openssl s_client -connect payment-api.example.com:443 -tls1_1 2>&1 | \
  grep -E "handshake failure|alert|no protocols"

# DNS-over-TLS (DoT) — port 853
openssl s_client -connect 1.1.1.1:853 -servername 1.1.1.1

# SMTP with STARTTLS
openssl s_client -connect smtp.example.com:587 -starttls smtp

# x509: inspect a certificate file
openssl x509 -noout -text -in cert.pem

# x509: quick summary — subject, issuer, dates, SANs
openssl x509 -noout -subject -issuer -dates -ext subjectAltName -in cert.pem

# x509: verify expiry of a local cert file
openssl x509 -noout -dates -in cert.pem

# x509: get fingerprint (SHA-256)
openssl x509 -noout -fingerprint -sha256 -in cert.pem

# verify: check cert chain against system CA bundle
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt cert-chain.pem

# verify: check cert chain with explicit CA and intermediate
openssl verify -CAfile root-ca.pem -untrusted intermediate.pem leaf.pem

# verify: verify client cert is signed by server's trusted CA
openssl verify -CAfile server-trusted-ca.pem client.crt

# genrsa: generate a 4096-bit RSA private key
openssl genrsa -out server.key 4096

# req: generate a CSR with SANs inline
openssl req -new -key server.key -out server.csr \
  -subj "/CN=api.example.com/O=Example Inc/C=US" \
  -addext "subjectAltName=DNS:api.example.com,DNS:api-v2.example.com"

# req: generate a self-signed cert for testing
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 \
  -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1" -nodes

# speed: benchmark TLS performance
openssl speed rsa2048 ecdh

# Check if private key matches certificate
openssl x509 -noout -modulus -in cert.pem | md5sum
openssl rsa  -noout -modulus -in key.pem  | md5sum
# (outputs must match)
```

---

## `iptables` — IPv4 packet filtering, NAT, and packet mangling

**When to use:** Inspect, modify, and debug the Linux kernel's IPv4 netfilter rules. Use it to allow/deny traffic, implement NAT, mark packets for policy routing, clamp TCP MSS, trace packet paths, and review kube-proxy rules in Kubernetes clusters.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-L [chain]` | List rules in chain (default: all chains in filter table) |
| `-n` | No hostname/service resolution (faster listing) |
| `-v` | Verbose: show interface, packet/byte counters |
| `--line-numbers` | Show rule line numbers |
| `-t <table>` | Target table: `filter` (default), `nat`, `mangle`, `raw` |
| `-A <chain>` | Append rule to chain |
| `-I <chain> [n]` | Insert rule at position N (default: 1 = top) |
| `-D <chain> [n]` | Delete rule by line number or by specification |
| `-R <chain> <n>` | Replace rule at line number N |
| `-F [chain]` | Flush (delete all rules in) chain |
| `-Z [chain]` | Zero packet/byte counters |
| `-P <chain> <target>` | Set chain default policy: `ACCEPT`, `DROP` |
| `-N <chain>` | Create a new user-defined chain |
| `-p <proto>` | Protocol: `tcp`, `udp`, `icmp`, `all` |
| `-s <cidr>` | Source address/network |
| `-d <cidr>` | Destination address/network |
| `--dport <port>` | Destination port (requires `-p tcp` or `-p udp`) |
| `--sport <port>` | Source port |
| `-i <iface>` | Input interface (PREROUTING, INPUT, FORWARD) |
| `-o <iface>` | Output interface (FORWARD, OUTPUT, POSTROUTING) |
| `-j <target>` | Jump to target: `ACCEPT`, `DROP`, `REJECT`, `LOG`, `DNAT`, `SNAT`, `MASQUERADE`, `MARK`, `RETURN`, `TRACE`, `NOTRACK` |
| `-m <module>` | Load match extension: `conntrack`, `state`, `set`, `limit`, `multiport`, `comment` |

### Common Usage

```bash
# List INPUT chain rules with packet counts and line numbers
iptables -L INPUT -n -v --line-numbers
```

**Output:**
```
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1     1823  145K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:443
2        0     0 DROP       tcp  --  *      *       10.10.10.0/24        0.0.0.0/0
```

### Important Variants

```bash
# --- Listing rules ---

# All filter table chains with counters
iptables -L -n -v

# Specific table listing
iptables -t nat -L -n -v
iptables -t mangle -L -n -v
iptables -t raw -L -n -v

# Show only active DROP/REJECT rules (non-zero counters)
iptables -L -n -v | grep -E "(DROP|REJECT)"
iptables -L -n -v | grep -E "DROP|REJECT" | awk '$1 != "0" || $2 != "0"'

# Dump all rules in restorable format
iptables-save

# Count total rules (high counts cause O(n) lookup latency)
iptables-save | wc -l

# --- Accepting and blocking ---

# Allow established and related connections (stateful)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Accept traffic on specific port
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Drop traffic from a CIDR
iptables -I INPUT -s 198.51.100.0/24 -j DROP

# Reject with ICMP port-unreachable (client gets immediate error)
iptables -A INPUT -p tcp --dport 8080 -j REJECT --reject-with icmp-port-unreachable

# Insert ACCEPT rule before an existing DROP (use line numbers)
iptables -I INPUT 2 -p tcp --dport 8443 -s 10.0.0.0/8 -j ACCEPT

# Delete a rule by line number
iptables -D INPUT 3

# Zero counters, then monitor which rules fire
iptables -Z INPUT
iptables -L INPUT -n -v --line-numbers | awk '$1 > 0 {print}'

# --- NAT ---

# SNAT: translate private source to a specific public IP
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j SNAT --to-source 1.2.3.4

# SNAT across a pool of IPs (expands port capacity)
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 10.0.1.1-10.0.1.10

# MASQUERADE: dynamic SNAT using interface IP (for DHCP/dynamic public IPs)
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# DNAT: redirect inbound traffic to internal host:port
iptables -t nat -A PREROUTING -d 1.2.3.4 -p tcp --dport 80 \
  -j DNAT --to-destination 10.0.0.10:8080

# List Docker NAT rules
iptables -t nat -L DOCKER -n -v

# List all POSTROUTING NAT rules
iptables -t nat -L POSTROUTING -n -v

# --- Packet marking for policy routing ---

# Mark packets destined for a network
iptables -t mangle -A OUTPUT -d 10.8.0.0/8 -j MARK --set-mark 0x1

# Mark monitoring traffic by destination port
iptables -t mangle -A OUTPUT -p tcp --dport 9090 -j MARK --set-mark 0x10

# --- MSS clamping (VXLAN/VPN overlay networks) ---

# Clamp MSS to PMTU (dynamic, recommended)
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# Set fixed MSS for VXLAN overlay (1450 MTU - 54B headers = 1396B MSS)
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN SYN \
  -j TCPMSS --set-mss 1396

# --- ICMP / PMTUD ---

# Allow ICMP Fragmentation Needed (required for PMTUD to work)
iptables -I INPUT  -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -I FORWARD -p icmp --icmp-type fragmentation-needed -j ACCEPT

# Remove a rule blocking ICMP (fix MTU black holes)
iptables -D INPUT -p icmp -j DROP

# --- Debugging and tracing ---

# Add a LOG rule at top of INPUT chain
iptables -I INPUT 1 -j LOG --log-prefix "IPT-DEBUG: " --log-level 4 --log-uid

# View logged packets from the kernel journal
# journalctl -k | grep "IPT-DEBUG"

# Enable TRACE for a specific flow (follow through all chains)
iptables -t raw -I PREROUTING -s 10.0.0.5 -p tcp --dport 80 -j TRACE
iptables -t raw -I PREROUTING -p tcp -d 10.96.100.50 --dport 443 -j TRACE

# Exempt high-frequency traffic from conntrack (health checks)
iptables -t raw -A PREROUTING -p tcp --dport 8080 -j NOTRACK
iptables -t raw -A OUTPUT     -p tcp --sport 8080 -j NOTRACK

# --- SYN flood mitigation ---

# Rate-limit SYN packets (100/s, burst 200)
iptables -I INPUT -p tcp --syn -m limit --limit 100/s --limit-burst 200 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# --- Kubernetes kube-proxy rules ---

# List KUBE-SERVICES NAT chain
iptables -t nat -L KUBE-SERVICES -n

# Find rule for a specific ClusterIP
iptables -t nat -L KUBE-SERVICES -n | grep 10.96.50.100

# Follow kube-proxy chain to endpoints
iptables -t nat -L KUBE-SVC-XXXXXXXXXXXXXXXX -n -v
iptables -t nat -L KUBE-SEP-XXXXXXXXXXXXXXXX -n -v

# --- Migration to nftables ---

# Translate a single iptables rule to nftables syntax
iptables-translate -A INPUT -p tcp --dport 80 -j ACCEPT

# Migrate full ruleset to nftables
iptables-save | iptables-restore-translate -f - | nft -f -
```

---

## `ip6tables` — IPv6 packet filtering (iptables API for IPv6)

**When to use:** Manage IPv6 firewall rules. Syntax is identical to `iptables` but operates on IPv6 (inet6) traffic. On modern systems, prefer `nft` which handles both IPv4 and IPv6 in a single dual-stack table.

### Key Options

Same flags as `iptables`. Key IPv6-specific differences:

| Difference | Notes |
|------------|-------|
| Protocol `-p ipv6-icmp` | ICMPv6 protocol name (not `icmp`) |
| `--icmpv6-type` | ICMPv6 type names (vs `--icmp-type`) |
| No MASQUERADE + dynamic address | Use SNAT with explicit IPv6 address |
| No fragmentation | IPv6 fragments handled by end-hosts only |

### Common Usage

```bash
# List IPv6 INPUT rules
ip6tables -L INPUT -n -v --line-numbers

# Allow established/related IPv6 traffic
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Important Variants

```bash
# Allow ICMPv6 Packet Too Big (mandatory for IPv6 PMTUD)
ip6tables -A INPUT  -p ipv6-icmp --icmpv6-type packet-too-big -j ACCEPT
ip6tables -A FORWARD -p ipv6-icmp --icmpv6-type packet-too-big -j ACCEPT

# Allow NDP (Neighbor Solicitation and Advertisement — equivalent to ARP in IPv4)
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type neighbor-solicitation -j ACCEPT
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type neighbor-advertisement -j ACCEPT

# Allow Router Advertisement (required for SLAAC)
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type router-advertisement -j ACCEPT

# Block all IPv6 traffic from a prefix
ip6tables -A INPUT -s 2001:db8:bad::/48 -j DROP

# Allow SSH over IPv6
ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT

# Flush all IPv6 rules in filter table
ip6tables -F

# Save and restore IPv6 rules
ip6tables-save > /etc/iptables/rules.v6
ip6tables-restore < /etc/iptables/rules.v6

# Translate to nftables
ip6tables-translate -A INPUT -p tcp --dport 443 -j ACCEPT
```

---

## `nft` / `nftables` — Modern dual-stack netfilter framework

**When to use:** The successor to `iptables`/`ip6tables`/`arptables`/`ebtables`. Handles IPv4 and IPv6 in a single table. Supports sets, maps, and verdict maps natively for O(1) lookups. Atomic ruleset loading eliminates rule-update races. Required on modern Linux distributions where `iptables` is a compatibility shim over nft.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **table** | Top-level container; family: `ip`, `ip6`, `inet` (dual-stack), `arp`, `bridge` |
| **chain** | Contains rules; types: `filter`, `nat`, `route`; hooks: `prerouting`, `input`, `forward`, `output`, `postrouting` |
| **rule** | Match + verdict: `accept`, `drop`, `reject`, `return`, `jump <chain>` |
| **set** | Named collection of IPs, ports, or other values; used in match expressions |
| **map** | Key-to-value lookup (e.g., IP to verdict) |

### Common Usage

```bash
# List full ruleset
nft list ruleset

# Load ruleset atomically from file
nft -f /etc/nftables/ruleset.nft
```

**Output (`nft list ruleset`):**
```
table inet myfilter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        tcp dport { 22, 443, 8443 } accept
        drop
    }
}
```

### Important Variants

```bash
# --- Table and chain management ---

# Create a dual-stack filter table
nft add table inet myfilter

# Create an input chain with default-drop policy
nft add chain inet myfilter input \
  '{ type filter hook input priority 0; policy drop; }'

# Create a forward chain
nft add chain inet myfilter forward \
  '{ type filter hook forward priority 0; policy accept; }'

# Create a NAT table and postrouting chain
nft add table ip nat
nft add chain ip nat postrouting \
  '{ type nat hook postrouting priority srcnat; }'

# --- Rules ---

# Allow established and related
nft add rule inet myfilter input ct state established,related accept

# Accept ports using inline set (no O(n) rule scan)
nft add rule inet myfilter input tcp dport { 22, 443, 8443 } accept

# Drop traffic from a source IP
nft add rule inet myfilter input ip saddr 198.51.100.0/24 drop

# Reject with TCP reset
nft add rule inet myfilter input tcp dport 8080 reject with tcp reset

# Log dropped packets (with log prefix)
nft add rule inet myfilter input log prefix "nft-drop: " drop

# MASQUERADE for NAT gateway
nft add rule ip nat postrouting oifname "eth0" masquerade

# DNAT — redirect port 80 to internal host
nft add rule ip nat prerouting ip daddr 1.2.3.4 tcp dport 80 \
  dnat to 10.0.0.10:8080

# Notrack — exempt health checks from conntrack
nft add rule ip raw prerouting tcp dport 8080 notrack

# --- Sets ---

# Create a named set of IP addresses
nft add set inet myfilter blocked-ips { type ipv4_addr; }
nft add element inet myfilter blocked-ips { 198.51.100.1, 198.51.100.2 }

# Create a set with CIDR ranges
nft add set inet myfilter trusted-nets { type ipv4_addr; flags interval; }
nft add element inet myfilter trusted-nets { 10.0.0.0/8, 172.16.0.0/12 }

# Use a set in a rule
nft add rule inet myfilter input ip saddr @blocked-ips drop
nft add rule inet myfilter input ip saddr @trusted-nets tcp dport 8443 accept

# --- Inspection and debugging ---

# List only a specific table
nft list table inet myfilter

# Check rules matching a port
nft list ruleset | grep -B5 -A2 "8080"

# Find drop/reject rules during incident
nft list ruleset | grep -B5 "drop\|reject"

# Enable nftables tracing for a specific flow
# Step 1: add trace rule
nft add rule inet myfilter input ip saddr 10.0.0.5 tcp dport 80 meta nftrace set 1
# Step 2: monitor trace events
nft monitor trace

# --- Atomic ruleset management (safest approach) ---

# Verify syntax without applying
nft -c -f /etc/nftables/ruleset.nft

# Apply atomically (entire ruleset swapped as one transaction)
nft -f /etc/nftables/ruleset.nft

# Complete ruleset file example saved for reference:
# table inet filter {
#     chain input {
#         type filter hook input priority 0; policy drop;
#         ct state vmap { established : accept, related : accept, invalid : drop }
#         tcp dport { 22, 443 } accept
#         icmpv6 type { nd-neighbor-solicit, nd-neighbor-advert, mld-listener-query } accept
#         log prefix "INPUT-DROP: "
#     }
# }
```

---

## `ipset` — High-performance IP set management for iptables/nftables

**When to use:** When you have more than a handful of IP addresses, CIDRs, or port ranges to match in iptables rules. `ipset` uses hash tables or bitmaps for O(1) lookups instead of the O(n) linear scan of individual iptables rules. Essential for blocklists, allowlists, and rate-limiting at scale.

### Key Options

| Subcommand | Description |
|------------|-------------|
| `create <name> <type>` | Create a new set |
| `add <name> <entry>` | Add an entry to a set |
| `del <name> <entry>` | Remove an entry from a set |
| `test <name> <entry>` | Test if entry is in set (exit 0 = found) |
| `list [name]` | List set contents |
| `flush [name]` | Remove all entries from set (or all sets) |
| `destroy [name]` | Delete set and free memory |
| `save [name]` | Print set in restorable format |
| `restore` | Restore sets from `save` output |
| `swap <s1> <s2>` | Atomically swap two sets of the same type |
| `-exist` | Ignore errors if entry already exists |
| `--timeout <secs>` | Entry lifetime (0 = permanent) |

### Set Types

| Type | Use case |
|------|----------|
| `hash:ip` | Single IPv4/IPv6 addresses |
| `hash:net` | CIDR prefixes |
| `hash:ip,port` | IP + port combinations |
| `hash:net,port` | CIDR + port combinations |
| `hash:ip,port,ip` | IP + port + IP (NAT mapping) |
| `bitmap:port` | Fast port range bitmaps (0-65535) |
| `list:set` | Set of sets (meta-set) |

### Common Usage

```bash
# Create a set and add entries
ipset create blocked-ips hash:ip
ipset add blocked-ips 198.51.100.1
ipset add blocked-ips 203.0.113.50

# Use the set in iptables
iptables -I INPUT -m set --match-set blocked-ips src -j DROP
```

**Output (`ipset list blocked-ips`):**
```
Name: blocked-ips
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 296
References: 1
Number of entries: 2
Members:
198.51.100.1
203.0.113.50
```

### Important Variants

```bash
# --- Creating sets ---

# Hash:net set for CIDR matching (O(1) for thousands of subnets)
ipset create trusted-sources hash:net
ipset add trusted-sources 10.0.0.0/8
ipset add trusted-sources 172.16.0.0/12
ipset add trusted-sources 192.168.0.0/16

# Hash:ip,port for combined IP+port matching
ipset create allowed-services hash:ip,port
ipset add allowed-services 10.0.1.5,tcp:443
ipset add allowed-services 10.0.1.5,tcp:8443

# Timed entries (automatically expire after N seconds)
ipset create temp-block hash:ip timeout 3600
ipset add temp-block 203.0.113.99 timeout 600   # expire in 10 min

# --- Using sets in iptables rules ---

# O(1) allowlist lookup for trusted CIDR accepting port 8443
iptables -I INPUT -m set --match-set trusted-sources src \
  -p tcp --dport 8443 -j ACCEPT

# Block all sources in a blocklist
iptables -I INPUT -m set --match-set blocked-ips src -j DROP

# Match both source IP and destination port using hash:ip,port
iptables -I INPUT -m set --match-set allowed-services src,dst -j ACCEPT

# --- Atomic blocklist update (avoid traffic gaps) ---

# Load new blocklist into a temporary set
ipset create new-blocklist hash:net
# populate new-blocklist with updated entries...
ipset add new-blocklist 198.51.100.0/24

# Atomically swap old and new (zero-downtime update)
ipset swap blocked-ips new-blocklist
ipset destroy new-blocklist

# --- Persistence ---

# Save all sets to file
ipset save > /etc/ipset.conf

# Restore sets from file
ipset restore < /etc/ipset.conf

# --- Inspection ---

# List all sets
ipset list

# Test if an IP is in a set (exit 0 = member, 1 = not member)
ipset test trusted-sources 10.0.1.50
echo $?   # 0 if member

# Count entries in a set
ipset list blocked-ips | grep "^Number of entries"

# Flush a set without destroying it (keep iptables rule intact)
ipset flush blocked-ips

# --- nftables equivalent ---
# nft add set inet filter blocked-ips { type ipv4_addr; flags interval; }
# nft add element inet filter blocked-ips { 198.51.100.0/24 }
# nft add rule inet filter input ip saddr @blocked-ips drop
```

---

# Part 3 — eBPF, Kubernetes, Cloud & Container

## `bpftool` — BPF program and map inspection

**When to use:** Inspect, load, and manage eBPF programs and maps on a running system. Essential for debugging Cilium CNI internals, XDP packet filters, and CO-RE program development.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `prog list` | List all loaded BPF programs with IDs, tags, and load times |
| `prog show id <id>` | Show details for a specific program (verifier stats, map refs) |
| `prog dump xlated id <id>` | Dump BPF bytecode (translated instructions) |
| `prog dump jited id <id>` | Dump JIT-compiled x86-64 machine code |
| `prog load <obj> <pin-path>` | Load a BPF program and pin it to the BPF filesystem |
| `prog tracelog` | Monitor BPF ring buffer / trace_pipe output |
| `prog profile id <id>` | Measure per-program hardware performance counters |
| `map list` | List all BPF maps with type, key/value sizes, memory usage |
| `map show id <id>` | Show map metadata |
| `map dump id <id>` | Dump all key-value entries in a map |
| `map update id <id> key ... value ...` | Update a map entry (e.g., add to XDP blocklist) |
| `map delete id <id> key ...` | Remove an entry from a map |
| `map pin id <id> <path>` | Pin map to BPF filesystem (survives program termination) |
| `map create <path>` | Create a pinned BPF map |
| `net list dev <iface>` | List BPF programs attached to a network device |
| `btf show` | List all BTF (BPF Type Format) objects in the kernel |
| `btf dump file <path>` | Dump BTF type definitions from a file or kernel |
| `--pretty` | Format output as pretty-printed JSON |
| `-d` | Enable debug/verbose output |

### Common Usage

```bash
# List all loaded BPF programs
bpftool prog list

# Show details for program ID 42
bpftool prog show id 42 --pretty
```

**Output:**
```
42: xdp  name xdp_pass  tag 3b185187f1855c4c  gpl
        loaded_at 2025-03-27T10:12:44+0000  uid 0
        xlated 96B  jited 67B  memlock 4096B  map_ids 7
        btf_id 42
        pids cilium-agent(1234)
```

### Important Variants

```bash
# Dump JIT-compiled instructions for a program
bpftool prog dump jited id 42

# Dump BPF bytecode (verifier-translated)
bpftool prog dump xlated id 42

# Load an XDP program and view verifier output on failure
bpftool prog load my_xdp.o /sys/fs/bpf/my_xdp type xdp 2>&1

# Load fentry program attached to tcp_connect
bpftool prog load tcp_trace.bpf.o /sys/fs/bpf/tcp_trace type fentry \
  --attach-to tcp_connect

# List all BPF maps
bpftool map list

# Dump all entries from a running map
bpftool map dump id 42

# Dump first 50 entries (pipe through head)
bpftool map dump id 7 | head -50

# Add an IP to an XDP blocklist map (key = 192.168.99.3 in hex)
bpftool map update id $BLOCKLIST_ID key hex c0 a8 63 03 value hex 01 00 00 00

# Delete a false-positive entry from XDP blocklist map
bpftool map delete id 7 key hex <hex-bytes>

# Pin a map so it persists after program exit
bpftool map pin id 42 /sys/fs/bpf/my_map

# Create a pinned hash map on the BPF filesystem
bpftool map create /sys/fs/bpf/conn_count type hash key 4 value 8 \
  entries 65536 name conn_count

# List BPF programs attached to eth0 (XDP, TC-BPF)
bpftool net list dev eth0

# List Cilium-owned maps and programs
bpftool map list | grep cilium
bpftool prog list | grep cilium

# Generate vmlinux.h from kernel BTF for CO-RE development
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h

# Inspect kernel BTF type definitions
bpftool btf dump file /sys/kernel/btf/vmlinux format c | grep "struct sock" | head -5

# List all BTF objects loaded in the kernel
bpftool btf show

# Profile a program for 10 seconds (cycles and instructions)
bpftool prog profile id 42 duration 10 cycles instructions

# Verify CO-RE relocations were applied during load
bpftool prog load prog.bpf.o /sys/fs/bpf/prog -d 2>&1 | grep "CO-RE"

# Monitor BPF trace output (equivalent to reading trace_pipe)
bpftool prog tracelog
```

---

## `bpftrace` — Dynamic tracing with BPF one-liners

**When to use:** Write ad-hoc kernel and userspace tracing scripts without compiling C. Ideal for investigating TCP retransmits, DNS latency, packet drops, and connection behavior in production.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-e '<program>'` | Run inline bpftrace program |
| `-l '<probe-pattern>'` | List available probes matching pattern |
| `kprobe:<func>` | Attach to kernel function entry |
| `kretprobe:<func>` | Attach to kernel function return |
| `tracepoint:<cat>:<name>` | Attach to a static kernel tracepoint |
| `interval:s:<n>` | Fire every N seconds (for periodic reporting) |
| `printf(...)` | Print formatted output |
| `@map[key] = count()` | Aggregate count per key |
| `@map = hist(expr)` | Power-of-2 histogram |
| `@map[key] = sum(expr)` | Aggregate sum per key |
| `comm` | Current process name (built-in) |
| `pid` / `tid` | Current process/thread ID |
| `nsecs` | Current timestamp in nanoseconds |
| `ntop(AF_INET, addr)` | Convert integer to dotted-decimal IPv4 string |
| `ntohs(port)` | Convert network-byte-order port to host order |
| `kstack(n)` | Kernel stack trace (n frames) |

### Common Usage

```bash
# List all available network tracepoints
bpftrace -l 'tracepoint:net:*'

# List TCP kernel probes
bpftrace -l 'kprobe:tcp_*' | head -20
```

**Output:**
```
tracepoint:net:net_dev_queue
tracepoint:net:net_dev_xmit
tracepoint:net:net_dev_xmit_timeout
tracepoint:net:napi_poll
...
kprobe:tcp_connect
kprobe:tcp_sendmsg
kprobe:tcp_retransmit_skb
...
```

### Important Variants

```bash
# Trace TCP retransmissions with source/dest addresses
bpftrace -e '
kprobe:tcp_retransmit_skb {
  $sk = (struct sock *)arg0;
  printf("RETRANSMIT: %s:%d -> %s:%d\n",
    ntop(2, $sk->__sk_common.skc_rcv_saddr),
    $sk->__sk_common.skc_num,
    ntop(2, $sk->__sk_common.skc_daddr),
    ntohs($sk->__sk_common.skc_dport));
}'

# Count TCP retransmits per process (summarize at exit)
bpftrace -e '
kprobe:tcp_retransmit_skb {
  @retransmits[comm] = count();
}
END { print(@retransmits); clear(@retransmits); }'

# Measure TCP connect() latency histogram (handshake RTT)
bpftrace -e '
kprobe:tcp_v4_connect      { @start[tid] = nsecs; }
kretprobe:tcp_v4_connect
/@start[tid]/ {
  @conn_latency_us = hist((nsecs - @start[tid]) / 1000);
  delete(@start[tid]);
}'

# Count new TCP connections per second by destination port
bpftrace -e '
kprobe:tcp_connect {
  @connections[((struct sock *)arg0)->__sk_common.skc_dport] = count();
}
interval:s:1 { print(@connections); clear(@connections); }'

# Trace TCP state changes (count by state)
bpftrace -e '
kprobe:tcp_set_state {
  @states[arg1] = count();
}
interval:s:5 { print(@states); clear(@states); }'

# Track accept queue depth as histogram (detect listen overflow)
bpftrace -e '
kprobe:inet_csk_accept {
  $sk = (struct sock *)arg0;
  $backlog = $sk->sk_ack_backlog;
  if ($backlog > 0) @backlog_hist = hist($backlog);
}'

# Count packet drops by kernel drop reason (kernel 5.17+)
bpftrace -e '
tracepoint:skb:kfree_skb {
  @drops[args->reason] = count();
}
interval:s:5 { print(@drops); }'

# Count packets dropped by stack location (kernel stack trace)
bpftrace -e '
kprobe:kfree_skb {
  @drops[kstack(5)] = count();
}
END { print(@drops); }'

# Track per-process TCP send/receive byte totals
bpftrace -e '
kprobe:tcp_sendmsg   { @send[comm] = sum(arg2); }
kprobe:tcp_recvmsg   { @recv[comm] = sum(arg2); }
END { print(@send); print(@recv); }'

# Track outbound bytes sent per process via TCP
bpftrace -e 'kprobe:tcp_sendmsg { @bytes[comm] = sum(arg2); }'

# Count outbound packets per process (tracepoint variant)
bpftrace -e 'tracepoint:net:net_dev_xmit { @[comm] = count(); }'

# Trace all processes making connect() syscalls
bpftrace -e '
tracepoint:syscalls:sys_enter_connect {
  printf("%-10s %-6d connect() called\n", comm, pid);
}'

# Trace BPF syscall invocations (meta-tracing)
bpftrace -e 'kprobe:__sys_bpf { printf("bpf syscall cmd=%d\n", arg0); }'

# Histogram of TCP kernel receive queue latency
bpftrace -e '
kprobe:tcp_data_queue    { @start[tid] = nsecs; }
kretprobe:tcp_data_queue {
  @latency_us = hist((nsecs - @start[tid]) / 1000);
  delete(@start[tid]);
}'
```

---

## `perf` — Linux performance profiling

**When to use:** CPU profiling, hardware counter collection, and tracing kernel events including network subsystem activity. Use `perf stat` for quick counters, `perf record` + `perf report` for flame graphs, and `perf top` for live CPU profiling.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `stat` | Count hardware/software events for a command or PID |
| `record` | Sample events and write to perf.data |
| `report` | Display a perf.data recording |
| `top` | Live sampling view (like top for CPU functions) |
| `-g` | Enable call graph (stack trace) collection |
| `-e <event>` | Select event (e.g., `net:*`, `cycles`, `cache-misses`) |
| `-a` | System-wide collection (all CPUs) |
| `-p <pid>` | Attach to an existing process by PID |
| `-F <hz>` | Sampling frequency in Hz (default 1000) |
| `sleep <n>` | Run collection for N seconds then stop |
| `--call-graph dwarf` | Use DWARF for stack unwinding (more accurate) |

### Common Usage

```bash
# Profile nginx CPU usage live (identify hot functions)
perf top -p $(pgrep -d, nginx)
```

**Output:**
```
Samples: 12K of event 'cycles', 4000 Hz, Event count (approx.): 3200000000
Overhead  Shared Object        Symbol
  18.35%  [kernel]             [k] __inet_lookup_established
  12.40%  nginx                [.] ngx_http_process_request
   8.22%  [kernel]             [k] tcp_rcv_established
   5.18%  libssl.so.1.1        [.] AES_encrypt
```

### Important Variants

```bash
# Record all network tracepoint events system-wide for 5 seconds
perf record -g -e 'net:*' -a sleep 5

# Display the recording with call graphs
perf report

# Record with DWARF call graphs for accurate stack unwinding
perf record -g --call-graph dwarf -p $(pgrep myapp) sleep 10

# Quick event counter for a command (cycles, instructions, cache misses)
perf stat -e cycles,instructions,cache-misses,cache-references \
  curl -s https://api.example.com/health -o /dev/null

# System-wide stat for 10 seconds (network context switching)
perf stat -e context-switches,cpu-migrations,page-faults -a sleep 10

# Record specific kernel network function (e.g., TCP receive path)
perf record -g -e 'probe:tcp_rcv_established' -a sleep 5

# Live top for a specific PID (useful during load test)
perf top -p $APP_PID --call-graph fractal

# Generate flamegraph data (pipe to Brendan Gregg's flamegraph.pl)
perf record -F 99 -g -p $(pgrep app) sleep 30
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

---

## `strace` — System call tracer

**When to use:** Trace system calls made by a process to diagnose socket errors, ephemeral port exhaustion, application misconfigurations (e.g., calling `setsockopt` that disables auto-tuning), and to capture verifier logs from failed BPF program loads.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-p <pid>` | Attach to running process by PID |
| `-e trace=<syscall>` | Filter to specific syscalls (comma-separated) |
| `-e trace=network` | Trace all network-related syscalls |
| `-f` | Follow forked child processes |
| `-s <n>` | Maximum string length to print (default 32) |
| `-T` | Show time spent in each syscall |
| `-tt` | Show absolute timestamp for each syscall |
| `-c` | Summary count of syscalls and time at exit |
| `-o <file>` | Write output to file |
| `2>&1` | Redirect strace output (it goes to stderr by default) |

### Common Usage

```bash
# Trace socket read calls to diagnose slow application reading (high Recv-Q)
strace -p <PID> -e trace=read,recv,recvfrom,recvmsg 2>&1 | head -50
```

**Output:**
```
recvfrom(5, "HTTP/1.1 200 OK\r\nContent-Type: ap"..., 65536, 0, NULL, NULL) = 1460
recvfrom(5, 0x7f3b4c000000, 65536, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
```

### Important Variants

```bash
# Detect ephemeral port exhaustion (EADDRNOTAVAIL on connect)
strace -e connect your-application 2>&1 | grep EADDRNOTAVAIL

# Detect application calling setsockopt(SO_RCVBUF) disabling auto-tuning
strace -e trace=setsockopt -p $APP_PID 2>&1 | grep -i rcvbuf

# Trace all network syscalls on a running process
strace -p $PID -e trace=network -s 128 2>&1 | tail -50

# Show verifier log for a failed BPF program load
strace -e bpf bpftool prog load fail.bpf.o /dev/null 2>&1 | grep -A 50 "log"

# Count all syscalls and time (exit summary)
strace -c -p $PID

# Trace with timestamps and follow forks
strace -tt -f -e trace=connect,accept,sendto,recvfrom -p $PID 2>&1
```

---

## `lsof` — List open files and sockets

**When to use:** Identify which process owns a listening socket, find all network connections for a given process or user, and diagnose "port already in use" conflicts. Complements `ss` when process-to-socket correlation is needed.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-i` | Show all network connections and listening sockets |
| `-i :<port>` | Show processes using a specific port |
| `-i TCP` | Show only TCP sockets |
| `-i UDP` | Show only UDP sockets |
| `-i @<host>` | Show connections to a specific host |
| `-i TCP:80-443` | Show TCP connections in a port range |
| `-p <pid>` | Show all files opened by a process |
| `-u <user>` | Show files opened by a user |
| `-n` | Do not resolve hostnames (faster) |
| `-P` | Do not resolve port names (show numbers) |
| `+D <dir>` | Show all files open under a directory |
| `-s TCP:LISTEN` | Show only TCP sockets in LISTEN state |
| `-s TCP:ESTABLISHED` | Show only established TCP sockets |

### Common Usage

```bash
# Show all network connections (no hostname/port resolution)
lsof -i -n -P
```

**Output:**
```
COMMAND     PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx      1234   root    6u  IPv4  28701      0t0  TCP *:80 (LISTEN)
nginx      1234   root    7u  IPv4  28702      0t0  TCP *:443 (LISTEN)
sshd       5678   root    3u  IPv4  29100      0t0  TCP *:22 (LISTEN)
curl       9001  alice    5u  IPv4  30210      0t0  TCP 10.0.1.5:54321->93.184.216.34:443 (ESTABLISHED)
```

### Important Variants

```bash
# Find which process is using port 8080
lsof -i :8080 -n -P

# Show all TCP sockets in LISTEN state (no name resolution)
lsof -i TCP -s TCP:LISTEN -n -P

# Show all connections for a specific process
lsof -p $PID -i -n -P

# Show all network activity for a user
lsof -u www-data -i -n -P

# Count connections by remote host (find top talkers)
lsof -i TCP -s TCP:ESTABLISHED -n -P | awk '{print $9}' | \
  cut -d'>' -f2 | cut -d':' -f1 | sort | uniq -c | sort -rn | head -10

# Find who is connected to port 443 right now
lsof -i TCP:443 -s TCP:ESTABLISHED -n -P

# Show connections to a specific host
lsof -i @10.0.1.100 -n -P
```

---

## `kubectl` — Kubernetes CLI (networking operations)

**When to use:** Inspect and debug all Kubernetes networking resources: pods, services, endpoints, network policies, ingress, DNS, and CNI state. The first tool to reach for in any Kubernetes networking incident.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-n <namespace>` | Target namespace |
| `-A` / `--all-namespaces` | All namespaces |
| `-o wide` | Extra columns (pod IP, node) |
| `-o yaml` | Full YAML spec |
| `-o jsonpath='<expr>'` | Extract specific fields |
| `--selector` / `-l` | Label selector filter |
| `-f` / `--follow` | Stream logs |
| `--previous` | Logs from last (crashed) container |
| `--tail=<n>` | Last N lines of logs |
| `--rm` | Auto-delete ephemeral pod on exit |
| `--restart=Never` | Run pod once (not a Deployment) |

### Common Usage

```bash
# Check all pods with IP and node (cross-node connectivity)
kubectl get pods -A -o wide
```

**Output:**
```
NAMESPACE   NAME             READY   STATUS    IP           NODE
default     frontend-7d4b    1/1     Running   10.244.1.5   node-1
default     backend-5c9f     1/1     Running   10.244.2.8   node-2
kube-system coredns-5d4f     1/1     Running   10.244.0.3   node-1
```

### Service & Endpoint Inspection

```bash
# List services and their ClusterIPs
kubectl get svc -n <namespace>

# Get full service spec including selector
kubectl get svc <service-name> -n <namespace> -o yaml

# Get ClusterIP to bypass DNS in testing
kubectl get svc <service-name> -n <namespace> -o jsonpath='{.spec.clusterIP}'

# Get service selector (compare to pod labels)
kubectl get svc <service-name> -n <namespace> -o jsonpath='{.spec.selector}'

# Check service endpoints — empty = no matching ready pods
kubectl get endpoints <service-name> -n <namespace>

# Describe service for selector, ports, events
kubectl describe svc <service-name> -n <namespace>
```

### Network Policy

```bash
# List all NetworkPolicies in a namespace
kubectl get networkpolicy -n <namespace>

# Describe a policy (show podSelector, ingress/egress rules)
kubectl describe networkpolicy <policy-name> -n <namespace>

# Find default-deny policies (empty podSelector = applies to all pods)
kubectl get networkpolicy -A -o yaml | grep -A5 "podSelector: {}"

# Check Cilium-specific policies active in cluster
kubectl exec -n kube-system <cilium-pod> -- cilium policy get
```

### Pod Exec & Debugging

```bash
# Execute a command inside a running pod
kubectl exec -it <pod-name> -- bash

# Test pod-to-pod TCP connectivity on specific port
kubectl exec -it pod-a -- nc -zv <POD_B_IP> <PORT>

# Test HTTP between pods via pod IP (bypasses service)
kubectl exec -it pod-a -- curl http://<POD_B_IP>:<PORT>/health

# Test MTU at overlay limit (detect VXLAN black holes)
kubectl exec -it pod-a -- ping -M do -s 1400 <POD_B_IP>

# Check pod DNS config (ndots, search domains)
kubectl exec <pod-name> -- cat /etc/resolv.conf

# Query CoreDNS directly to isolate DNS from pod config
kubectl exec <pod-name> -- dig @10.96.0.10 service.namespace.svc.cluster.local

# Read pod resolv.conf via nsenter
nsenter -t $(kubectl get pod <pod-name> -o jsonpath='{.status.hostIP}') \
  -n cat /etc/resolv.conf
```

### Logs

```bash
# Stream logs from a pod
kubectl logs <pod-name> -f

# Logs from previous (crashed) container
kubectl logs <pod-name> --previous

# CoreDNS logs for DNS debugging
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# CoreDNS logs for slow queries or errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50 \
  | grep -i "slow\|error\|timeout"

# Kubelet CNI logs for networking issues
journalctl -u kubelet --since "30 minutes ago" | grep -i "cni\|network"

# VPC CNI (aws-node) IP allocation errors
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100 \
  | grep -i "ip\|eni\|prefix\|warm"
```

### Port-Forward & Proxy

```bash
# Forward local port to a pod
kubectl port-forward <pod-name> 8080:80

# Forward to a service
kubectl port-forward svc/<service-name> -n <namespace> 9090:9090

# Forward to Envoy admin interface
kubectl port-forward deploy/frontend 15000

# Forward to Kiali service graph UI
kubectl port-forward svc/kiali -n istio-system 20001:20001

# Start kubectl proxy (exposes API server locally on port 8001)
kubectl proxy --port=8001
```

### Resource Usage (Top)

```bash
# Check pod CPU/memory
kubectl top pod <pod-name>

# Check CoreDNS resource usage
kubectl top pod -n kube-system -l k8s-app=kube-dns

# Check nodes
kubectl top node
```

### CNI & Node Debugging

```bash
# Check if CNI has assigned pod CIDR to a node
kubectl describe node <node-name> | grep -A5 "PodCIDR"

# Check CNI pods are running on every node
kubectl get pod -n kube-system -l k8s-app=cilium -o wide
kubectl get pod -n kube-system -l k8s-app=calico-node -o wide
kubectl get pod -n kube-system -l k8s-app=flannel -o wide

# Check node for MemoryPressure / DiskPressure
kubectl describe node <node-name> | grep -A5 "Conditions:"

# Restart kube-proxy to resync iptables/IPVS rules
kubectl rollout restart daemonset/kube-proxy -n kube-system
```

### Ingress

```bash
# Check ingress address assignment
kubectl get ingress -n <namespace>

# Describe ingress rules and address
kubectl describe ingress <ingress-name> -n <namespace>

# Check nginx ingress controller logs
kubectl logs -n ingress-nginx <controller-pod> | tail -50
```

### Ephemeral Debug Pods

```bash
# Launch ephemeral debug pod with network tools
kubectl run netdebug --image=nicolaka/netshoot --rm -it --restart=Never -- bash

# Attach ephemeral container sharing pod's network namespace
kubectl debug -it <pod-name> --image=nicolaka/netshoot \
  --target=<container-name>

# Launch busybox for basic DNS/network testing
kubectl run -it --rm debug --image=busybox --restart=Never
```

---

## `istioctl` — Istio service mesh CLI

**When to use:** Diagnose Istio proxy synchronization, inspect Envoy xDS configuration (listeners, clusters, routes, endpoints), check AuthorizationPolicy enforcement, and manage multi-cluster mesh configuration.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `proxy-status` | Show all proxy xDS sync status with Istiod |
| `proxy-config <type> <pod>` | Show Envoy config (listeners/clusters/routes/endpoints) |
| `analyze` | Analyze Istio config for misconfigurations |
| `x authz check` | Check AuthorizationPolicies for a workload |
| `dashboard envoy` | Open Envoy admin UI in browser |
| `install` | Install or upgrade Istio |
| `create-remote-secret` | Create cross-cluster remote secret |

### Common Usage

```bash
# Check all proxy xDS sync status
istioctl proxy-status
```

**Output:**
```
NAME                              CLUSTER  CDS    LDS    EDS    RDS    ECDS   ISTIOD
frontend-7d4b-xyz.default         Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-xxx
backend-5c9f-abc.default          Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-xxx
```

### Important Variants

```bash
# Show route config for a deployment's sidecar proxy
istioctl proxy-config routes deploy/frontend -n default

# Show cluster config (upstream services)
istioctl proxy-config clusters deploy/frontend -n default

# Show listener config (inbound/outbound ports)
istioctl proxy-config listeners deploy/frontend -n default

# Show endpoint config (healthy upstream IPs)
istioctl proxy-config endpoints deploy/frontend -n default

# Analyze all namespaces for config issues
istioctl analyze --all-namespaces

# Check which AuthorizationPolicies apply to a workload
istioctl x authz check deploy/backend -n default

# Open Envoy admin dashboard in browser
istioctl dashboard envoy deploy/frontend

# Check proxy versions during upgrade (detect mixed versions)
istioctl proxy-status | awk '{print $NF}' | sort | uniq -c

# Install a canary Istio revision alongside existing
istioctl install --set revision=1-24 --set profile=default

# Migrate a namespace to a new Istio revision
kubectl label namespace staging istio.io/rev=1-24 istio-injection-
kubectl rollout restart deploy -n staging

# Roll back namespace to previous Istio revision
kubectl label namespace staging istio.io/rev=1-22 --overwrite

# Create remote secret for multi-cluster primary mesh
istioctl create-remote-secret --name=cluster2 \
  --kubeconfig=/path/to/cluster2.kubeconfig | kubectl apply -f -
```

---

## `cilium` — Cilium CNI CLI

**When to use:** Inspect Cilium eBPF policy maps, connection tracking, load balancing state, endpoint identity, and ClusterMesh connectivity. Complements `hubble` for policy enforcement debugging.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `status` | Show overall Cilium agent health |
| `monitor` | Live packet event monitor |
| `bpf policy list` | List eBPF network policy maps |
| `bpf lb list` | Show load balancing BPF map entries |
| `bpf ct list global` | Show connection tracking map |
| `bpf endpoint list` | Show endpoint identity map |
| `connectivity test` | Run built-in connectivity test suite |
| `clustermesh status` | Verify ClusterMesh connectivity |
| `clustermesh connect` | Connect two Cilium clusters |
| `install` | Install Cilium into a cluster |
| `hubble enable` | Enable Hubble observability relay |

### Common Usage

```bash
# Check overall Cilium agent health
cilium status --wait
```

**Output:**
```
    /¯¯\__/¯¯\    Cilium:         OK
 /¯¯\__/¯¯\__/   Operator:       OK
 \__/¯¯\__/¯¯/   Hubble Relay:   OK
    \__/¯¯\__/    ClusterMesh:    OK
```

### Important Variants

```bash
# Show packets dropped by NetworkPolicy in real time
cilium monitor --type drop

# List Cilium eBPF network policy maps
cilium bpf policy list

# Show Cilium service load balancing BPF map entries
cilium bpf lb list

# Show Cilium connection tracking map (replaces conntrack for Cilium traffic)
cilium bpf ct list global

# Show Cilium endpoint identity map
cilium bpf endpoint list

# Run built-in connectivity tests across the cluster
cilium connectivity test

# Install Cilium version 1.14.0
cilium install --version 1.14.0

# Enable Hubble relay and UI
cilium hubble enable --ui

# Connect two clusters for ClusterMesh
cilium clustermesh connect --destination-context cluster2

# Verify ClusterMesh connectivity status
cilium clustermesh status

# Export a Kubernetes service for cross-cluster discovery
kubectl annotate service backend service.cilium.io/global=true
```

---

## `hubble` — Cilium network observability

**When to use:** Observe real-time network flows in a Cilium-managed cluster. Filter by namespace, pod, verdict (DROPPED/FORWARDED), protocol (DNS, HTTP), and build traffic matrices for security auditing.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `observe` | Stream or query network flows |
| `status` | Show Hubble relay status and connected node count |
| `--namespace` | Filter flows by namespace |
| `--pod` | Filter by pod name |
| `--verdict DROPPED` | Show only dropped flows |
| `--verdict FORWARDED` | Show only forwarded flows |
| `--protocol dns/http/tcp` | Filter by L7/L4 protocol |
| `--follow` | Stream flows continuously |
| `--output json` | JSON output (for piping to jq) |
| `--output jsonpb` | JSON protobuf output (richer L7 data) |
| `--from-pod <ns/pod>` | Filter by source pod |
| `--to-pod <ns/pod>` | Filter by destination pod |
| `--type drop` | Show dropped flows with drop reason |

### Common Usage

```bash
# Start Hubble port-forward (required before using CLI)
cilium hubble port-forward &

# Check Hubble relay connectivity
hubble status
```

**Output:**
```
Healthcheck (via localhost:4245): Ok
Current/Max Flows: 12,285/16,383 (74.98%)
Flows/s: 185.34
Connected Nodes: 3/3
```

### Important Variants

```bash
# Observe all network flows in default namespace
hubble observe --namespace default

# Show only dropped flows (diagnose NetworkPolicy violations)
hubble observe --verdict DROPPED --namespace default

# Real-time drop monitoring with source, dest, and drop reason
hubble observe --verdict DROPPED --follow --output json | \
  jq '{src: .flow.source.pod_name, dst: .flow.destination.pod_name, reason: .flow.drop_reason_desc}'

# Watch HTTP traffic to a specific pod (L7 visibility)
hubble observe --pod frontend --protocol http --output json

# Monitor DNS flows and extract DNS layer data
hubble observe --protocol dns --output jsonpb | jq '.flow.l7.dns'

# Show all DNS queries and responses in a namespace
hubble observe --protocol dns --namespace payments

# Filter by source pod
hubble observe --from-pod payments/client --follow

# Build traffic matrix: flow counts per source-destination pair
hubble observe --namespace default --output json | \
  jq -r '[.flow.source.pod_name, .flow.destination.pod_name, .flow.verdict] | @csv' | \
  sort | uniq -c | sort -rn

# Check drops in payments namespace (incident triage)
hubble observe --verdict DROPPED --namespace payments \
  --follow --output json | head -20
```

---

## `aws` — AWS CLI (networking-focused)

**When to use:** Query and modify AWS network infrastructure: VPCs, subnets, security groups, route tables, load balancers, Route53 DNS, and EKS cluster networking. The primary tool for cloud-layer connectivity debugging in AWS.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `--filters` | Server-side filtering (key-value pairs) |
| `--query` | JMESPath expression to filter/shape output |
| `--output json/table/text` | Output format |
| `--region` | AWS region override |
| `--profile` | Named credentials profile |

### Common Usage

```bash
# List all VPCs with their CIDR blocks
aws ec2 describe-vpcs --query \
  'Vpcs[*].{ID:VpcId,CIDR:CidrBlock,Name:Tags[?Key==`Name`]|[0].Value}'
```

**Output:**
```json
[
  { "ID": "vpc-0abc1234", "CIDR": "10.0.0.0/16",  "Name": "prod-vpc" },
  { "ID": "vpc-0def5678", "CIDR": "172.16.0.0/12", "Name": "dev-vpc" }
]
```

### EC2 & VPC Inspection

```bash
# List subnets with available IP counts
aws ec2 describe-subnets \
  --query 'Subnets[*].{ID:SubnetId,CIDR:CidrBlock,Available:AvailableIpAddressCount,AZ:AvailabilityZone}'

# Get security groups attached to an EC2 instance
aws ec2 describe-instances --instance-ids i-XXXXXXXXXXXXXXXXX \
  --query 'Reservations[*].Instances[*].SecurityGroups'

# Check inbound rules on a security group
aws ec2 describe-security-groups --group-ids sg-XXXXXXXXXXXXXXXXX \
  --query 'SecurityGroups[*].IpPermissions'

# Check outbound rules on a security group
aws ec2 describe-security-groups --group-ids sg-XXXXXXXXXXXXXXXXX \
  --query 'SecurityGroups[*].IpPermissionsEgress'

# Find NSG rules by port/CIDR using JQ
aws ec2 describe-security-groups --group-ids $SG_ID | \
  jq -r '.SecurityGroups[0].IpPermissions[] |
    select(.IpRanges[0].CidrIp == "10.0.0.0/8") | .FromPort'

# Remove overly broad ingress rule
aws ec2 revoke-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 0-65535 --cidr 10.0.0.0/8

# Add a precise ingress rule (minimal access)
aws ec2 authorize-security-group-ingress --group-id $CDE_SG_ID \
  --protocol tcp --port 8443 --source-group $PAYMENT_PROXY_SG_ID

# Check NACL rules on a subnet
aws ec2 describe-network-acls \
  --filters Name=association.subnet-id,Values=subnet-isolated-a

# Get routes for a subnet's route table
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-XXXXXXXXX" \
  --query 'RouteTables[*].Routes'

# Add a VPC peering route to a route table
aws ec2 create-route --route-table-id rtb-XXXXXXXX \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-XXXXXXXXXXXXXXXXX

# Add a Transit Gateway route
aws ec2 create-route --route-table-id rtb-XXXXXXXX \
  --destination-cidr-block 10.1.0.0/16 \
  --transit-gateway-id tgw-XXXXXXXXXXXXXXXXX

# List active VPC peering connections with CIDR info
aws ec2 describe-vpc-peering-connections \
  --filters "Name=status-code,Values=active" \
  --query 'VpcPeeringConnections[*].{ID:VpcPeeringConnectionId,
    Status:Status.Code,Requester:RequesterVpcInfo.CidrBlock,
    Accepter:AccepterVpcInfo.CidrBlock}'

# Check TGW attachment states
aws ec2 describe-transit-gateway-attachments \
  --filters "Name=transit-gateway-id,Values=tgw-XXXXXXXX" \
  --query 'TransitGatewayAttachments[*].{VPC:ResourceId,State:State}'

# Check TGW route table for active routes
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id tgw-rtb-XXXXXXXX \
  --filters "Name=state,Values=active"

# Enforce IMDSv2 on EC2 to block SSRF
aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890abcdef0 \
  --http-tokens required --http-put-response-hop-limit 1
```

### ELB / Load Balancers

```bash
# Check ALB idle timeout (default 60s; causes 504 on long requests)
aws elbv2 describe-load-balancer-attributes --load-balancer-arn arn:... \
  --query 'Attributes[?Key==`idle_timeout.timeout_seconds`]'

# Increase ALB idle timeout to 300s
aws elbv2 modify-load-balancer-attributes --load-balancer-arn arn:... \
  --attributes Key=idle_timeout.timeout_seconds,Value=300

# Enable ALB access logs to S3
aws elbv2 modify-load-balancer-attributes --load-balancer-arn arn:... \
  --attributes Key=access_logs.s3.enabled,Value=true \
               Key=access_logs.s3.bucket,Value=my-alb-logs

# Reduce target deregistration delay for fast-response APIs
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

### Route53 & DNS

```bash
# List Route53 private hosted zones for a VPC
aws route53 list-hosted-zones-by-vpc \
  --vpc-id vpc-xxxxx --vpc-region us-east-1

# Associate a private hosted zone with a VPC
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z1234567890ABC \
  --vpc VPCRegion=us-east-1,VPCId=vpc-xxxxx

# Create Route53 Resolver outbound endpoint for hybrid DNS
aws route53resolver create-resolver-endpoint \
  --creator-request-id unique-$(date +%s) \
  --direction OUTBOUND \
  --ip-addresses SubnetId=subnet-a,Ip=10.0.1.100 SubnetId=subnet-b,Ip=10.0.2.100 \
  --security-group-ids sg-resolver-xxx

# Create DNS forwarding rule for on-premises domain
aws route53resolver create-resolver-rule \
  --rule-type FORWARD \
  --domain-name corp.example.com \
  --resolver-endpoint-id rslvr-out-xxx \
  --target-ips Ip=192.168.1.53,Port=53 Ip=192.168.2.53,Port=53

# Check VPN tunnel status and last change
aws ec2 describe-vpn-connections \
  --vpn-connection-id vpn-xxx \
  --query 'VpnConnections[].VgwTelemetry'
```

### VPC Reachability Analyzer

```bash
# Create reachability analysis path (e.g., Lambda to RDS)
aws ec2 create-network-insights-path \
  --source eni-lambda-xxx --destination eni-rds-xxx \
  --protocol tcp --destination-port 5432

# Run the analysis
aws ec2 start-network-insights-analysis \
  --network-insights-path-id nip-xxx
```

### EKS

```bash
# Get EKS auth token (for kubeconfig)
aws eks get-token --cluster-name my-cluster

# Create EKS cluster with IPv6-only pod networking
aws eks create-cluster --name my-cluster \
  --kubernetes-network-config ipFamily=ipv6
```

---

## `az` — Azure CLI (networking-focused)

**When to use:** Manage and debug Azure network resources: VNets, NSGs, load balancers, Application Gateway, VNet peering, and AKS networking. Essential for Azure cloud-layer incident response.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-g` / `--resource-group` | Target resource group |
| `-n` / `--name` | Resource name |
| `--query` | JMESPath expression |
| `--output table/json/tsv` | Output format |

### Common Usage

```bash
# Show effective NSG deny rules on an Azure NIC
az network nic show-effective-nsg \
  --name $NIC_NAME --resource-group $RG | \
  jq '.effectiveSecurityRules[] | select(.access == "Deny") |
    {name, protocol, srcPort: .sourcePortRange,
     dstPort: .destinationPortRange, src: .sourceAddressPrefix}'
```

**Output:**
```json
{
  "name": "DenyAll",
  "protocol": "*",
  "srcPort": "0-65535",
  "dstPort": "0-65535",
  "src": "*"
}
```

### VNet & NSG

```bash
# Add a second CIDR block to an existing VNet
az network vnet update -g prod-rg -n prod-vnet \
  --add addressSpace.addressPrefixes "172.20.0.0/16"

# Create an NSG rule using Application Security Groups (ASGs)
az network nsg rule create -g prod-rg --nsg-name prod-nsg \
  --name AllowWebToDb --priority 200 \
  --source-asgs asg-web-tier --destination-asgs asg-db-tier \
  --destination-port-ranges 5432 --protocol Tcp --access Allow

# Show effective routes for a NIC (system + UDR)
az network nic show-effective-route-table \
  --resource-group prod-rg --name prod-vm-nic --output table

# Show merged NSG rules (subnet + NIC) in priority order
az network nic list-effective-nsg \
  --resource-group prod-rg --name prod-vm-nic

# Azure Network Watcher IP Flow Verify
az network watcher test-ip-flow \
  --vm prod-vm --direction Inbound \
  --protocol TCP --local 10.0.1.10:443 \
  --remote 1.2.3.4:52000 --resource-group prod-rg

# Enable NSG flow logs for visibility/forensics
az network watcher flow-log create \
  --resource-group prod-rg --name prod-nsg-flowlog --nsg prod-nsg

# Create UDR to force all spoke traffic through Azure Firewall
az network route-table route create -g prod-rg \
  --route-table-name spoke1-udr -n default-to-firewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.4

# Create private DNS zone
az network private-dns zone create -g prod-rg -n "internal.contoso.com"

# Link private DNS zone to VNet with autoregistration
az network private-dns link vnet create -g prod-rg \
  -n prod-vnet-link -z "internal.contoso.com" \
  -v prod-vnet --registration-enabled true

# Create Private Endpoint for a storage account
az network private-endpoint create -g prod-rg -n pe-prodstorage \
  --vnet-name prod-vnet --subnet pe-subnet \
  --private-connection-resource-id /subscriptions/.../storageAccounts/prodstorage \
  --group-id blob --connection-name storage-pe-conn
```

### VNet Peering & Connectivity

```bash
# Create hub-to-spoke peering with gateway transit
az network vnet peering create -g prod-rg -n hub-to-spoke \
  --vnet-name hub-vnet --remote-vnet spoke-vnet \
  --allow-gateway-transit true

# Create spoke-to-hub peering using remote gateways
az network vnet peering create -g prod-rg -n spoke-to-hub \
  --vnet-name spoke-vnet --remote-vnet hub-vnet \
  --use-remote-gateways true

# List VNet peering state (both sides must show Connected)
az network vnet peering list -g prod-rg --vnet-name hub-vnet -o table

# Check ExpressRoute circuit provisioning state
az network express-route show -g prod-rg -n prod-er-circuit \
  --query "circuitProvisioningState"
```

### Load Balancers & Application Gateway

```bash
# Check AppGW backend health (identify unhealthy targets causing 502)
az network application-gateway show-backend-health \
  -g prod-rg -n prod-appgw \
  --query "backendAddressPools[?name=='api-backend-pool'].backendHttpSettingsCollection[].servers[]"

# Create WAF policy for Application Gateway
az network application-gateway waf-policy create \
  -g prod-rg -n prod-waf-policy --location eastus

# Set Application Gateway WAF to Prevention mode
az network application-gateway waf-policy policy-setting update \
  -g prod-rg --policy-name prod-waf-policy \
  --mode Prevention --state Enabled

# Configure outbound SNAT rule with multiple PIPs
az network lb outbound-rule create -g prod-rg --lb-name prod-lb \
  -n outbound-rule --address-pool backend-pool \
  --frontend-ip-configs pip1-frontend pip2-frontend pip3-frontend \
  --protocol All --idle-timeout 15 --outbound-ports 10000
```

### AKS Networking

```bash
# Create AKS cluster with Azure CNI Overlay and Calico policy
az aks create -g prod-rg -n prod-cluster \
  --network-plugin azure --network-policy calico \
  --network-plugin-mode overlay

# Create AKS cluster with Cilium eBPF dataplane
az aks create -g prod-rg -n prod-cluster \
  --network-plugin azure --network-dataplane cilium \
  --network-plugin-mode overlay

# Check IP allocation count in AKS subnet (IP exhaustion diagnosis)
az network vnet subnet show -g MC_prod-rg_prod-cluster_eastus \
  -n aks-subnet --vnet-name prod-vnet \
  --query "ipConfigurations | length(@)"

# Create private AKS cluster
az aks create -g prod-rg -n prod-cluster \
  --enable-private-cluster --private-dns-zone system
```

---

## `docker` — Container networking

**When to use:** Inspect Docker container network namespaces, create custom networks, diagnose inter-container connectivity, and retrieve container PIDs for `nsenter` debugging.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `network ls` | List all Docker networks |
| `network inspect <name>` | Detailed network info (subnet, gateway, containers) |
| `network create` | Create a custom network |
| `inspect --format '{{.State.Pid}}'` | Get container PID for nsenter |
| `exec -it <c> <cmd>` | Execute a command in a running container |
| `run --network <net>` | Connect container to a specific network |
| `--network host` | Use host network namespace |
| `--network none` | Disable networking for container |

### Common Usage

```bash
# List all Docker networks
docker network ls
```

**Output:**
```
NETWORK ID     NAME      DRIVER    SCOPE
abc123def456   bridge    bridge    local
fed987cba654   host      host      local
123abc456def   none      null      local
789xyz012uvw   myapp     bridge    local
```

### Important Variants

```bash
# Inspect bridge network subnet
docker network inspect bridge | grep Subnet

# Show full network details including connected containers
docker network inspect myapp

# Create an isolated network with custom subnet
docker network create --driver bridge \
  --subnet 172.18.0.0/16 --gateway 172.18.0.1 \
  myapp-net

# Run container on a specific network
docker run --network myapp-net --name backend myapp:latest

# Run container with host networking (no isolation)
docker run --network host nginx:alpine

# Run container with no networking
docker run --network none my-batch-job:latest

# Get container PID for nsenter
docker inspect --format '{{.State.Pid}}' <container_name>

# Execute shell inside running container
docker exec -it <container_name> bash

# Execute network diagnostic from inside container
docker exec -it <container_name> ss -tnlp

# Show container IP address
docker inspect -f '{{.NetworkSettings.IPAddress}}' <container_name>
```

---

## `crictl` — Container runtime CLI

**When to use:** Inspect containers managed by a CRI-compatible runtime (containerd, CRI-O) in Kubernetes nodes where Docker is not present. Primarily used to get container PIDs for `nsenter` network namespace debugging.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `ps` | List running containers |
| `inspect <id>` | Get full container metadata including PID |
| `pods` | List pods on the node |
| `logs <id>` | Stream container logs |
| `exec -it <id> <cmd>` | Execute command in container |
| `images` | List container images |
| `stats` | Show container resource usage |

### Common Usage

```bash
# List running containers on the node
crictl ps
```

**Output:**
```
CONTAINER      IMAGE          CREATED      STATE    NAME         ATTEMPT  POD ID
a1b2c3d4e5f6   myapp:v1.2     2 hours ago  Running  app          0        pod-xyz123
7890abcdef01   nginx:1.25     3 hours ago  Running  sidecar      0        pod-xyz123
```

### Important Variants

```bash
# Find container ID by pod name
crictl ps | grep mypod

# Get container PID for nsenter network namespace entry
crictl inspect $CONTAINER_ID | jq '.info.pid'

# Alternatively using the nested JSON format
crictl inspect $CONTAINER_ID | grep -i '"pid"' | head -1

# List all pods on the node
crictl pods

# Stream container logs
crictl logs -f $CONTAINER_ID

# Execute command inside container
crictl exec -it $CONTAINER_ID sh

# Show container resource usage
crictl stats

# List container images
crictl images
```

---

## `nsenter` — Enter process namespaces

**When to use:** Run diagnostic tools (ip, ss, tcpdump) inside a container's network namespace from the host, without needing the tools installed in the container. Works for both Docker and Kubernetes (via `crictl` to get the PID).

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-t <pid>` | Target process PID |
| `-n` | Enter network namespace |
| `-u` | Enter UTS (hostname) namespace |
| `-i` | Enter IPC namespace |
| `-m` | Enter mount namespace |
| `-p` | Enter PID namespace |
| `--all` | Enter all namespaces of the target process |

### Common Usage

```bash
# Get container PID (crictl or docker)
CONTAINER_ID=$(crictl ps | grep mypod | awk '{print $1}')
PID=$(crictl inspect $CONTAINER_ID | jq '.info.pid')

# Enter container network namespace and show IP addresses
nsenter -t $PID -n ip addr
```

**Output:**
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
    inet 127.0.0.1/8 scope host lo
2: eth0@if42: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450
    inet 10.244.1.5/24 brd 10.244.1.255 scope global eth0
```

### Important Variants

```bash
# Show routes inside container network namespace
nsenter -t $PID -n ip route

# Show listening sockets inside container
nsenter -t $PID -n ss -tlnp

# Capture packets inside container using host's tcpdump
nsenter -t $PID -n tcpdump -i eth0 -w /tmp/capture.pcap

# Enter all namespaces (full container isolation)
nsenter -t $PID --all

# Enter network and UTS namespace together
nsenter -t $PID -n -u

# Read pod's resolv.conf to verify NodeLocal DNS config
nsenter -t $POD_PID -n cat /etc/resolv.conf

# Get container-side interface index (to match veth pair on host)
nsenter -t $CTR_PID -n ip link show eth0 | grep -oP 'if\K[0-9]+'
```

---

## `dmesg` — Kernel ring buffer

**When to use:** Inspect kernel-level messages for network interface errors, conntrack table overflow, OOM kills, SYN flood detection, and NIC driver issues. The ground truth for low-level kernel events.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-T` | Human-readable timestamps |
| `-H` | Human-readable with auto-colorized output |
| `--level err,warn` | Filter by log level |
| `-k` | Kernel messages only |
| `-w` | Follow new messages (like `tail -f`) |
| `| grep <pattern>` | Search for specific events |

### Common Usage

```bash
# Check kernel messages for network interface errors
dmesg -T | grep -i "eth0\|ens\|bond\|dropped\|error" | tail -20
```

**Output:**
```
[2025-03-27 10:14:32] eth0: renamed from veth1234abc
[2025-03-27 10:15:11] nf_conntrack: table full, dropping packet
[2025-03-27 10:15:12] nf_conntrack: table full, dropping packet
```

### Important Variants

```bash
# Check for conntrack table overflow (causes connection drops)
dmesg | grep "nf_conntrack: table full"

# Check for SYN flood detection and SYN cookie activation
dmesg | grep "SYN flooding"

# Check for OOM kills (memory pressure causes latency)
dmesg | grep 'oom-kill'

# Check NIC link state changes
dmesg | grep -i "eth0\|link\|carrier"

# Watch new kernel messages in real time
dmesg -Tw

# Show only error and warning messages with timestamps
dmesg -T --level err,warn
```

---

## `journalctl` — Systemd logs (network-related filtering)

**When to use:** Query systemd journal for service-specific logs, kubelet CNI errors, and network service events. Prefer over reading raw log files as it supports structured filtering, time ranges, and kernel log integration.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-u <unit>` | Filter by systemd unit (service) |
| `--since <time>` | Start time (`"30 minutes ago"`, `"2025-03-27 10:00"`) |
| `--until <time>` | End time |
| `-f` | Follow new log entries |
| `-k` | Kernel messages only (equivalent to dmesg) |
| `-n <lines>` | Last N lines |
| `-p err` | Filter by priority (emerg/alert/crit/err/warning/notice/info/debug) |
| `-b` | Current boot only |
| `--no-pager` | Output without pager (useful in scripts) |
| `-o json` | JSON output for structured parsing |

### Common Usage

```bash
# Check systemd service logs from the last 30 minutes
journalctl -u myservice --since "30 minutes ago"
```

**Output:**
```
Mar 27 10:14:32 node-1 myservice[1234]: Failed to connect to 10.0.1.50:5432
Mar 27 10:14:33 node-1 myservice[1234]: Retrying connection (attempt 3/5)
Mar 27 10:15:01 node-1 myservice[1234]: Connection established
```

### Important Variants

```bash
# Check kubelet logs for CNI errors during K8s networking issues
journalctl -u kubelet --since "30 minutes ago" | grep -i "cni\|network"

# Stream a service's logs
journalctl -u nginx -f

# Check iptables LOG rule output from kernel journal
journalctl -k | grep "IPT-DEBUG"

# Check kernel messages for a specific unit (systemd-networkd)
journalctl -u systemd-networkd --since "1 hour ago" -p warning

# Show last 100 lines of a service log
journalctl -u docker -n 100

# Filter to current boot
journalctl -u kubelet -b -p err

# JSON output for structured parsing
journalctl -u myservice --since "10 minutes ago" -o json | \
  jq '.MESSAGE' | tail -20
```

---

## `systemctl` — Service management (network services)

**When to use:** Check, start, stop, and restart network-critical services. Use during service-not-reachable triage to confirm the service process is running before checking ports and connectivity.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `status <unit>` | Show service state, recent logs, PID |
| `start <unit>` | Start a stopped service |
| `stop <unit>` | Stop a running service |
| `restart <unit>` | Restart a service |
| `reload <unit>` | Reload configuration without restart |
| `enable <unit>` | Enable service to start at boot |
| `disable <unit>` | Prevent service from starting at boot |
| `is-active <unit>` | Returns 0 if active, non-zero otherwise |
| `list-units --state=failed` | List all failed units |
| `cat <unit>` | Show the unit file definition |

### Common Usage

```bash
# Check service status (first step in service-not-reachable debugging)
systemctl status nginx
```

**Output:**
```
● nginx.service - A high performance web server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
   Active: active (running) since Thu 2025-03-27 10:00:00 UTC; 2h 15min ago
 Main PID: 1234 (nginx)
    Tasks: 5
   Memory: 12.4M
```

### Important Variants

```bash
# List all currently failed services
systemctl list-units --state=failed

# Restart a network service
systemctl restart systemd-networkd

# Check if kubelet is active (exit 0 = running)
systemctl is-active kubelet

# Reload nginx config without dropping connections
systemctl reload nginx

# Enable Docker to start at boot
systemctl enable docker

# Check systemd-resolved (DNS resolver) status
systemctl status systemd-resolved

# Restart containerd (if CRI is unresponsive)
systemctl restart containerd
```

---

## `vtysh` — FRRouting/Quagga CLI (BGP, OSPF)

**When to use:** Inspect and configure BGP sessions, routing tables, RPKI validation, BFD sessions, and OSPF state on FRRouting-based routers or on Kubernetes nodes using Calico/Cilium BGP mode.

### Key Options (Interactive Mode)

| Command | Description |
|---------|-------------|
| `show bgp summary` | BGP session states and prefix counts |
| `show bgp neighbors <ip>` | Detailed BGP neighbor info |
| `show bgp ipv4 unicast` | Full BGP RIB (all paths) |
| `show bgp ipv4 unicast <prefix>` | Route details for specific prefix |
| `show bgp ipv4 unicast neighbors <ip> routes` | Routes received from neighbor |
| `show bgp ipv4 unicast neighbors <ip> advertised-routes` | Routes sent to neighbor |
| `show rpki prefix-table` | RPKI validated prefix table |
| `show rpki cache-connection` | RPKI validator connection status |
| `show bfd peers` | BFD session states and intervals |
| `show ip route` | Linux routing table (FIB) |
| `debug bgp updates` | Enable BGP UPDATE debug logging |
| `debug bgp neighbor-events` | Enable BGP session event logging |

### Common Usage

```bash
# Check all BGP session states and prefix counts
vtysh -c "show bgp summary"
```

**Output:**
```
IPv4 Unicast Summary:
BGP router identifier 10.0.0.1, local AS number 64512

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.1.1     4 64513   12345   11234        0    0    0 2d10h00s        150000
192.168.2.1     4 65001    4567    4123        0    0    0 1d05h30s          8192
```

### Important Variants

```bash
# Show detailed BGP neighbor info including hold timers
vtysh -c "show bgp neighbors 192.168.1.1"

# Check RPKI validation state for a specific prefix
vtysh -c "show bgp ipv4 unicast 1.1.1.0/24"

# Show BGP route details including all path attributes
vtysh -c "show bgp ipv4 unicast 10.0.0.0/8"

# Show all routes received from a specific BGP neighbor
vtysh -c "show bgp ipv4 unicast neighbors 192.168.1.1 routes"

# Show all routes advertised to a specific BGP neighbor
vtysh -c "show bgp ipv4 unicast neighbors 192.168.1.1 advertised-routes"

# View BGP RIB (first 100 entries)
vtysh -c "show bgp ipv4 unicast" | head -100

# Enable BGP UPDATE debug (use on low-traffic sessions only)
vtysh -c "debug bgp updates"

# Enable BGP neighbor event debug logging
vtysh -c "debug bgp neighbor-events"

# Show RPKI validated prefix table
vtysh -c "show rpki prefix-table"

# Check RPKI validator cache connection
vtysh -c "show rpki cache-connection"

# Show BFD session states (fast failure detection)
vtysh -c "show bfd peers"

# Announce more-specific prefixes to win over route leaks
vtysh << 'EOF'
conf t
router bgp 64512
  address-family ipv4 unicast
    network 203.0.113.0/25
    network 203.0.113.128/25
EOF
```

---

## `trivy` — Container vulnerability scanner

**When to use:** Scan container images for OS package CVEs, application dependency vulnerabilities, misconfigurations, and embedded secrets before or after deployment. Integrates into CI/CD pipelines.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `image <name>` | Scan a container image |
| `fs <path>` | Scan a filesystem or directory |
| `repo <url>` | Scan a Git repository |
| `k8s` | Scan a Kubernetes cluster |
| `--scanners <types>` | Comma-separated: vuln, secret, misconfig, license |
| `--severity <level>` | Filter: UNKNOWN, LOW, MEDIUM, HIGH, CRITICAL |
| `--format json/table/sarif` | Output format |
| `--output <file>` | Write report to file |
| `--ignore-unfixed` | Skip vulnerabilities without available fix |
| `--exit-code 1` | Return non-zero exit if vulnerabilities found (CI use) |

### Common Usage

```bash
# Scan a container image for vulnerabilities and secrets
trivy image myapp:v1.2.3
```

**Output:**
```
myapp:v1.2.3 (debian 11.6)
Total: 8 (HIGH: 3, CRITICAL: 2, MEDIUM: 2, LOW: 1)

┌──────────────┬───────────────┬──────────┬────────────────────────┬───────────────┐
│   Library    │ Vulnerability │ Severity │    Installed Version   │ Fixed Version │
├──────────────┼───────────────┼──────────┼────────────────────────┼───────────────┤
│ libssl1.1    │ CVE-2022-0778 │ CRITICAL │ 1.1.1n-0+deb11u3      │ 1.1.1n-0+deb11u4 │
│ zlib1g       │ CVE-2022-37434│ CRITICAL │ 1:1.2.11.dfsg-2       │ 1:1.2.11.dfsg-2+deb11u2 │
└──────────────┴───────────────┴──────────┴────────────────────────┴───────────────┘
```

### Important Variants

```bash
# Scan for embedded secrets in image layers
trivy image --scanners secret myapp:v1.2.3

# Scan for vulnerabilities only (high and critical)
trivy image --scanners vuln --severity HIGH,CRITICAL myapp:v1.2.3

# Scan and write JSON report for SIEM ingestion
trivy image --format json --output report.json myapp:v1.2.3

# Scan in CI: exit with code 1 if CRITICAL vulns found
trivy image --exit-code 1 --severity CRITICAL myapp:v1.2.3

# Skip vulnerabilities without available fix (reduce noise)
trivy image --ignore-unfixed --severity HIGH,CRITICAL myapp:v1.2.3

# Scan a local filesystem (e.g., application directory)
trivy fs --scanners vuln,secret ./app/

# Scan a running Kubernetes cluster
trivy k8s --report summary cluster

# Scan for misconfigurations in a Kubernetes manifest
trivy config ./k8s/manifests/
```

---

## `vault` — HashiCorp Vault CLI

**When to use:** Manage dynamic secrets, PKI certificates, and Kubernetes authentication integration. Use for issuing short-lived TLS certificates, configuring K8s service account auth, and creating scoped policies for database credential access.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `auth enable <method>` | Enable an auth method (kubernetes, aws, etc.) |
| `auth list` | List enabled auth methods |
| `write <path> <params>` | Write data to a Vault path |
| `read <path>` | Read data from a Vault path |
| `policy write <name>` | Create or update a Vault policy |
| `policy read <name>` | Read a policy definition |
| `secrets enable <engine>` | Enable a secrets engine |
| `secrets list` | List enabled secrets engines |
| `kv get <path>` | Read a KV secret |
| `kv put <path>` | Write a KV secret |
| `lease revoke <id>` | Revoke a dynamic secret lease |
| `token lookup` | Inspect the current token |
| `-address` | Vault server address (or use VAULT_ADDR env) |

### Common Usage

```bash
# Enable Kubernetes auth method
vault auth enable kubernetes

# Configure Kubernetes auth with cluster CA and reviewer JWT
vault write auth/kubernetes/config \
  kubernetes_host="https://$K8S_HOST" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
```

### Important Variants

```bash
# Create a Vault role mapping K8s service account to Vault policy
vault write auth/kubernetes/role/payments-api \
  bound_service_account_names=payments-sa \
  bound_service_account_namespaces=payments \
  policies=payments-db-policy \
  ttl=1h

# Create a Vault policy granting read access to dynamic DB credentials
vault policy write payments-db-policy - <<'EOF'
path "database/creds/payments-role" {
  capabilities = ["read"]
}
EOF

# Issue a short-lived (5-minute) TLS certificate from Vault PKI engine
vault write pki/issue/payments-role \
  common_name="payments-api.payments.svc.cluster.local" \
  ttl=5m

# Read a static KV secret
vault kv get prod/payments/database

# Test K8s service account access from a pod
vault write auth/kubernetes/login \
  role=payments-api \
  jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# List all enabled auth methods
vault auth list

# List enabled secrets engines
vault secrets list

# Revoke a dynamic secret lease immediately
vault lease revoke <lease-id>

# Inspect current token permissions
vault token lookup
```
