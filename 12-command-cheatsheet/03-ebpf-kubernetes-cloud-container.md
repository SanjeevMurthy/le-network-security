# Network & Security Command Cheatsheet

> Part 3 of 3 — **eBPF, Kubernetes, Cloud & Container**
> | [Part 1: Linux Network Core](./01-linux-network-core.md) | [Part 2: Capture / DNS / HTTP / TLS / Firewall](./02-capture-dns-http-tls-firewall.md) |

---

## Table of Contents

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
