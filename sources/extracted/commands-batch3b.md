# Commands Batch 3B — Extracted from 10-interview-prep/ and 11-hands-on-labs/

<!-- Directory: 10-interview-prep/ -->

---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_keepalive_time
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Show keepalive idle time before first probe (default 7200s)
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_keepalive_intvl
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Show interval between keepalive probes (default 75s)
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_keepalive_probes
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Show number of keepalive probes before RST (default 9)
---

TOOL: ss
CMD: ss -tanp state close-wait
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: List all CLOSE_WAIT sockets with process info
---

TOOL: ss
CMD: ss -s | grep "close wait"
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Count CLOSE_WAIT sockets on the system
---

TOOL: sysctl
CMD: sysctl net.ipv4.ip_local_port_range
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Show ephemeral port range (default 32768-60999)
---

TOOL: ss
CMD: ss -tn | awk '{print $4}' | awk -F: '{print $NF}' | sort -n | uniq -c | sort -rn | head
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Show ephemeral port usage distribution sorted by count
---

TOOL: nstat
CMD: nstat -a | grep -E "TcpExt(TWRecycled|PAWSPassive|SynRetrans)"
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Check TIME_WAIT recycling and PAWS failure counters
---

TOOL: ss
CMD: ss -tn dst <specific-ip> | wc -l
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Count connections to a specific destination IP
---

TOOL: ss
CMD: ss -s | grep "time wait"
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Count TIME_WAIT connections on the system
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.ip_local_port_range="1024 65535"
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Expand ephemeral port range to gain ~32K additional ports
---

TOOL: iptables
CMD: iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 10.0.1.1-10.0.1.10
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: SNAT outbound traffic across 10 IPs to expand port capacity
---

TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Reduce TIME_WAIT conntrack timeout from 120s to 30s
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_tw_reuse=1
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Allow reuse of TIME_WAIT sockets for outbound connections
---

TOOL: dig
CMD: dig +trace api.example.com
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Trace full DNS resolution chain to authoritative server
---

TOOL: dig
CMD: dig SOA example.com
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Check negative cache TTL (minimum field in SOA record)
---

TOOL: dig
CMD: dig api.example.com
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Basic DNS query; returns SERVFAIL if resolver error
---

TOOL: dig
CMD: dig +cd api.example.com
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Query with DNSSEC checking disabled to isolate DNSSEC failure
---

TOOL: dig
CMD: dig +dnssec +trace api.example.com
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Trace DNS resolution with DNSSEC to find broken chain
---

TOOL: nc
CMD: nc -u -z <resolver-ip> 53
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Test UDP connectivity to DNS resolver on port 53
---

TOOL: dig
CMD: dig +tcp api.example.com @<resolver-ip>
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Force TCP for DNS query to test TCP fallback
---

TOOL: dig
CMD: dig api.example.com @8.8.8.8
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Query using alternative resolver to isolate resolver problem
---

TOOL: conntrack
CMD: conntrack -S | grep insert_failed
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Check for conntrack UDP race condition causing DNS drops
---

TOOL: ping
CMD: ping -M do -s 1472 <server-ip>
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Test PMTUD with DF bit set at 1500-byte total packet size
---

TOOL: ping
CMD: ping -M do -s 1372 <server-ip>
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Test PMTUD at 1400-byte total; compare to find path MTU
---

TOOL: ip
CMD: ip route get <client-ip>
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Check PMTU cache for a specific client IP
---

TOOL: tcpdump
CMD: tcpdump -ni eth0 'icmp and icmp[icmptype] == 3 and icmp[icmpcode] == 4'
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Capture ICMP Fragmentation Needed messages for PMTUD debugging
---

TOOL: nstat
CMD: nstat -a | grep -i "MTUPFail"
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Check PLPMTUD failure counters (kernel 5.1+)
---

TOOL: iptables
CMD: iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Enable MSS clamping to path MTU on forwarded connections
---

TOOL: iptables
CMD: iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1360
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Set fixed MSS to 1360 bytes for 1400-byte path MTU
---

TOOL: iptables
CMD: iptables -A INPUT -p icmp --icmp-type fragmentation-needed -j ACCEPT
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Allow ICMP Fragmentation Needed inbound for PMTUD to work
---

TOOL: iptables
CMD: iptables -A FORWARD -p icmp --icmp-type fragmentation-needed -j ACCEPT
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Allow ICMP Fragmentation Needed on forwarded traffic
---

TOOL: ip6tables
CMD: ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Allow ICMPv6 Packet Too Big for IPv6 PMTUD
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_mtu_probing=1
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Enable PLPMTUD to probe MTU without ICMP (kernel 5.1+)
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_base_mss=1024
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Set starting probe size for PLPMTUD
---

TOOL: arping
CMD: arping -A -I eth0 10.0.1.100
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Send gratuitous ARP reply to update neighbor caches after failover
---

TOOL: arping
CMD: arping -U -I eth0 10.0.1.100
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Send gratuitous ARP request for broader compatibility
---

TOOL: tcpdump
CMD: tcpdump -i eth0 arp and arp[6:2] = 2
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Capture ARP Reply packets to verify gratuitous ARP receipt
---

TOOL: ip
CMD: ip neigh show | grep <VIP>
SOURCE: 10-interview-prep/01-networking-fundamentals-qa.md
CONTEXT: Check ARP cache entry for a VIP address
---

<!-- 02-linux-networking-qa.md -->

TOOL: sysctl
CMD: sysctl net.netfilter.nf_conntrack_count
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Show current number of conntrack table entries
---

TOOL: sysctl
CMD: sysctl net.netfilter.nf_conntrack_max
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Show maximum allowed conntrack table entries
---

TOOL: conntrack
CMD: conntrack -S
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Show per-CPU conntrack stats including drops and insert_failed
---

TOOL: conntrack
CMD: conntrack -L -o extended | awk '{print $4}' | sort | uniq -c | sort -rn
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Break down conntrack entries by TCP state
---

TOOL: conntrack
CMD: conntrack -L -o extended | awk '{print $3}' | sort | uniq -c | sort -rn
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Break down conntrack entries by protocol
---

TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Reduce TIME_WAIT conntrack entry timeout from 120s default
---

TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_udp_timeout=10
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Reduce UDP conntrack timeout to 10s (helps DNS entry churn)
---

TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_udp_timeout_stream=60
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Set streaming UDP conntrack timeout
---

TOOL: iptables
CMD: iptables -t raw -A PREROUTING -p tcp --dport 8080 -j NOTRACK
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Exempt health check endpoint from conntrack to save table space
---

TOOL: iptables
CMD: iptables -t raw -A OUTPUT -p tcp --sport 8080 -j NOTRACK
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Exempt health check responses from conntrack tracking
---

TOOL: nft
CMD: nft add rule ip raw prerouting tcp dport 8080 notrack
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: nftables equivalent of iptables NOTRACK for port 8080
---

TOOL: ss
CMD: ss -tanp state close-wait | awk '{print $NF}' | sort | uniq -c | sort -rn
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Count CLOSE_WAIT sockets grouped by process
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_keepalive_time=60
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Set aggressive keepalive to probe after 60s idle (vs 7200 default)
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_keepalive_intvl=10
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Set keepalive probe interval to 10 seconds
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_keepalive_probes=6
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Set number of keepalive probes to 6 before RST
---

TOOL: ping
CMD: ping -M do -s 1452 <remote-ip>
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Test path MTU at 1480-byte total (1452 + 28 header)
---

TOOL: tcpdump
CMD: tcpdump -ni eth0 'icmp and icmp[icmptype] == 3 and icmp[icmpcode] == 4'
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Capture ICMP Fragmentation Needed at server for PMTUD debug
---

TOOL: nstat
CMD: nstat -a | grep MTUPFail
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Check PLPMTUD failure counters
---

