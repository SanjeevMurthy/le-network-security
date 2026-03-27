# Commands Extracted — Batch 3a
# Directories: 08-system-design/, 09-advanced-topics/

---

## Directory: 08-system-design/

### 01-secure-multi-tier-architecture.md

TOOL: aws
CMD: aws ec2 revoke-security-group-ingress
SOURCE: 08-system-design/01-secure-multi-tier-architecture.md
CONTEXT: Rollback unauthorized security group ingress rule
---

TOOL: aws
CMD: aws secretsmanager get-secret-value --secret-id prod/db/password
SOURCE: 08-system-design/01-secure-multi-tier-architecture.md
CONTEXT: Retrieve current secret value after rotation
---

TOOL: nslookup
CMD: nslookup <rds-endpoint>
SOURCE: 08-system-design/01-secure-multi-tier-architecture.md
CONTEXT: Verify RDS endpoint resolves to standby AZ after failover
---

### 03-multi-region-failover.md

TOOL: aws
CMD: aws rds failover-global-cluster
SOURCE: 08-system-design/03-multi-region-failover.md
CONTEXT: Promote Aurora Global DB secondary region to primary writer
---

### 04-cdn-edge-architecture.md

TOOL: aws
CMD: aws cloudfront create-invalidation --paths "/video/<content-id>/*"
SOURCE: 08-system-design/04-cdn-edge-architecture.md
CONTEXT: DMCA emergency cache invalidation for specific content
---

TOOL: aws
CMD: aws cloudfront create-invalidation --paths "/index.html" "/app*.js" "/app*.css"
SOURCE: 08-system-design/04-cdn-edge-architecture.md
CONTEXT: Invalidate changed web app files on deployment
---

---

## Directory: 09-advanced-topics/

### 01-ebpf-internals.md

TOOL: bpftool
CMD: bpftool btf dump file /sys/kernel/btf/vmlinux format c | grep "struct task_struct" | head -5
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Inspect kernel BTF type definitions for task_struct
---

TOOL: ls
CMD: ls /sys/kernel/btf/vmlinux
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Verify BTF is enabled in running kernel
---

TOOL: bpftool
CMD: bpftool map create /sys/fs/bpf/conn_count type hash key 4 value 8 entries 65536 name conn_count
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Create a pinned BPF hash map on the BPF filesystem
---

TOOL: bpftool
CMD: bpftool map dump id 42
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Dump all entries from a running BPF map by ID
---

TOOL: bpftool
CMD: bpftool map pin id 42 /sys/fs/bpf/my_map
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Pin BPF map to filesystem so it survives program termination
---

TOOL: bpftool
CMD: bpftool prog load tcp_trace.bpf.o /sys/fs/bpf/tcp_trace type fentry --attach-to tcp_connect
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Load and attach fentry BPF program to tcp_connect
---

TOOL: bpftool
CMD: bpftool prog list
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: List all loaded BPF programs on the system
---

TOOL: bpftool
CMD: bpftool prog show id 42 --pretty
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Show detailed info including verifier stats for a program
---

TOOL: bpftool
CMD: bpftool prog dump jited id 42
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Dump JIT-compiled x86-64 instructions for a BPF program
---

TOOL: strace
CMD: strace -e bpf bpftool prog load fail.bpf.o /dev/null 2>&1 | grep -A 50 "log"
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Show verifier log for a failed BPF program load
---

TOOL: bpftrace
CMD: bpftrace -e 'kprobe:__sys_bpf { printf("bpf syscall cmd=%d\n", arg0); }'
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Trace BPF syscall invocations with command number
---

TOOL: bpftool
CMD: bpftool prog tracelog
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Monitor BPF ring buffer output from userspace
---

TOOL: bpftool
CMD: bpftool map list
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: List all BPF maps and their memory usage
---

TOOL: bpftool
CMD: bpftool prog load prog.bpf.o /sys/fs/bpf/prog -d 2>&1 | grep "CO-RE"
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Verify CO-RE relocations were applied on program load
---

TOOL: bpftool
CMD: bpftool btf show
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: List all BTF objects loaded in the kernel
---

