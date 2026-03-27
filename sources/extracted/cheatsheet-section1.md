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