TOOL: iptables
CMD: iptables -Z INPUT
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Zero iptables INPUT chain counters before traffic test
---

TOOL: iptables
CMD: iptables -L INPUT -n -v --line-numbers | awk '$1 > 0 {print}'
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Show only iptables INPUT rules that matched traffic
---

TOOL: iptables
CMD: iptables -L INPUT -n -v --line-numbers | grep 8443
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Find existing iptables rules matching port 8443
---

TOOL: iptables
CMD: iptables -I INPUT <line-before-drop> -p tcp --dport 8443 -s <trusted-cidr> -j ACCEPT
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Insert ACCEPT rule before conflicting DROP at specific line
---

TOOL: ipset
CMD: ipset create trusted-sources hash:ip
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Create ipset for O(1) lookup of trusted source IPs
---

TOOL: ipset
CMD: ipset add trusted-sources 10.0.1.0/24
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Add subnet to ipset for efficient firewall matching
---

TOOL: iptables
CMD: iptables -I INPUT -m set --match-set trusted-sources src -p tcp --dport 8443 -j ACCEPT
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Use ipset for O(1) firewall lookup instead of O(n) rules
---

TOOL: nft
CMD: nft add table inet filter
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Create nftables inet filter table
---

TOOL: nft
CMD: nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Create nftables input chain with default drop policy
---

TOOL: nft
CMD: nft add rule inet filter input tcp dport { 443, 8443 } accept
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Add nftables rule accepting ports 443 and 8443 using sets
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_syncookies=1
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Enable SYN cookies for SYN flood defense (default on Linux)
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_max_syn_backlog
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Show SYN backlog queue size (syncookies triggers when full)
---

TOOL: ip
CMD: ip link set eth0 xdp obj syn_flood_xdp.o sec xdp
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Load XDP SYN flood protection program onto interface
---

TOOL: bpftool
CMD: bpftool map dump name syn_stats
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Read XDP SYN flood drop statistics from BPF map
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_max_syn_backlog=65536
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Increase SYN backlog queue size to 65536
---

TOOL: sysctl
CMD: sysctl -w net.core.somaxconn=65536
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Increase maximum socket listen backlog
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_syn_retries=2
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Reduce SYN retransmissions to attacker to minimize state retention
---

TOOL: ss
CMD: ss -s | grep "time wait"
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Count TIME_WAIT sockets
---

TOOL: ss
CMD: ss -tn state time-wait | head
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: List TIME_WAIT connections with destinations
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_fin_timeout
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Show FIN_WAIT_2 timeout (not actual TIME_WAIT duration)
---

TOOL: ethtool
CMD: ethtool -G eth0 rx 4096
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Set NIC receive ring buffer size to 4096 entries
---

TOOL: sysctl
CMD: sysctl net.core.netdev_budget=600
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Show/set NAPI budget packets per softirq run
---

TOOL: ethtool
CMD: ethtool -K eth0 gro on
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Enable Generic Receive Offload to reduce stack traversals
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_rmem="4096 87380 4194304"
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Set TCP receive buffer min/default/max sizes
---

TOOL: ethtool
CMD: ethtool -S eth0 | grep rx_dropped
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Check NIC receive drop counter (ring buffer overflow)
---

TOOL: nstat
CMD: nstat -a | grep TcpExtRcvPruned
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Check socket buffer pruning counter (socket too small)
---

TOOL: iptables
CMD: iptables -L -t nat -n | grep KUBE-
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: List kube-proxy NAT rules in iptables
---

TOOL: iptables
CMD: iptables -L KUBE-SERVICES -n | wc -l
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: Count total kube-proxy service rules in iptables
---

TOOL: bpftool
CMD: bpftool prog show
SOURCE: 10-interview-prep/02-linux-networking-qa.md
CONTEXT: List all loaded eBPF programs with IDs and types
---

<!-- 03-cloud-networking-qa.md -->

TOOL: aws
CMD: aws ec2 create-route --route-table-id rtb-XXXXXXXX --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-XXXXXXXX
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Add route to VPC-B's CIDR via peering connection in VPC-A
---

TOOL: aws
CMD: aws ec2 create-route --route-table-id rtb-YYYYYYYY --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id pcx-XXXXXXXX
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Add return route to VPC-A's CIDR in VPC-B for bidirectional peering
---

TOOL: kubectl
CMD: kubectl describe node <node-name> | grep -A 10 "Allocatable"
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Show allocatable pod IP capacity on an EKS node
---

TOOL: kubectl
CMD: kubectl logs -n kube-system aws-node-<pod-id> | grep -i "exhausted\|failed\|no.*ip"
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Check AWS VPC CNI logs for IP exhaustion errors
---

TOOL: aws
CMD: aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[].{Available:AvailableIpAddressCount, CIDR:CidrBlock}'
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Check available IP count in an EKS subnet
---

TOOL: kubectl
CMD: kubectl get pods -A -o wide | grep <node-name> | wc -l
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Count pods scheduled on a specific node
---

TOOL: aws
CMD: aws ec2 describe-network-interfaces --filters "Name=attachment.instance-id,Values=<instance-id>" --query 'NetworkInterfaces[].PrivateIpAddresses[].PrivateIpAddress'
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: List all private IPs allocated to an EC2 instance's ENIs
---

TOOL: kubectl
CMD: kubectl describe daemonset -n kube-system aws-node | grep -A 5 "WARM_IP"
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Show current WARM_IP_TARGET setting in aws-node DaemonSet
---

TOOL: kubectl
CMD: kubectl edit daemonset -n kube-system aws-node
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Edit aws-node DaemonSet to tune IP warm pool settings
---

TOOL: aws
CMD: aws ec2 associate-vpc-cidr-block --vpc-id vpc-XXXXXXXX --cidr-block 100.64.0.0/16
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Add secondary CIDR block to VPC to expand IP space
---

TOOL: aws
CMD: aws ec2 create-subnet --vpc-id vpc-XXXXXXXX --cidr-block 100.64.0.0/18 --availability-zone us-east-1a
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Create subnet from secondary CIDR for pod IP expansion
---

TOOL: kubectl
CMD: kubectl set env daemonset -n kube-system aws-node ENABLE_PREFIX_DELEGATION=true
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Enable prefix delegation in AWS VPC CNI for IP efficiency
---

TOOL: kubectl
CMD: kubectl set env daemonset -n kube-system aws-node AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Enable custom networking mode in AWS VPC CNI
---

TOOL: aws
CMD: aws ec2 create-flow-logs --resource-type Subnet --resource-ids subnet-XXXXXXXX --traffic-type REJECT --log-destination-type s3 --log-destination arn:aws:s3:::my-vpc-flow-logs
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Enable VPC Flow Logs for rejected traffic on a subnet
---

TOOL: aws
CMD: aws cloudwatch get-metric-statistics --namespace AWS/NATGateway --metric-name ErrorPortAllocation --dimensions Name=NatGatewayId,Value=nat-XXXXXXXX --start-time 2026-01-01T00:00:00Z --end-time 2026-01-02T00:00:00Z --period 60 --statistics Sum
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Check NAT Gateway port allocation error metric
---

TOOL: aws
CMD: aws ec2 create-vpc-endpoint --vpc-id vpc-XXX --service-name com.amazonaws.us-east-1.s3 --route-table-ids rtb-XXX
SOURCE: 10-interview-prep/03-cloud-networking-qa.md
CONTEXT: Create S3 VPC Gateway Endpoint to bypass NAT Gateway
---

<!-- 04-kubernetes-networking-qa.md -->

TOOL: ip
CMD: ip route show | grep 10.244.2.0
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Check if BGP route to remote pod CIDR exists on node
---

TOOL: sysctl
CMD: sysctl net.ipv4.ip_forward
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Verify IP forwarding is enabled on Kubernetes nodes
---