TOOL: ip
CMD: ip link show dev eth0
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Show XDP program attached status on interface
---

TOOL: bpftool
CMD: bpftool net list dev eth0
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: List BPF programs attached to a network device
---

TOOL: tc
CMD: tc filter show dev eth0 ingress
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Show TC-BPF programs attached on ingress
---

TOOL: bpftool
CMD: bpftool prog profile id 42 duration 10 cycles instructions
SOURCE: 09-advanced-topics/01-ebpf-internals.md
CONTEXT: Measure BPF program run-time performance counters
---

### 02-service-mesh-advanced.md

TOOL: kubectl
CMD: kubectl exec -it deploy/frontend -- curl -s localhost:15000/clusters | grep -A 10 "backend::circuit_breakers"
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Check Envoy circuit breaker state for a cluster
---

TOOL: curl
CMD: curl -s localhost:15000/config_dump | jq .
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Dump full Envoy configuration via admin API
---

TOOL: curl
CMD: curl -s localhost:15000/config_dump | jq '.configs[] | select(.["@type"] | contains("ClustersConfigDump")) | .dynamic_active_clusters[] | {name: .cluster.name, lb_policy: .cluster.lb_policy}'
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Extract cluster names and load balancing policies from config dump
---

TOOL: curl
CMD: curl -s localhost:15000/clusters
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Get live Envoy cluster stats including circuit breaker state
---

TOOL: curl
CMD: curl -s localhost:15000/clusters | grep "backend::" | grep -E "(healthy|cx_active|rq_active|rq_timeout)"
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Check specific endpoint health and active request counts
---

TOOL: curl
CMD: curl -s localhost:15000/config_dump | jq '.configs[] | select(.["@type"] | contains("ListenersConfigDump"))'
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Extract Envoy listener configuration from admin config dump
---

TOOL: curl
CMD: curl -s localhost:15000/stats | grep -E "(cx_active|rq_active|upstream_rq_retry)"
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Show active connections, requests, and retry counters
---

TOOL: curl
CMD: curl -s -X POST "localhost:15000/logging?level=info"
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Enable info-level access logging on Envoy temporarily
---

TOOL: istioctl
CMD: istioctl proxy-status
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Check all proxy xDS sync status with Istiod
---

TOOL: istioctl
CMD: istioctl analyze --all-namespaces
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Analyze Istio configuration for common misconfigurations
---

TOOL: istioctl
CMD: istioctl proxy-config routes deploy/frontend -n default
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Show route configuration for a specific deployment's proxy
---

TOOL: istioctl
CMD: istioctl proxy-config clusters deploy/frontend -n default
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Show cluster configuration for a specific deployment's proxy
---

TOOL: istioctl
CMD: istioctl proxy-config listeners deploy/frontend -n default
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Show listener configuration for a specific deployment's proxy
---

TOOL: istioctl
CMD: istioctl proxy-config endpoints deploy/frontend -n default
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Show endpoint configuration for a specific deployment's proxy
---

TOOL: istioctl
CMD: istioctl x authz check deploy/backend -n default
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Check which AuthorizationPolicies apply to a workload
---

TOOL: istioctl
CMD: istioctl dashboard envoy deploy/frontend
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Open Envoy admin dashboard in browser for a deployment
---

TOOL: kubectl
CMD: kubectl port-forward deploy/frontend 15000
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Port-forward to Envoy admin interface on port 15000
---

TOOL: kubectl
CMD: kubectl port-forward svc/kiali -n istio-system 20001:20001
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Port-forward to Kiali service graph UI
---

TOOL: spire-server
CMD: spire-server bundle show -format spiffe > bundle.jwks
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Export local SPIRE trust bundle for federation
---

TOOL: spire-server
CMD: spire-server bundle set -format spiffe -id spiffe://remote-cluster.local -path remote-bundle.jwks
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Import remote cluster's trust bundle for cross-cluster mTLS
---

TOOL: istioctl
CMD: istioctl create-remote-secret --name=cluster2 --kubeconfig=/path/to/cluster2.kubeconfig | kubectl apply -f -
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Create remote secret for multi-primary cluster API access
---

TOOL: cilium
CMD: cilium clustermesh connect --destination-context cluster2
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Connect two Cilium clusters for ClusterMesh
---

TOOL: cilium
CMD: cilium clustermesh status
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Verify Cilium ClusterMesh connectivity status
---

TOOL: kubectl
CMD: kubectl annotate service backend service.cilium.io/global=true
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Export a Kubernetes service for cross-cluster discovery
---

TOOL: istioctl
CMD: istioctl proxy-status | awk '{print $NF}' | sort | uniq -c
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Count proxy versions during Istio upgrade (detect mixed versions)
---

TOOL: kubectl
CMD: kubectl logs -n istio-system deploy/istiod | grep "NACKed"
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Check for xDS NACK errors indicating proxies rejected config
---

TOOL: kubectl
CMD: kubectl exec -it deploy/prometheus -n istio-system -- curl -s 'localhost:9090/api/v1/query?query=rate(istio_requests_total{response_code=~"5.."}[1m])'
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Query Prometheus for Istio 5xx error rates during upgrade
---

TOOL: istioctl
CMD: istioctl install --set revision=1-24 --set profile=default
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Install new Istiod as a canary revision alongside existing
---

TOOL: kubectl
CMD: kubectl -n istio-system get deploy istiod-1-24
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Verify new Istiod canary revision is healthy
---

TOOL: kubectl
CMD: kubectl label namespace staging istio.io/rev=1-24 istio-injection-
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Migrate a namespace to new Istio revision for canary testing
---

TOOL: kubectl
CMD: kubectl rollout restart deploy -n staging
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Roll out pods to pick up new Istio sidecar version
---

TOOL: kubectl
CMD: kubectl label namespace staging istio.io/rev=1-22 --overwrite
SOURCE: 09-advanced-topics/02-service-mesh-advanced.md
CONTEXT: Roll back namespace to previous Istio revision
---

### 03-network-observability.md

TOOL: hubble
CMD: cilium hubble port-forward &
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Start Hubble port-forward for CLI access
---

TOOL: hubble
CMD: hubble observe --namespace default
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Observe all network flows in a namespace
---

TOOL: hubble
CMD: hubble observe --verdict DROPPED --namespace default
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Filter to dropped packets to diagnose network policy violations
---

TOOL: hubble
CMD: hubble observe --pod frontend --protocol http --output json
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Watch HTTP traffic to a specific pod in JSON format
---

TOOL: hubble
CMD: hubble observe --protocol dns --output jsonpb | jq '.flow.l7.dns'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Monitor DNS flows and extract DNS layer data
---

TOOL: hubble
CMD: hubble observe --namespace default --output json | jq -r '[.flow.source.pod_name, .flow.destination.pod_name, .flow.verdict] | @csv' | sort | uniq -c | sort -rn
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Build traffic matrix of flow counts per source-destination pair
---

TOOL: hubble
CMD: hubble observe --verdict DROPPED --follow --output json | jq '{src: .flow.source.pod_name, dst: .flow.destination.pod_name, reason: .flow.drop_reason_desc}'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Real-time drop monitoring showing source, dest, and drop reason
---

TOOL: bpftrace
CMD: bpftrace -e 'kprobe:tcp_connect { @connections[((struct sock *)arg0)->__sk_common.skc_dport] = count(); } interval:s:1 { print(@connections); clear(@connections); }'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Count new TCP connections per second by destination port
---

TOOL: bpftrace
CMD: bpftrace -e 'kprobe:tcp_set_state { @states[arg1] = count(); } interval:s:5 { print(@states); clear(@states); }'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Trace TCP connection state changes and count by state
---

TOOL: bpftrace
CMD: bpftrace -e 'kprobe:tcp_retransmit_skb { $sk = (struct sock *)arg0; printf("retransmit: %s:%d -> %s:%d\n", ntop($sk->__sk_common.skc_rcv_saddr), $sk->__sk_common.skc_num, ntop($sk->__sk_common.skc_daddr), ntohs($sk->__sk_common.skc_dport)); }'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Monitor TCP retransmissions with source and destination addresses
---