TOOL: iptables
CMD: iptables -L FORWARD -n -v | grep DROP
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Check for DROP rules in FORWARD chain blocking pod traffic
---

TOOL: kubectl
CMD: kubectl get networkpolicies -n <namespace> -o yaml
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: List all NetworkPolicies in a namespace with full spec
---

TOOL: kubectl
CMD: kubectl get pod <pod> -n <namespace> --show-labels
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Show pod labels to verify NetworkPolicy selector match
---

TOOL: kubectl
CMD: kubectl exec -n <ns> <blocked-pod> -- curl -v --connect-timeout 5 http://<target-pod-ip>:8080
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Test outbound connectivity from blocked pod
---

TOOL: iptables
CMD: iptables -L cali-fw-<pod-interface> -n -v
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Inspect Calico iptables rules for a specific pod interface
---

TOOL: kubectl
CMD: kubectl exec -n kube-system cilium-<hash> -- cilium endpoint list
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: List Cilium endpoints with their IDs and policy status
---

TOOL: kubectl
CMD: kubectl exec -n kube-system cilium-<hash> -- cilium monitor --from <endpoint-id> --type drop
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Monitor live traffic drops from a specific Cilium endpoint
---

TOOL: kubectl
CMD: kubectl exec -n kube-system cilium-<hash> -- cilium policy get
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Show Cilium policy verdicts and rules
---

TOOL: kubectl
CMD: kubectl exec <pod> -- nslookup kubernetes.default
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Test DNS resolution from a pod to verify egress port 53 is allowed
---

TOOL: kubectl
CMD: kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Deploy NodeLocal DNSCache DaemonSet to reduce CoreDNS load
---

TOOL: conntrack
CMD: conntrack -S | grep insert_failed
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Check if conntrack UDP race condition is causing DNS drops
---

TOOL: istioctl
CMD: istioctl authn tls-check <pod> <service>
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Verify mTLS status between a pod and service in Istio
---

TOOL: kubectl
CMD: kubectl exec -n production <pod> -c istio-proxy -- openssl x509 -in /etc/certs/cert-chain.pem -noout -dates -subject
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Check Istio sidecar certificate validity dates and subject
---

TOOL: kubectl
CMD: kubectl exec -it some-pod -- nslookup postgres-0.postgres.default.svc.cluster.local
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: Verify StatefulSet pod DNS name resolves correctly
---

TOOL: kubectl
CMD: kubectl exec -it some-pod -- dig postgres-0.postgres.default.svc.cluster.local
SOURCE: 10-interview-prep/04-kubernetes-networking-qa.md
CONTEXT: DNS lookup for individual StatefulSet pod by stable name
---

<!-- 05-network-security-qa.md -->

TOOL: openssl
CMD: openssl s_client -connect api.example.com:443 -tls1_3
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Test TLS 1.3 connection and verify cipher suite
---

TOOL: openssl
CMD: openssl s_client -connect api.example.com:443 -tls1_2
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Test TLS 1.2 fallback (verify server has disabled it)
---

TOOL: openssl
CMD: openssl s_client -connect api.example.com:443 -showcerts 2>/dev/null | openssl x509 -noout -dates
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Check certificate expiry dates from live TLS connection
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_syncookies=2
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Always use SYN cookies (kernel 4.4+), not just when backlog full
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_syn_retries=2
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Reduce SYN-ACK retransmissions to reduce state per spoofed SYN
---

TOOL: curl
CMD: curl "https://crt.sh/?q=%.yourdomain.com&output=json" | jq '.[].name_value'
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Monitor Certificate Transparency logs for unauthorized certs
---

TOOL: kubectl
CMD: kubectl get certificate -A
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: List all cert-manager certificates across all namespaces
---

TOOL: kubectl
CMD: kubectl describe certificate <name>
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Show cert-manager certificate status and renewal details
---

TOOL: openssl
CMD: echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null | openssl x509 -noout -enddate
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Check certificate expiry date from external probe
---

TOOL: openssl
CMD: openssl x509 -noout -modulus -in cert.pem | openssl md5
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Get certificate modulus hash to verify cert/key match
---

TOOL: openssl
CMD: openssl rsa -noout -modulus -in key.pem | openssl md5
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Get private key modulus hash to verify cert/key match
---

TOOL: aws
CMD: aws ec2 start-network-insights-analysis --network-insights-path-id nip-XXXXXXXX
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: Run AWS Network Reachability Analyzer for PCI compliance audit
---

TOOL: aws
CMD: aws ec2 describe-security-groups --filters "Name=tag:Environment,Values=PCI" --query 'SecurityGroups[].IpPermissions'
SOURCE: 10-interview-prep/05-network-security-qa.md
CONTEXT: List inbound rules for PCI-tagged security groups
---

<!-- 06-system-design-qa.md (minimal CLI content) -->

TOOL: helm
CMD: helm install cilium cilium/cilium --version 1.15.x --set eni.enabled=true --set ipam.mode=eni --set egressMasqueradeInterfaces=eth0 --set tunnel=disabled --set kubeProxyReplacement=true
SOURCE: 10-interview-prep/06-system-design-qa.md
CONTEXT: Install Cilium with ENI mode and kube-proxy replacement enabled
---

TOOL: cilium
CMD: cilium hubble enable
SOURCE: 10-interview-prep/06-system-design-qa.md
CONTEXT: Enable Hubble observability in a Cilium installation
---

TOOL: cilium
CMD: cilium hubble ui
SOURCE: 10-interview-prep/06-system-design-qa.md
CONTEXT: Launch Hubble UI for real-time Kubernetes flow visualization
---

TOOL: aws
CMD: aws ec2 create-vpc --cidr-block 10.1.0.0/16
SOURCE: 10-interview-prep/06-system-design-qa.md
CONTEXT: Create a new VPC for service tier segmentation
---

TOOL: aws
CMD: aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id tgw-XXXXXXXX --vpc-id vpc-payment --subnet-ids subnet-XXXXXXXX
SOURCE: 10-interview-prep/06-system-design-qa.md
CONTEXT: Attach a VPC to Transit Gateway for hub-and-spoke connectivity
---

<!-- 07-cross-domain-scenarios.md -->

TOOL: ip
CMD: ssh node-a ip route show | grep 10.244.2.0
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check if BGP route to remote pod CIDR exists on node
---

TOOL: aws
CMD: aws ec2 describe-instance-attribute --instance-id <id> --attribute sourceDestCheck
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Verify source/dest check is disabled on EC2 nodes for Calico
---

TOOL: iptables
CMD: iptables -L cali-FORWARD -n -v
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Inspect Calico-specific FORWARD chain rules
---

TOOL: conntrack
CMD: conntrack -S | grep insert_failed
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check Docker DNS conntrack UDP race condition
---

TOOL: iptables
CMD: iptables -t nat -L DOCKER_OUTPUT -n -v
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check Docker NAT output rules for DNS proxy
---

TOOL: iptables
CMD: iptables -t nat -L DOCKER_POSTROUTING -n -v
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check Docker NAT postrouting rules
---

TOOL: aws
CMD: aws elbv2 create-load-balancer --name payment-nlb --type network --subnets subnet-XXXXXXXX
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Create NLB in provider VPC for PrivateLink endpoint service
---

TOOL: aws
CMD: aws ec2 create-vpc-endpoint-service-configuration --network-load-balancer-arns arn:aws:elasticloadbalancing:...
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Create VPC Endpoint Service (PrivateLink) backed by NLB
---

TOOL: aws
CMD: aws ec2 create-vpc-endpoint --service-name com.amazonaws.vpce.<region>.<svc-id> --vpc-endpoint-type Interface --vpc-id vpc-XXXXXXXX
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Create Interface VPC Endpoint (PrivateLink consumer side)
---