TOOL: bpftrace
CMD: bpftrace -e 'kprobe:tcp_data_queue { @start[tid] = nsecs; } kretprobe:tcp_data_queue { @latency_us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Histogram of TCP receive latency in kernel receive queue
---

TOOL: bpftrace
CMD: bpftrace -e 'kprobe:inet_csk_accept { $sk = (struct sock *)arg0; $backlog = $sk->sk_ack_backlog; if ($backlog > 0) @backlog_hist = hist($backlog); }'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Track socket accept queue depth as histogram
---

TOOL: bpftrace
CMD: bpftrace -e 'tracepoint:skb:kfree_skb { @drops[args->reason] = count(); } interval:s:5 { print(@drops); }'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Count packet drops grouped by kernel drop reason
---

TOOL: px
CMD: px deploy
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Install Pixie eBPF observability on Kubernetes cluster
---

TOOL: px
CMD: px run px/http_data -- --start_time='-5m' | px stats http_latency
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Query HTTP latency p99 per service via Pixie
---

TOOL: px
CMD: px run px/node_stats -- --start_time='-5m'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Identify which pod is dropping packets via Pixie
---

TOOL: hubble
CMD: hubble observe --verdict DROPPED --namespace payments --follow --output json | head -20
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Check Hubble for L3/L4 drops in payments namespace
---

TOOL: kubectl
CMD: kubectl get pods -n payments -o wide
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Identify nodes running payment pods for NIC-level inspection
---

TOOL: ethtool
CMD: ethtool -S eth0 | grep -E '(drop|miss|overflow)' | grep -v ': 0'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Check NIC-level drop and miss counters (non-zero only)
---

TOOL: ethtool
CMD: ethtool -g eth0
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Check NIC ring buffer (RX/TX queue) sizes
---

TOOL: ethtool
CMD: ethtool -G eth0 rx 4096 tx 4096
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Increase NIC ring buffer size to reduce packet drops
---

TOOL: hubble
CMD: hubble observe --namespace payments --verdict DROPPED --output json | jq '{time: .time, src: .flow.source.pod_name}'
SOURCE: 09-advanced-topics/03-network-observability.md
CONTEXT: Validate no drops after ring buffer fix
---

### 04-performance-engineering-at-scale.md

TOOL: ethtool
CMD: ethtool -k eth0 | grep tcp-seg
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check TSO (TCP Segmentation Offload) status on interface
---

TOOL: ethtool
CMD: ethtool -K eth0 tso off
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Disable TSO for debugging MTU issues (restore after)
---

TOOL: ethtool
CMD: ethtool -K eth0 tso on
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Re-enable TCP Segmentation Offload after debugging
---

TOOL: ethtool
CMD: ethtool -k eth0 | grep generic-receive
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check GRO (Generic Receive Offload) status
---

TOOL: ethtool
CMD: ethtool -k eth0 | grep large-receive
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Verify LRO is disabled (LRO breaks IP forwarding)
---

TOOL: ethtool
CMD: ethtool -K eth0 lro off
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Disable Large Receive Offload if erroneously enabled
---

TOOL: ethtool
CMD: ethtool -k eth0 | grep checksum
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check hardware checksum offload status
---

TOOL: ethtool
CMD: ethtool -l eth0
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check current and maximum NIC queue count
---

TOOL: ethtool
CMD: ethtool -L eth0 combined 16
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Set NIC queue count to match CPU count for RSS
---

TOOL: ethtool
CMD: ethtool -x eth0
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: View RSS indirection table (hash-to-queue mapping)
---

TOOL: ethtool
CMD: ethtool -X eth0 equal 16
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Redistribute RSS indirection table evenly across queues
---

TOOL: sysctl
CMD: sysctl net.ipv4.ip_local_port_range
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check current ephemeral port range for outbound connections
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.ip_local_port_range="1024 65535"
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Expand ephemeral port range for more concurrent connections
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_max_tw_buckets
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check maximum TIME_WAIT socket bucket count
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_tw_reuse=1
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Allow reuse of TIME_WAIT ports for new outbound connections
---