TOOL: kubectl
CMD: kubectl exec -n production <pod> -- curl -v --connect-timeout 5 http://payment-service:8080/health
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Test pod-to-service connectivity and characterize the failure type
---

TOOL: kubectl
CMD: kubectl get svc payment-service -n production
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check if a Kubernetes service exists and its ClusterIP
---

TOOL: kubectl
CMD: kubectl get endpoints payment-service -n production
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Verify service has healthy endpoints (pods matching selector)
---

TOOL: kubectl
CMD: kubectl exec -n production <pod> -- nc -zv payment-service 8080
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Test TCP connectivity to a Kubernetes service
---

TOOL: kubectl
CMD: kubectl get networkpolicies -n production
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: List NetworkPolicies that could be blocking pod connectivity
---

TOOL: iptables
CMD: iptables -L -t nat | grep payment-service
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Find kube-proxy DNAT rule for a specific service
---

TOOL: kubectl
CMD: kubectl get pods -n production -l app=payment-service -o wide
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Get pod IPs for direct pod-to-pod connectivity test
---

TOOL: kubectl
CMD: kubectl exec -n <ns> <pod> -- ip route show
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Verify pod has default route and network connectivity
---

TOOL: kubectl
CMD: kubectl exec -n <ns> <pod> -- nslookup <rds-endpoint>
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Test DNS resolution of RDS endpoint from pod
---

TOOL: aws
CMD: aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<pod-subnet-id>"
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check VPC route table for pod's subnet to verify routing
---

TOOL: aws
CMD: aws ec2 describe-security-groups --group-ids <pod-sg-id> --query 'SecurityGroups[].IpPermissionsEgress'
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check pod security group for outbound RDS port allowance
---

TOOL: aws
CMD: aws ec2 describe-security-groups --group-ids <rds-sg-id> --query 'SecurityGroups[].IpPermissions'
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check RDS security group for inbound access from pod SG
---

TOOL: aws
CMD: aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=<pod-subnet-id>"
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check NACL rules for pod subnet to find stateless blocks
---

TOOL: nc
CMD: kubectl exec -n <ns> <pod> -- nc -zv <rds-endpoint> 5432
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Test raw TCP connection to RDS bypassing TLS and application
---

TOOL: openssl
CMD: kubectl exec -n <ns> <pod> -- openssl s_client -connect <rds-endpoint>:5432 -starttls postgres
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Test TLS connection to RDS using PostgreSQL STARTTLS
---

TOOL: kubectl
CMD: kubectl top pods -n kube-system -l k8s-app=kube-dns
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check CoreDNS CPU usage to identify DNS bottleneck
---

TOOL: kubectl
CMD: kubectl logs -n kube-system -l k8s-app=kube-dns | grep -i "error\|timeout"
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check CoreDNS logs for resolution errors or timeouts
---

TOOL: dig
CMD: dig @<coredns-ip> external.domain.com
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Query CoreDNS directly to measure its response time
---

TOOL: dig
CMD: dig @<upstream-resolver-ip> external.domain.com
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Query upstream resolver directly to compare with CoreDNS latency
---

TOOL: istioctl
CMD: istioctl proxy-status
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check Envoy proxy sync status with istiod
---

TOOL: kubectl
CMD: kubectl exec -n <ns> <pod> -c istio-proxy -- curl localhost:15000/stats | grep -E "ssl.handshake_error|ssl.connection_error"
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check Envoy TLS handshake error counts after cert rotation
---

TOOL: kubectl
CMD: kubectl rollout restart deployment istiod -n istio-system
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Force istiod restart to re-push all certificates to Envoy proxies
---

TOOL: arp
CMD: arp -n | grep <metallb-ip>
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Check ARP table for MetalLB VIP to verify GARP was received
---

TOOL: tcpdump
CMD: tcpdump -ni <uplink-interface> arp | grep <metallb-ip>
SOURCE: 10-interview-prep/07-cross-domain-scenarios.md
CONTEXT: Capture ARP announcements for MetalLB VIP
---

<!-- 08-interview-patterns-pitfalls.md -->

TOOL: ethtool
CMD: ethtool -S eth0 | grep -i error
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Check NIC errors and CRC failures at Layer 1/2
---

TOOL: ip
CMD: ip link show
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Check interface link state (UP/DOWN)
---

TOOL: ip
CMD: ip route get <destination>
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Show which route kernel selects for a specific destination
---

TOOL: ping
CMD: ping <gateway>
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Test basic Layer 3 reachability to default gateway
---

TOOL: traceroute
CMD: traceroute <destination>
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Trace packet path to find where routing diverges
---

TOOL: ss
CMD: ss -tn state established
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: List established TCP connections to verify handshake completing
---

TOOL: nstat
CMD: nstat -a | grep Retrans
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Check TCP retransmission counters for packet loss
---

TOOL: tcpdump
CMD: tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Capture SYN packets to verify connections are arriving
---

TOOL: curl
CMD: curl -v https://host/health
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Test HTTPS endpoint with verbose output for Layer 7 diagnosis
---

TOOL: openssl
CMD: openssl s_client -connect host:443
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Test TLS handshake and certificate chain
---

TOOL: dig
CMD: dig +trace hostname
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Full DNS trace to verify name resolves correctly
---

TOOL: ss
CMD: ss -tlnp | grep 8080
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Verify service is listening on expected port and interface
---

TOOL: curl
CMD: curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Test local connectivity and get HTTP status code
---

TOOL: nft
CMD: nft list ruleset | grep 8080
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Check nftables rules for port 8080 blocking
---

TOOL: nc
CMD: nc -zv <server-ip> 8080
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Test TCP connectivity from client to server
---

TOOL: dig
CMD: dig <hostname> +short
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Quick DNS lookup to verify hostname resolves to expected IP
---

TOOL: ss
CMD: ss -ltn | grep 8080
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Check accept queue depth (Recv-Q > 0 = accept queue overflow)
---

TOOL: aws
CMD: aws ec2 describe-security-groups --group-ids <sg-id>
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Check cloud security group inbound rules for a service
---

TOOL: kubectl
CMD: kubectl exec -n production <pod> -- curl -v --connect-timeout 5 http://payment-service:8080/health
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Characterize Kubernetes service connectivity failure type
---

TOOL: kubectl
CMD: kubectl exec -n production <pod> -- nslookup payment-service
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Test DNS resolution of a Kubernetes service name
---

TOOL: kubectl
CMD: kubectl get svc payment-service -n production -o yaml | grep selector
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Verify service selector labels match pod labels
---

TOOL: kubectl
CMD: kubectl get pods -n production -l app=payment-service
SOURCE: 10-interview-prep/08-interview-patterns-pitfalls.md
CONTEXT: Find pods matching a service selector
---

<!-- 09-exotic-deep-dive-qa.md -->

TOOL: ethtool
CMD: ethtool -S eth0 | grep rx_dropped
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Check ring buffer overflow drops at NIC level
---

TOOL: sysctl
CMD: sysctl -w net.core.netdev_budget=600
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Increase NAPI budget to process more packets per softirq run
---

TOOL: sysctl
CMD: sysctl -w net.core.netdev_budget_usecs=8000
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Set maximum time per softirq NAPI run in microseconds
---

TOOL: sysctl
CMD: sysctl net.core.busy_poll=50
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Enable busy polling for 50us before sleeping (trades CPU for latency)
---

TOOL: sysctl
CMD: sysctl net.core.busy_read=50
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Enable busy read polling on receive path
---

TOOL: ethtool
CMD: ethtool -c eth0
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Show current interrupt coalescing settings (rx-usecs, rx-frames)
---

TOOL: ethtool
CMD: ethtool -C eth0 rx-usecs 100 rx-frames 128
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Set interrupt coalescing for throughput (adds up to 100us latency)
---