TOOL: ss
CMD: ss -tlnp | grep :80
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Verify SO_REUSEPORT is in use (multiple sockets on same port)
---

TOOL: ss
CMD: ss -lnt | grep :80
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check accept queue depth and maximum size for port 80
---

TOOL: sysctl
CMD: sysctl net.core.somaxconn
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check system-wide maximum accept queue depth per socket
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_max_syn_backlog
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check SYN backlog (half-open connections in SYN_RECV state)
---

TOOL: tc
CMD: tc qdisc replace dev eth0 root fq
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Install fq (fair queuing) qdisc required for BBR pacing
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_congestion_control=bbr
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Set BBR as the TCP congestion control algorithm
---

TOOL: sysctl
CMD: sysctl -w net.core.somaxconn=65535
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Increase system-wide accept queue maximum for high-RPS nginx
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_max_syn_backlog=65535
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Increase SYN backlog to handle burst connections
---

TOOL: ss
CMD: ss -lnp | grep nginx
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check nginx LISTEN sockets and SO_REUSEPORT usage
---

TOOL: sysctl
CMD: sysctl -w fs.file-max=2000000
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Increase system-wide file descriptor limit
---

TOOL: nstat
CMD: nstat -z | grep ListenOverflows
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check TCP listen overflow counter (connections being dropped)
---

TOOL: nstat
CMD: nstat -z | grep -E '(Retrans|Drop|Overflow)'
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check key network counters for retransmissions, drops, overflow
---

TOOL: perf
CMD: perf top -p $(pgrep -d, nginx)
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Profile nginx CPU usage to identify next bottleneck after tuning
---

TOOL: ethtool
CMD: ethtool -K eth0 gro off tso off
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Disable GRO and TSO for accurate pps benchmarking
---

TOOL: ethtool
CMD: ethtool -K eth0 gro on tso on
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Re-enable GRO and TSO after pps benchmarking
---

TOOL: sysctl
CMD: sysctl -w net.core.netdev_budget=600
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Increase NAPI poll budget to reduce time_squeeze at high pps
---

TOOL: sysctl
CMD: sysctl -w net.core.netdev_budget_usecs=8000
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Increase NAPI poll time budget in microseconds
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_max_tw_buckets
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Check TIME_WAIT bucket limit for high-connection-rate servers
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_max_tw_buckets=4000000
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Increase TIME_WAIT bucket limit for 50K connections/sec load
---

TOOL: echo
CMD: echo "ff" > /sys/class/net/eth0/queues/rx-0/rps_cpus
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Enable RPS to distribute packet processing across all CPUs
---

TOOL: sysctl
CMD: echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Set global RFS flow table size for socket-affinity steering
---

TOOL: echo
CMD: echo 4096 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Set per-queue RFS flow count for receive flow steering
---

TOOL: echo
CMD: echo "1" > /proc/irq/32/smp_affinity_list
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Pin NIC IRQ to a specific NUMA-local CPU
---

TOOL: echo
CMD: echo "1" > /sys/class/net/eth0/queues/tx-0/xps_cpus
SOURCE: 09-advanced-topics/04-performance-engineering-at-scale.md
CONTEXT: Set XPS to map TX queue 0 to CPU 0 for cache locality
---

### 05-bgp-at-scale.md

TOOL: vtysh
CMD: vtysh -c "show bgp summary"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Check all BGP session states and prefix counts
---

TOOL: vtysh
CMD: vtysh -c "show bgp neighbors 192.168.1.1"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Show detailed BGP neighbor info including hold timers
---

TOOL: vtysh
CMD: vtysh -c "show bgp ipv4 unicast 1.1.1.0/24"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Check RPKI validation state for a specific prefix
---

TOOL: vtysh
CMD: vtysh -c "show bgp ipv4 unicast 10.0.0.0/8"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Show BGP route details including path attributes
---

TOOL: vtysh
CMD: vtysh -c "show bgp ipv4 unicast neighbors 192.168.1.1 routes"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Show all routes received from a specific BGP neighbor
---

TOOL: vtysh
CMD: vtysh -c "show bgp ipv4 unicast neighbors 192.168.1.1 advertised-routes"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Show all routes advertised to a specific BGP neighbor
---

TOOL: vtysh
CMD: vtysh -c "show bgp ipv4 unicast" | head -100
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: View BGP RIB including all paths, not just best path
---

TOOL: vtysh
CMD: vtysh -c "debug bgp updates"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Enable BGP UPDATE debug logging (use on low-traffic sessions)
---

TOOL: vtysh
CMD: vtysh -c "debug bgp neighbor-events"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Enable BGP neighbor event debug logging
---

TOOL: vtysh
CMD: vtysh -c "show rpki prefix-table"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Show RPKI validated prefix table on FRRouting router
---

TOOL: vtysh
CMD: vtysh -c "show rpki cache-connection"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Check RPKI validator cache connection status
---

TOOL: vtysh
CMD: vtysh -c "show bfd peers"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Show BFD session state, diagnostics, and intervals
---

TOOL: ip
CMD: ip route show 10.0.0.0/8
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Verify ECMP paths are installed in the routing table
---

TOOL: curl
CMD: curl -s "https://stat.ripe.net/data/looking-glass/data.json?resource=203.0.113.0/24" | jq '.data.rrcs[].peers[].as_path'
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Check BGP AS paths for a prefix from RIPE RIS vantage points
---

TOOL: vtysh
CMD: vtysh -c "show bgp ipv4 unicast 203.0.113.0/24"
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Check if leaked route is being accepted on own routers
---

TOOL: vtysh
CMD: vtysh << EOF
conf t
router bgp 64512
  address-family ipv4 unicast
    network 203.0.113.0/25
    network 203.0.113.128/25
EOF
SOURCE: 09-advanced-topics/05-bgp-at-scale.md
CONTEXT: Announce more-specific prefixes to win over leaked routes
---

### 06-ipv6-production.md

TOOL: aws
CMD: aws ec2 create-vpc --cidr-block 10.0.0.0/16 --amazon-provided-ipv6-cidr-block
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Create a dual-stack VPC with AWS-provided IPv6 /56 prefix
---

TOOL: aws
CMD: aws ec2 create-subnet --vpc-id vpc-xxxx --cidr-block 10.0.1.0/24 --ipv6-cidr-block 2600:1f18:xxxx:xx01::/64
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Create dual-stack subnet with both IPv4 and IPv6 CIDRs
---

TOOL: aws
CMD: aws ec2 run-instances --image-id ami-xxxx --instance-type t3.medium --ipv6-address-count 1
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Launch EC2 instance with both IPv4 and IPv6 addresses
---

TOOL: aws
CMD: aws eks create-cluster --name my-cluster --kubernetes-network-config ipFamily=ipv6
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Create EKS cluster with IPv6-only pod networking
---

TOOL: kubectl
CMD: kubectl get pods -o wide
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Verify pod IP addresses are IPv6 on IPv6-only EKS cluster
---

TOOL: ip
CMD: ip -6 addr show eth0
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Check IPv6 addresses and SLAAC configuration on interface
---

TOOL: ip
CMD: ip -6 route show
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Check IPv6 routing table including default route from RA
---

TOOL: ip
CMD: ip tunnel add sit1 mode sit remote 192.0.2.1 local 203.0.113.1
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Create a 6in4 tunnel for IPv6-over-IPv4 transport
---

TOOL: ip
CMD: ip link set sit1 up
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Bring up 6in4 tunnel interface
---

TOOL: ip
CMD: ip -6 addr add 2001:db8::1/64 dev sit1
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Assign IPv6 address to 6in4 tunnel interface
---

TOOL: ip
CMD: ip -6 route add ::/0 via ::192.0.2.1 dev sit1
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Add default IPv6 route via 6in4 tunnel
---

TOOL: sysctl
CMD: sysctl -w net.ipv6.neigh.eth0.gc_thresh1=1024
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Set IPv6 neighbor cache GC lower threshold
---

TOOL: sysctl
CMD: sysctl -w net.ipv6.neigh.eth0.gc_thresh2=2048
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Set IPv6 neighbor cache GC trigger threshold
---