TOOL: ethtool
CMD: ethtool -C eth0 adaptive-rx on
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Enable adaptive interrupt coalescing (NIC auto-tunes)
---

TOOL: ethtool
CMD: ethtool -C eth0 rx-usecs 0 rx-frames 1
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Disable interrupt coalescing for lowest latency (high CPU cost)
---

TOOL: ethtool
CMD: ethtool -k eth0 | grep generic-receive-offload
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Check if GRO (Generic Receive Offload) is enabled
---

TOOL: ethtool
CMD: ethtool -l eth0
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Show number of NIC receive/transmit queues for RSS
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_available_congestion_control
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: List available TCP congestion control algorithms
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_congestion_control=bbr
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Set BBR as the TCP congestion control algorithm system-wide
---

TOOL: ss
CMD: ss -ti dst <remote-ip> | grep bbr
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Verify BBR is active on a connection and show bandwidth estimate
---

TOOL: sysctl
CMD: sysctl net.netfilter.nf_conntrack_count
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Show current conntrack table entry count
---

TOOL: conntrack
CMD: conntrack -L | awk '{for(i=1;i<=NF;i++) if($i ~ /dport=/) print $i}' | sort | uniq -c | sort -rn | head
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Find which destination ports are generating most conntrack entries
---

TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_max=1000000
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Increase conntrack max to 1M entries
---

TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Set established TCP conntrack timeout to 1 day
---

TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_udp_timeout=15
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Reduce UDP conntrack timeout to 15s
---

TOOL: bpftool
CMD: bpftool prog load myprog.o /sys/fs/bpf/myprog 2>&1 | head -50
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Load eBPF program and capture verifier output for debugging
---

TOOL: bpftool
CMD: bpftool prog list | grep cilium
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Verify Cilium eBPF programs are loaded (cgroup, xdp, tc hooks)
---

TOOL: cilium
CMD: cilium bpf lb list
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Show ClusterIP endpoints in Cilium BPF load balancer maps
---

TOOL: cilium
CMD: cilium monitor --type drop
SOURCE: 10-interview-prep/09-exotic-deep-dive-qa.md
CONTEXT: Monitor dropped packets with policy drop reason in Cilium
---

<!-- Directory: 11-hands-on-labs/ -->

<!-- 01-linux-networking-lab.md -->

TOOL: ip
CMD: sudo ip netns add ns1
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Create a Linux network namespace (container simulation)
---

TOOL: ip
CMD: ip netns list
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: List all network namespaces on the system
---

TOOL: ip
CMD: sudo ip link add br0 type bridge
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Create a Linux bridge (analogous to Docker's docker0)
---

TOOL: ip
CMD: sudo ip link set br0 up
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Bring up the bridge interface
---

TOOL: ip
CMD: sudo ip addr add 10.0.0.1/24 dev br0
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Assign IP address to bridge interface
---

TOOL: ip
CMD: sudo ip link add veth1 type veth peer name veth1-ns
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Create veth pair for connecting namespace to bridge
---

TOOL: ip
CMD: sudo ip link set veth1-ns netns ns1
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Move veth end into network namespace
---

TOOL: ip
CMD: sudo ip link set veth1 master br0
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Attach host-side veth to bridge
---

TOOL: ip
CMD: sudo ip netns exec ns1 ip addr add 10.0.0.10/24 dev veth1-ns
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Assign IP inside a network namespace
---

TOOL: ip
CMD: sudo ip netns exec ns1 ping -c 3 10.0.0.20
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Ping from namespace ns1 to ns2 to verify connectivity
---

TOOL: ip
CMD: sudo ip netns exec ns1 ip neigh show
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Show ARP table inside namespace to verify MAC learning
---

TOOL: sysctl
CMD: sudo sysctl -w net.ipv4.ip_forward=1
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Enable IP forwarding on host for namespace internet access
---

TOOL: ip
CMD: sudo ip netns exec ns1 ip route add default via 10.0.0.1
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Add default route inside namespace via bridge gateway
---

TOOL: iptables
CMD: sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o "$HOST_IFACE" -j MASQUERADE
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Masquerade namespace traffic for internet access via NAT
---

TOOL: curl
CMD: sudo ip netns exec ns1 curl -s --max-time 5 http://1.1.1.1 | head -c 100
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Test internet access from inside a network namespace
---

TOOL: bridge
CMD: bridge fdb show dev br0
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Show L2 forwarding database — MACs known on each bridge port
---

TOOL: ip
CMD: ip link show type bridge
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: List all bridge interfaces on the system
---

TOOL: ip
CMD: ip link show master br0
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: List all interfaces attached to a bridge
---

TOOL: ip
CMD: sudo ip netns del ns1
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Delete a network namespace
---

TOOL: conntrack
CMD: conntrack -L 2>/dev/null | wc -l
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Count current conntrack table entries
---

TOOL: conntrack
CMD: conntrack -L 2>/dev/null | head -20
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Inspect first 20 conntrack entries to see TIME_WAIT buildup
---

TOOL: conntrack
CMD: conntrack -L 2>/dev/null | grep TIME_WAIT | wc -l
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Count TIME_WAIT entries consuming conntrack table slots
---

TOOL: nstat
CMD: nstat -az | grep -i conntrack
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Check kernel conntrack overflow counters
---

TOOL: sysctl
CMD: sudo sysctl -w net.netfilter.nf_conntrack_max=131072
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Increase conntrack max to 131072 entries
---

TOOL: sysctl
CMD: sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Reduce TIME_WAIT conntrack timeout to free slots faster
---

TOOL: ss
CMD: ss -tnlp | grep 8080
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Verify service is listening on port 8080
---

TOOL: iptables
CMD: sudo iptables -I INPUT -p tcp --dport 8080 -j DROP
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Add DROP rule at top of INPUT chain for testing
---

TOOL: iptables
CMD: sudo iptables -L INPUT -n -v --line-numbers
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: List INPUT rules with line numbers and packet counters
---

TOOL: sysctl
CMD: sudo sysctl -w net.netfilter.nf_log.2=nf_log_ipv4
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Enable IPv4 netfilter logging for TRACE target
---

TOOL: iptables
CMD: sudo iptables -t raw -I PREROUTING -p tcp --dport 8080 -j TRACE
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Enable packet tracing for traffic destined for port 8080
---

TOOL: iptables
CMD: sudo iptables -t raw -I OUTPUT -p tcp --dport 8080 -j TRACE
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Enable TRACE for locally-generated traffic to port 8080
---

TOOL: iptables
CMD: sudo iptables -t filter -L INPUT -n -v --line-numbers
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Identify which specific rule number is dropping traffic
---

TOOL: iptables
CMD: sudo iptables -D INPUT 1
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Remove iptables DROP rule by line number
---

TOOL: iptables
CMD: sudo iptables -D INPUT -p tcp --dport 8080 -j DROP
SOURCE: 11-hands-on-labs/01-linux-networking-lab.md
CONTEXT: Remove iptables DROP rule by specification
---

<!-- 02-kubernetes-networking-lab.md -->

TOOL: kubectl
CMD: kubectl get nodes -o wide
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: List nodes with IP addresses for cross-node packet tracing
---

TOOL: kubectl
CMD: kubectl exec pod-a -- ip addr show eth0
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Show pod network interface and IP address
---

TOOL: kubectl
CMD: kubectl exec pod-a -- ip route show
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Show routing table inside a pod
---

TOOL: kubectl
CMD: kubectl exec pod-a -- ping -c 3 $POD_B_IP
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Test pod-to-pod connectivity across nodes
---

TOOL: tcpdump
CMD: docker exec $NODE1 bash -c "tcpdump -i eth0 -n 'udp port 8472' -c 5"
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Capture VXLAN encapsulated traffic between Kubernetes nodes
---

TOOL: kubectl
CMD: kubectl exec pod-a -- cat /etc/resolv.conf
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Inspect pod DNS configuration including ndots setting
---

TOOL: kubectl
CMD: kubectl exec pod-a -- time nslookup google.com
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Measure DNS resolution time with ndots:5 search expansion
---

TOOL: kubectl
CMD: kubectl exec pod-a -- dig google.com +search +stats 2>/dev/null | grep -E "Query time|status"
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Show DNS query stats including latency for ndots comparison
---

TOOL: kubectl
CMD: kubectl exec pod-a -- time nslookup google.com.
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Test DNS with trailing dot to bypass ndots search expansion
---

TOOL: kubectl
CMD: kubectl exec -n netpol-lab frontend -- curl -s http://backend-svc/
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Test frontend-to-backend connectivity before NetworkPolicy
---

TOOL: kubectl
CMD: kubectl exec -n netpol-lab frontend -- nslookup backend-svc.netpol-lab.svc.cluster.local
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Test DNS resolution of Kubernetes service FQDN
---

TOOL: kubectl
CMD: kubectl describe networkpolicy frontend-policy -n netpol-lab
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Inspect NetworkPolicy rules to find missing DNS egress
---

TOOL: kubectl
CMD: kubectl get pods -n kube-system -l k8s-app=kube-dns
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Find CoreDNS pods in kube-system namespace
---

TOOL: kubectl
CMD: kubectl get svc -n kube-system kube-dns
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Get CoreDNS ClusterIP for NetworkPolicy DNS egress rule
---

TOOL: kubectl
CMD: kubectl exec -n netpol-lab frontend -- curl --max-time 3 http://1.1.1.1 || echo "BLOCKED (expected)"
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Verify internet egress is correctly blocked by NetworkPolicy
---

TOOL: kubectl
CMD: kubectl get configmap -n kube-system kube-proxy -o jsonpath='{.data.config\.conf}' | grep mode
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Check kube-proxy operating mode (iptables vs IPVS)
---

TOOL: iptables
CMD: iptables -t nat -L KUBE-SERVICES -n | grep "$CLUSTER_IP"
SOURCE: 11-hands-on-labs/02-kubernetes-networking-lab.md
CONTEXT: Find kube-proxy service chain for a specific ClusterIP
---

<!-- 03-aws-vpc-lab.md -->

TOOL: aws
CMD: aws sts get-caller-identity
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Verify AWS CLI credentials and show account identity
---

TOOL: aws
CMD: aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc}]' --query 'Vpc.VpcId' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Create a VPC with a /16 CIDR block and name tag
---

TOOL: aws
CMD: aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Enable DNS hostnames for VPC (required for RDS)
---

TOOL: aws
CMD: aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Create public subnet in specific AZ
---

TOOL: aws
CMD: aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Create Internet Gateway for public subnet internet access
---

TOOL: aws
CMD: aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Attach Internet Gateway to VPC
---

TOOL: aws
CMD: aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Allocate Elastic IP for NAT Gateway
---

TOOL: aws
CMD: aws ec2 create-nat-gateway --subnet-id $PUBLIC_1A --allocation-id $EIP_ALLOC --query 'NatGateway.NatGatewayId' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Create NAT Gateway in public subnet for private subnet egress
---

TOOL: aws
CMD: aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Wait for NAT Gateway to become available (~60-90 seconds)
---

TOOL: aws
CMD: aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Create a new route table for a VPC tier
---

TOOL: aws
CMD: aws ec2 create-route --route-table-id $PUBLIC_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Add default route to Internet Gateway in public route table
---

TOOL: aws
CMD: aws ec2 associate-route-table --subnet-id $PUBLIC_1A --route-table-id $PUBLIC_RT
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Associate a subnet with a route table
---

TOOL: aws
CMD: aws ec2 modify-subnet-attribute --subnet-id $PUBLIC_1A --map-public-ip-on-launch
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Enable auto-assign public IP for instances in public subnet
---

TOOL: aws
CMD: aws ec2 create-route --route-table-id $PRIVATE_RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Add default route via NAT Gateway for private subnet egress
---

TOOL: aws
CMD: aws ec2 describe-route-tables --route-table-ids $PUBLIC_RT --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,GatewayId]' --output table
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Verify public route table has IGW default route
---

TOOL: aws
CMD: aws ec2 create-security-group --group-name bastion-sg --description "Bastion host SG" --vpc-id $VPC_ID --query 'GroupId' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Create security group for bastion host
---

TOOL: aws
CMD: aws ec2 authorize-security-group-ingress --group-id $BASTION_SG --protocol tcp --port 22 --cidr 0.0.0.0/0
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Allow SSH inbound to bastion security group
---

TOOL: aws
CMD: aws ec2 authorize-security-group-ingress --group-id $APPSERVER_SG --protocol tcp --port 22 --source-group $BASTION_SG
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Allow SSH from bastion SG to app server (SG-to-SG reference)
---

TOOL: aws
CMD: aws ec2 create-network-acl --vpc-id $VPC_ID --query 'NetworkAcl.NetworkAclId' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Create custom Network ACL for subnet-level stateless filtering
---

TOOL: aws
CMD: aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL --ingress --rule-number 100 --protocol tcp --port-range From=22,To=22 --cidr-block 0.0.0.0/0 --rule-action allow
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Add inbound NACL rule allowing SSH
---

TOOL: aws
CMD: aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL --egress --rule-number 400 --protocol tcp --port-range From=1024,To=65535 --cidr-block 0.0.0.0/0 --rule-action allow
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Add outbound NACL rule for ephemeral return traffic ports
---

TOOL: aws
CMD: aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL --egress --rule-number 500 --protocol udp --port-range From=1024,To=65535 --cidr-block 0.0.0.0/0 --rule-action allow
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Add outbound NACL rule for UDP ephemeral ports (DNS returns)
---

TOOL: aws
CMD: aws ec2 describe-network-acls --network-acl-ids $PRIVATE_NACL --query 'NetworkAcls[0].Entries[*].[RuleNumber,Protocol,PortRange,Egress,RuleAction]' --output table
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: List NACL rules to verify ephemeral port rules are present
---

TOOL: aws
CMD: aws logs create-log-group --log-group-name /aws/vpc/flowlogs
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Create CloudWatch log group for VPC Flow Logs
---

TOOL: aws
CMD: aws ec2 create-flow-logs --resource-ids $VPC_ID --resource-type VPC --traffic-type ALL --log-destination-type cloud-watch-logs --log-group-name /aws/vpc/flowlogs --deliver-logs-permission-arn $FLOW_LOG_ROLE
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Enable VPC Flow Logs to CloudWatch for NACL/SG diagnosis
---

TOOL: aws
CMD: aws ec2 create-vpc-peering-connection --vpc-id $VPC_A --peer-vpc-id $VPC_B --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Create VPC peering connection request between two VPCs
---

TOOL: aws
CMD: aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEERING_ID
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Accept pending VPC peering connection request
---

TOOL: aws
CMD: aws ec2 describe-vpc-peering-connections --vpc-peering-connection-ids $PEERING_ID --query 'VpcPeeringConnections[0].Status.Code' --output text
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Verify VPC peering connection status is active
---

TOOL: aws
CMD: aws ec2 create-route --route-table-id $PRIVATE_RT --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PEERING_ID
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Add route to VPC-B's CIDR via peering in VPC-A route table
---

TOOL: aws
CMD: aws ec2 describe-route-tables --route-table-ids $ROUTE_TABLE_B --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId,GatewayId]' --output table
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Verify VPC-B route table has return route to VPC-A
---