TOOL: sysctl
CMD: sysctl -w net.ipv6.neigh.eth0.gc_thresh3=4096
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Set IPv6 neighbor cache hard limit (neighbor exhaustion defense)
---

TOOL: sysctl
CMD: sysctl -w net.ipv6.neigh.eth0.unres_qlen=3
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Limit queued packets per unresolved neighbor to reduce memory
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -p icmpv6 --icmpv6-type destination-unreachable -j ACCEPT
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Allow ICMPv6 Destination Unreachable (required for IPv6)
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Allow ICMPv6 Packet Too Big (required for IPv6 PMTUD)
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -p icmpv6 --icmpv6-type time-exceeded -j ACCEPT
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Allow ICMPv6 Time Exceeded messages
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -p icmpv6 --icmpv6-type parameter-problem -j ACCEPT
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Allow ICMPv6 Parameter Problem messages
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -p icmpv6 --icmpv6-type router-advertisement -j ACCEPT
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Allow Router Advertisement for SLAAC and default gateway
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-solicitation -j ACCEPT
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Allow Neighbor Solicitation (IPv6 ARP equivalent)
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -p icmpv6 --icmpv6-type neighbor-advertisement -j ACCEPT
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Allow Neighbor Advertisement (IPv6 ARP reply equivalent)
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -m ipv6header --header rh -j DROP
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Drop routing extension headers (deprecated RH0 attack surface)
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -m frag --fragfirst -j DROP
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Drop fragmented first packets that evade port-based rules
---

TOOL: nft
CMD: nft add table inet filter
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Create unified nftables table for both IPv4 and IPv6
---

TOOL: nft
CMD: nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Add default-drop input chain to nftables inet table
---

TOOL: nft
CMD: nft add rule inet filter input tcp dport 22 accept
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Allow SSH for both IPv4 and IPv6 in single nftables rule
---

TOOL: dig
CMD: dig AAAA ipv4only.arpa @<coredns-ip>
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Verify DNS64 is synthesizing AAAA records from A records
---

TOOL: kubectl
CMD: kubectl edit configmap coredns -n kube-system
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Edit CoreDNS configmap to add DNS64 plugin configuration
---

TOOL: kubectl
CMD: kubectl exec -it test-pod -- nslookup postgres.internal
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Test DNS64 synthesis returns IPv6 address for IPv4-only host
---

TOOL: kubectl
CMD: kubectl exec -it test-pod -- nc -z 64:ff9b::a32:64 5432
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Test TCP connectivity through NAT64 gateway to IPv4 host
---

TOOL: ip
CMD: ip -6 neigh show
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Show IPv6 neighbor cache (equivalent of ARP cache)
---

TOOL: traceroute6
CMD: traceroute6 2001:4860:4860::8888
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Trace IPv6 path to Google's IPv6 DNS resolver
---

TOOL: ping6
CMD: ping6 -M do -s 1452 2001:db8::1
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Test IPv6 PMTUD with large packets (1452 + 40 byte header)
---

TOOL: nstat
CMD: nstat -z | grep Icmp6
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Check ICMPv6 statistics counters
---

TOOL: dig
CMD: dig AAAA ipv4-only-host.internal @<coredns-ip>
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Test DNS64 synthesis for a specific internal hostname
---

TOOL: ip6tables
CMD: ip6tables -L -n -v
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: List all IPv6 firewall rules with verbose output
---

TOOL: tcpdump
CMD: tcpdump -ni eth0 icmp6
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Monitor all NDP/ICMPv6 traffic on interface
---

TOOL: tcpdump
CMD: tcpdump -ni eth0 'icmp6 and (ip6[40] == 134)'
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Capture only Router Advertisement ICMPv6 messages
---

TOOL: sysctl
CMD: sysctl net.ipv6.conf.all.forwarding
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Check if IPv6 forwarding is enabled (required for routers)
---

TOOL: ip
CMD: ip -6 neigh show | wc -l
SOURCE: 09-advanced-topics/06-ipv6-production.md
CONTEXT: Count IPv6 neighbor cache entries (detect exhaustion)
---