TOOL: aws
CMD: aws ec2 create-route --route-table-id $ROUTE_TABLE_B --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id $PEERING_ID
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Add missing return route in VPC-B for asymmetric peering fix
---

TOOL: aws
CMD: aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $PEERING_ID
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Delete VPC peering connection during cleanup
---

TOOL: aws
CMD: aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Delete NAT Gateway to avoid ongoing charges
---

TOOL: aws
CMD: aws ec2 release-address --allocation-id $EIP_ALLOC
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Release Elastic IP allocation after deleting NAT Gateway
---

TOOL: aws
CMD: aws ec2 detach-internet-gateway --vpc-id $VPC_A --internet-gateway-id $IGW_ID
SOURCE: 11-hands-on-labs/03-aws-vpc-lab.md
CONTEXT: Detach Internet Gateway from VPC before deletion
---

<!-- 04-tls-debugging-lab.md -->

TOOL: openssl
CMD: openssl genrsa -out rootCA.key 4096
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Generate 4096-bit RSA private key for Root CA
---

TOOL: openssl
CMD: openssl req -new -x509 -key rootCA.key -sha256 -days 3650 -out rootCA.pem -subj "/C=US/ST=California/O=LabCA/CN=Lab Root CA"
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Create self-signed Root CA certificate valid 10 years
---

TOOL: openssl
CMD: openssl x509 -noout -text -in rootCA.pem | grep -E "Subject:|Issuer:|Not (Before|After)|CA:"
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Verify Root CA certificate attributes and validity
---

TOOL: openssl
CMD: openssl genrsa -out intermediateCA.key 4096
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Generate private key for Intermediate CA
---

TOOL: openssl
CMD: openssl req -new -key intermediateCA.key -out intermediateCA.csr -subj "/C=US/ST=California/O=LabCA/CN=Lab Intermediate CA"
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Create Certificate Signing Request for Intermediate CA
---

TOOL: openssl
CMD: openssl x509 -req -in intermediateCA.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out intermediateCA.pem -days 1825 -sha256 -extensions v3_intermediate_ca -extfile intermediate-ext.cnf
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Sign Intermediate CA certificate with Root CA
---

TOOL: openssl
CMD: openssl verify -CAfile rootCA.pem intermediateCA.pem
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Verify Intermediate CA certificate against Root CA
---

TOOL: openssl
CMD: openssl genrsa -out server.key 2048
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Generate 2048-bit RSA server private key
---

TOOL: openssl
CMD: openssl req -new -key server.key -out server.csr -subj "/C=US/ST=California/O=LabApp/CN=localhost" -config server-ext.cnf
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Create server CSR with SAN extensions for localhost
---

TOOL: openssl
CMD: openssl x509 -req -in server.csr -CA intermediateCA.pem -CAkey intermediateCA.key -CAcreateserial -out server.pem -days 365 -sha256 -extensions v3_req -extfile server-ext.cnf
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Sign server certificate with Intermediate CA
---

TOOL: openssl
CMD: openssl verify -CAfile rootCA.pem -untrusted intermediateCA.pem server.pem
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Verify full certificate chain from server to root
---

TOOL: openssl
CMD: openssl s_client -connect localhost:8443 -CAfile $LAB_DIR/rootCA.pem -quiet
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Test TLS connection with custom CA and show chain depth
---

TOOL: curl
CMD: curl -s --cacert $LAB_DIR/rootCA.pem https://localhost:8443/
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Test HTTPS with custom CA file
---

TOOL: openssl
CMD: openssl x509 -noout -dates -in expired.pem
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Check certificate validity dates (notBefore, notAfter)
---

TOOL: openssl
CMD: openssl s_client -connect localhost:8443 -CAfile $LAB_DIR/rootCA.pem 2>&1 | grep -E "Verify return:|error|expire|notAfter"
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Check TLS handshake for certificate expiry errors
---

TOOL: openssl
CMD: openssl x509 -noout -checkend 0 -in expired.pem
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Check if certificate is currently expired (exit 1 = expired)
---

TOOL: openssl
CMD: openssl x509 -noout -checkend 2592000 -in server.pem
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Check if certificate expires within 30 days for monitoring
---

TOOL: openssl
CMD: openssl s_client -connect localhost:8443 -CAfile $LAB_DIR/rootCA.pem 2>&1 | grep -E "Certificate chain|depth|verify"
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Check certificate chain depth to find missing intermediate
---

TOOL: openssl
CMD: openssl s_client -connect localhost:8443 -showcerts 2>/dev/null | grep -c "BEGIN CERTIFICATE"
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Count certificates in TLS chain (should be 2: leaf + intermediate)
---

TOOL: openssl
CMD: openssl verify -CAfile $LAB_DIR/rootCA.pem -untrusted $LAB_DIR/intermediateCA.pem $LAB_DIR/server.pem
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Manually verify cert chain providing intermediate explicitly
---

TOOL: openssl
CMD: openssl x509 -noout -issuer -in $LAB_DIR/server.pem
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Show issuer of leaf cert to identify required intermediate
---

TOOL: openssl
CMD: openssl genrsa -out clientCA.key 4096
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Generate Client CA private key for mTLS setup
---

TOOL: openssl
CMD: openssl req -new -x509 -key clientCA.key -sha256 -days 1825 -out clientCA.pem -subj "/C=US/ST=California/O=LabCA/CN=Lab Client CA"
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Create self-signed Client CA certificate for mTLS
---

TOOL: openssl
CMD: openssl genrsa -out client.key 2048
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Generate client private key for mTLS authentication
---

TOOL: openssl
CMD: openssl x509 -req -in client.csr -CA clientCA.pem -CAkey clientCA.key -CAcreateserial -out client.pem -days 365 -sha256
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Sign client certificate with Client CA for mTLS
---

TOOL: openssl
CMD: openssl verify -CAfile $LAB_DIR/clientCA.pem $LAB_DIR/client.pem
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Verify client certificate against its Client CA
---

TOOL: curl
CMD: curl -s --cacert $LAB_DIR/rootCA.pem --cert $LAB_DIR/client.pem --key $LAB_DIR/client.key https://localhost:8443/
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Test mTLS connection with client certificate
---

TOOL: openssl
CMD: openssl verify -CAfile $LAB_DIR/clientCA.pem $LAB_DIR/rogue-client.pem
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Simulate mTLS verification failure with wrong CA
---

TOOL: openssl
CMD: openssl s_client -connect localhost:8443 -CAfile $LAB_DIR/rootCA.pem -cert $LAB_DIR/rogue-client.pem -key $LAB_DIR/rogue-client.key -brief 2>&1
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Test mTLS handshake failure with untrusted client cert
---

TOOL: openssl
CMD: openssl s_client -connect host:443
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Basic TLS connection test
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -CAfile /etc/ssl/certs/ca-certificates.crt
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: TLS connection test with system CA bundle
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -showcerts
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Show full certificate chain from TLS server
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -cert client.pem -key client.key
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: mTLS test: connect with client certificate and key
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -tls1_2
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Test TLS 1.2 specifically
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -tls1_3
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Test TLS 1.3 specifically
---

TOOL: openssl
CMD: echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -text
SOURCE: 11-hands-on-labs/04-tls-debugging-lab.md
CONTEXT: Non-interactive TLS check extracting certificate text
---

<!-- 05-ebpf-xdp-lab.md -->

TOOL: bpftool
CMD: sudo bpftool version
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Verify bpftool is installed and show version
---

TOOL: ip
CMD: sudo ip link add veth-test type veth peer name veth-peer
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Create veth pair for XDP testing
---

TOOL: bpftool
CMD: sudo bpftool prog load xdp_counter.o /sys/fs/bpf/xdp_counter
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Load XDP eBPF program from compiled object file
---

TOOL: ip
CMD: sudo ip link set veth-test xdpgeneric obj xdp_counter.o sec xdp
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Attach XDP program to interface in generic/skb mode
---

TOOL: bpftool
CMD: sudo bpftool map list
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: List all loaded BPF maps with IDs and types
---

TOOL: bpftool
CMD: sudo bpftool map dump id $MAP_ID
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Dump all entries in a BPF map
---

TOOL: ip
CMD: sudo ip link set veth-test xdpgeneric off
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Detach XDP program from interface
---

TOOL: bpftool
CMD: sudo bpftool map update id $BLOCKLIST_ID key hex c0 a8 63 03 value hex 01 00 00 00
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Add IP to XDP blocklist BPF map to trigger XDP_DROP
---

TOOL: bpftool
CMD: sudo bpftool prog show
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: List all loaded eBPF programs with metadata
---

TOOL: bpftool
CMD: sudo bpftool prog dump xlated id 42
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Disassemble JIT-compiled eBPF program instructions
---

TOOL: iptables
CMD: sudo iptables -I INPUT -s 192.168.99.3 -j DROP
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Add iptables DROP rule for performance comparison with XDP
---

TOOL: bpftrace
CMD: sudo bpftrace -e 'tracepoint:syscalls:sys_enter_connect { printf("%-10s %-6d connect() called\n", comm, pid); }'
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Trace all processes making connect() syscalls
---

TOOL: bpftrace
CMD: sudo bpftrace -e 'kprobe:tcp_retransmit_skb { @retransmits[comm] = count(); } END { print(@retransmits); clear(@retransmits); }'
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Count TCP retransmits per process for packet loss diagnosis
---

TOOL: bpftrace
CMD: sudo bpftrace -e 'kprobe:tcp_v4_connect { @start[tid] = nsecs; } kretprobe:tcp_v4_connect / @start[tid] / { @conn_latency_us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }'
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Measure TCP connection latency histogram (handshake RTT)
---

TOOL: bpftrace
CMD: sudo bpftrace -e 'tracepoint:net:net_dev_xmit { @[comm] = count(); }'
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Count outbound packets per process using tracepoint
---

TOOL: bpftrace
CMD: sudo bpftrace -e 'kprobe:tcp_sendmsg { @bytes[comm] = sum(arg2); }'
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Track TCP bytes sent per process
---

TOOL: bpftrace
CMD: sudo bpftrace -l 'tracepoint:net:*'
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: List available network tracepoints for bpftrace
---

TOOL: bpftrace
CMD: sudo bpftrace -l 'tracepoint:tcp:*'
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: List available TCP tracepoints for bpftrace
---

TOOL: bpftrace
CMD: sudo bpftrace -l 'kprobe:tcp_*' | head -20
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: List available TCP kernel probe points for bpftrace
---

TOOL: cilium
CMD: cilium install --version 1.14.0
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Install Cilium CNI in a Kubernetes cluster
---

TOOL: cilium
CMD: cilium status --wait
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Wait for Cilium to be fully ready
---

TOOL: cilium
CMD: cilium hubble enable --ui
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Enable Hubble observability relay and UI
---

TOOL: hubble
CMD: hubble observe --namespace hubble-lab --follow
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Watch live Kubernetes network flows in a namespace
---

TOOL: hubble
CMD: hubble observe --from-pod hubble-lab/client --follow
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Filter Hubble flows by source pod
---

TOOL: hubble
CMD: hubble observe --verdict DROPPED --namespace hubble-lab
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Show only dropped flows to identify NetworkPolicy blocks
---

TOOL: hubble
CMD: hubble observe --type drop --namespace hubble-lab
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Show dropped packets with drop reason in Cilium
---

TOOL: hubble
CMD: hubble observe --protocol dns --namespace hubble-lab
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Show all DNS queries and responses in a namespace
---

TOOL: hubble
CMD: hubble observe --protocol http --namespace hubble-lab
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Show HTTP flows with request/response metadata
---

TOOL: hubble
CMD: hubble status
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Show Hubble relay status and connected node count
---

TOOL: cilium
CMD: cilium connectivity test
SOURCE: 11-hands-on-labs/05-ebpf-xdp-lab.md
CONTEXT: Run Cilium built-in connectivity tests across the cluster
---

<!-- Additional commands from cheat sheets in 00-study-guide.md -->

TOOL: ss
CMD: ss -tnp
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: List TCP connections with PIDs — first command for connectivity issues
---

TOOL: ss
CMD: ss -s
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Socket state summary showing counts by state
---

TOOL: ss
CMD: ss -tlnp
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show listening ports with process info
---

TOOL: ip
CMD: ip route show table all
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show all routing tables including policy routing and VPN routes
---

TOOL: conntrack
CMD: conntrack -L | wc -l
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Count current conntrack entries to compare to nf_conntrack_max
---

TOOL: tcpdump
CMD: tcpdump -i eth0 -nn -s0 port 443
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Packet capture on port 443 for network-level debugging
---

TOOL: tcpdump
CMD: tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Capture SYN packets only for DDoS and connection debugging
---

TOOL: ethtool
CMD: ethtool -S eth0 | grep -i drop
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Check NIC drop counters for ring buffer and driver drops
---

TOOL: ethtool
CMD: ethtool -c eth0
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show interrupt coalescing settings to debug high NIC interrupt CPU
---

TOOL: nstat
CMD: nstat -a | grep -i retrans
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Check TCP retransmission counters for packet loss and congestion
---

TOOL: bpftool
CMD: bpftool prog show
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: List loaded eBPF programs for XDP/TC debugging
---

TOOL: nft
CMD: nft list ruleset
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show all nftables rules for modern kernel firewall audit
---

TOOL: iptables
CMD: iptables -L FORWARD -n -v --line-numbers
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: List iptables FORWARD rules for container and Kubernetes pod-to-pod
---

TOOL: ip
CMD: ip netns exec <ns> <cmd>
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Run command in network namespace for container/pod debugging
---

TOOL: dig
CMD: dig +trace <name>
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Full DNS resolution trace for delegation failure diagnosis
---

TOOL: dig
CMD: dig @<dns-ip> <name>
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Query specific DNS server to isolate stub vs recursive vs auth
---

TOOL: curl
CMD: curl -w "dns:%{time_namelookup} connect:%{time_connect} ttfb:%{time_starttransfer}"
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: HTTP timing breakdown by phase for latency diagnosis
---

TOOL: ping
CMD: ping -M do -s 1472 <IP>
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Test path MTU with DF bit set for PMTUD black hole detection
---

TOOL: traceroute
CMD: traceroute -T -p 443 <host>
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: TCP traceroute for firewall detection and path analysis
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_retries2
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show max TCP retransmissions before connection timeout
---

TOOL: bpftrace
CMD: bpftrace -e 'kprobe:tcp_drop { @[kstack] = count(); }'
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Trace where kernel drops TCP packets (advanced kernel debugging)
---

TOOL: kubectl
CMD: kubectl get endpoints <svc>
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Check Kubernetes service endpoints to debug routing failures
---

TOOL: kubectl
CMD: kubectl exec <pod> -- ss -tnp
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Run ss inside a pod for Kubernetes networking debugging
---

TOOL: bridge
CMD: bridge fdb show dev <vxlan-if>
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show VXLAN forwarding database for overlay debugging
---

TOOL: ip
CMD: ip -s link show eth0
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show interface statistics with TX/RX errors and drops
---

TOOL: ethtool
CMD: ethtool -S eth0
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show NIC statistics including all driver counters
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_keepalive_time
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show keepalive idle time before first probe
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_keepalive_intvl
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show keepalive probe interval
---

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_keepalive_probes
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show keepalive probe count before RST
---

TOOL: ip
CMD: ip route get <IP>
SOURCE: 10-interview-prep/00-study-guide.md
CONTEXT: Show which route kernel selects including interface and gateway
---
