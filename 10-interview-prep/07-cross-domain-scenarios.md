# Cross-Domain Scenarios — Interview Q&A

## Quick Reference

Cross-domain scenarios test whether you can reason across the full stack: Linux kernel → container runtime → Kubernetes → cloud networking → security policy. Real production incidents don't stay in one domain — a Kubernetes pod can't reach RDS because of a combination of NACL, Security Group, DNS, and certificate issues. These scenarios require a systematic layer-by-layer approach. Source material: `06_Cross_Domain_Integration.md` (extended). See all sections 01-09 for depth on individual layers.

---

## Q: A Kubernetes pod on Node A cannot reach a pod on Node B. Both have IPs from Calico BGP mode. Walk through every failure point.

**What the interviewer is testing:** Whether you can trace the complete cross-node packet path and map failure modes to each hop.

**Model Answer:**

In Calico BGP mode, each node announces its pod CIDR via BGP. The packet path: Pod A → veth → Node A host routing table → physical network → Node B host routing table → veth → Pod B.

**Systematic debugging:**
```bash
# Step 1: Same-node connectivity (isolates veth/pod-level issues)
kubectl exec -n <ns> <pod-a> -- ping <node-a-ip>
# Fail = veth pair broken or pod routing broken

# Step 2: Check Node A routing table for Node B's pod CIDR
ssh node-a ip route show | grep 10.244.2.0
# Expected: 10.244.2.0/24 via <node-b-ip> dev eth0 proto bird
# Missing = BGP not propagating routes

# Step 3: Check BIRD (Calico BGP daemon) session state
kubectl exec -n calico-system <calico-node-a> -c calico-node -- birdcl show protocols
# Session to Node B should be "Established"
# "Connect" or "Active" = BGP session down (firewall blocking TCP 179?)

# Step 4: Check BGP routes received
kubectl exec -n calico-system <calico-node-a> -c calico-node -- \
  birdcl show route protocol <node-b-peer>
# Should show: 10.244.2.0/24 via <node-b-ip>

# Step 5: Physical network anti-spoofing
# AWS: source/dest check must be disabled on EC2 instances
aws ec2 describe-instance-attribute --instance-id <id> --attribute sourceDestCheck
# On-prem: ToR switches need routes for pod CIDRs

# Step 6: Node B return path
ssh node-b ip route show | grep 10.244.1.0
# Missing = asymmetric routing — packets arrive but replies are dropped

# Step 7: iptables FORWARD chain (NetworkPolicy enforcement)
iptables -L FORWARD -n -v | grep DROP
iptables -L cali-FORWARD -n -v    # Calico-specific chain
```

**Three most common root causes in order of probability:**
1. **BGP session not established** — firewall blocking TCP/179 between nodes
2. **Anti-spoofing on underlay** — cloud VPC drops packets with unexpected src IPs
3. **IP forwarding disabled** — `sysctl net.ipv4.ip_forward=0` on a node

**Follow-up Questions:**
- How does the debugging differ if Calico is using IPIP mode instead of BGP?
- What is the impact of a route reflector failure in a 500-node cluster?
- How do you verify the FORWARD chain is allowing pod-to-pod traffic?

**Connecting Points:**
- `04-kubernetes-networking/` for Calico BGP architecture
- `02-linux-networking/` for iptables FORWARD chain

---

## Q: Docker containers are experiencing intermittent DNS SERVFAIL. The host DNS works fine. Explain the Docker DNS architecture and debug path.

**What the interviewer is testing:** Whether you understand Docker's dual DNS architecture and the conntrack UDP race condition.

**Model Answer:**

Docker has two DNS behaviors:
- Default bridge (docker0): uses host's /etc/resolv.conf directly
- User-defined bridge: embedded DNS server at 127.0.0.11 inside the container

```bash
# Identify which DNS model is active
docker exec <container> cat /etc/resolv.conf
# "nameserver 127.0.0.11" → embedded DNS (most deployments)
# "nameserver 8.8.8.8"   → host DNS directly

# Check if conntrack is accumulating insert_failed (UDP race condition)
conntrack -S | grep insert_failed
# Incrementing = conntrack UDP race condition
watch -n1 'conntrack -S | grep insert_failed'
```

**The conntrack UDP race condition:**
When a container makes a DNS query, glibc sends A and AAAA queries simultaneously. Both UDP packets traverse the conntrack path. If both arrive at the conntrack insertion point simultaneously on different CPUs, one insertion fails silently. The reply to the failed-to-track query is dropped (no matching conntrack entry). This manifests as intermittent SERVFAIL or timeout.

**Fixes:**
```bash
# Fix 1: Disable userland proxy (reduces single-threaded bottleneck)
# /etc/docker/daemon.json:
{ "userland-proxy": false }
# Restart Docker

# Fix 2: Check Docker's NAT rules (may be corrupted/missing)
iptables -t nat -L DOCKER_OUTPUT -n -v
iptables -t nat -L DOCKER_POSTROUTING -n -v
# Missing rules = restart Docker daemon to regenerate

# Fix 3: Use single-request-reopen (sequential A/AAAA queries)
docker run --dns-opt=single-request-reopen <image>

# Fix 4: Reduce search domains (amplification)
docker exec <container> cat /etc/resolv.conf | grep search
docker run --dns-search=. <image>    # empty search domain
```

**Follow-up Questions:**
- How does Podman handle DNS differently (no daemon)?
- What happens to DNS records when a container is stopped?
- How does Docker Compose create service name resolution?

**Connecting Points:**
- `04-kubernetes-networking/` for CoreDNS and K8s DNS architecture (similar conntrack issue)
- `02-linux-networking/` for conntrack UDP race condition

---

## Q: Your AWS VPC (10.0.0.0/16) needs to peer with an on-premises network that also uses 10.0.0.0/16. How do you solve overlapping CIDRs?

**What the interviewer is testing:** Whether you understand why VPC peering requires non-overlapping CIDRs and know the alternatives.

**Model Answer:**

VPC peering adds routes to each VPC's route table. With identical CIDRs (10.0.0.0/16 on both sides), routing is ambiguous — both destinations look the same. Direct peering is impossible.

**Solution 1: PrivateLink (best for service-oriented connectivity):**
```bash
# In the AWS VPC: create Network Load Balancer for the service
# Create Endpoint Service pointing to the NLB
aws elbv2 create-load-balancer --name payment-nlb --type network \
  --subnets subnet-XXXXXXXX

aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:...

# In the on-premises VPC (or another AWS VPC): create Endpoint
# The endpoint gets an IP from the consumer subnet — no CIDR conflict
aws ec2 create-vpc-endpoint \
  --service-name com.amazonaws.vpce.<region>.<svc-id> \
  --vpc-endpoint-type Interface \
  --vpc-id vpc-XXXXXXXX
```

PrivateLink resolves the CIDR conflict because the endpoint IP is local to the consumer network. The consumer accesses the service via the endpoint IP, not the provider's actual 10.x.x.x IP. Limitation: unidirectional (consumer to provider only).

**Solution 2: NAT double-translation (for full bidirectional connectivity):**
```bash
# Map on-prem 10.0.0.0/16 to 172.16.0.0/16 (non-conflicting range)
# On the connection boundary (VPN gateway or TGW with NAT instance):
iptables -t nat -A POSTROUTING -s 10.0.0.0/16 -d 10.0.0.0/16 -j NETMAP --to 172.16.0.0/16
# On-prem accesses AWS 10.x → connects to 172.16.x (NAT'd)
# AWS accesses on-prem 10.x → connects to 172.16.x (NAT'd)
# DNS must be translated too: on-prem records pointing to 10.x must return 172.16.x
```

**Solution 3: Re-address one side (clean long-term fix):**
- Create new AWS VPC with different CIDR (e.g., 10.1.0.0/16)
- Migrate services incrementally (strangler fig pattern)
- Peering works normally after re-addressing

**Trade-off matrix:**

| Solution | Bidirectional | DNS complexity | Operational cost |
|----------|--------------|----------------|-----------------|
| PrivateLink | No (svc oriented) | Low | Low |
| Double-NAT | Yes | High (DNS rewrite) | High |
| Re-address | Yes (after migration) | None | High (one-time) |

**Follow-up Questions:**
- What is the maximum number of routes in an AWS VPC route table?
- How does IPv6 solve the CIDR overlap problem?
- How does AWS Transit Gateway NAT interact with overlapping CIDRs?

**Connecting Points:**
- `03-cloud-networking/` for VPC peering and PrivateLink architecture
- `10-interview-prep/03-cloud-networking-qa.md` for VPC peering route tables

---

## Q: After an Istio upgrade, services have 5x higher latency on the first request after a period of inactivity. Subsequent requests are fast. What's causing this?

**What the interviewer is testing:** Whether you understand Envoy's connection pool and the cold-start latency sources.

**Model Answer:**

This is Envoy connection pool cold-start. The 5x latency pattern (slow first request, fast subsequent requests) is a signature of re-establishing idle connections.

**Envoy data path latency breakdown:**
```
Application → [iptables REDIRECT to port 15001] → Envoy outbound
  → Connection pool lookup → [if pool empty: new TCP connection]
  → [mTLS handshake to upstream Envoy: ~2-5ms]
  → Upstream Envoy inbound port 15006 → application
```

**Idle connection expiry sources (from the upgrade):**

1. **HTTP/2 connection idle timeout changed:**
```bash
# Check Envoy's HTTP/2 idle timeout
# Default was increased in Istio 1.19+ from 10m to 1h for HTTP/2
# But some configs have it lower:
kubectl exec -n production <pod> -c istio-proxy -- \
  pilot-agent request GET /config_dump | jq '.configs[] |
  select(.["@type"] | contains("type.googleapis.com/envoy.admin.v3.ClustersConfigDump")) |
  .dynamic_active_clusters[].cluster.http2_protocol_options.connection_keepalive'
```

2. **mTLS certificate rotation during upgrade:**
```bash
# If certificates expired during the upgrade, Envoy closes existing connections
# and must re-establish (new TLS handshake = latency)
istioctl proxy-status    # check sync status with istiod
kubectl exec <pod> -c istio-proxy -- openssl x509 -in /etc/certs/cert-chain.pem -noout -dates
```

3. **Envoy worker thread restart:**
```bash
# Drain timeout changes in the upgrade may have restarted workers
# Check Envoy admin:
kubectl exec -n production <pod> -c istio-proxy -- \
  curl localhost:15000/config_dump | jq '.configs[] |
  select(.["@type"] | contains("ListenersConfigDump"))'
```

**Fix:**

For idle connection expiry:
```yaml
# Configure connection pool settings to keep connections warm
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
spec:
  host: my-service.default.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        http2MaxRequests: 1000
        idleTimeout: 3600s      # keep connections for 1 hour
      tcp:
        maxConnections: 100
        connectTimeout: 5s
        tcpKeepalive:
          time: 7200s
          interval: 75s
```

**Follow-up Questions:**
- How does Envoy's Aggregate Discovery Service (ADS) work for configuration updates?
- What is the performance cost of mTLS per connection vs per request?
- How do you tune Envoy's connection pool for a gRPC streaming service?

**Connecting Points:**
- `04-kubernetes-networking/` for Istio architecture
- `10-interview-prep/05-network-security-qa.md` for mTLS certificate rotation

---

## Q: A MetalLB LoadBalancer service is unreachable from outside the cluster. Pods can reach the service internally. Debug this.

**What the interviewer is testing:** Whether you understand MetalLB's Layer 2 ARP mechanism and BGP mode.

**Model Answer:**

MetalLB provides LoadBalancer services for bare-metal clusters (no cloud provider). Two modes: Layer 2 (ARP/NDP) and BGP.

**Layer 2 mode debugging:**
```bash
# Step 1: Verify MetalLB assigned an IP
kubectl get svc <service-name>
# EXTERNAL-IP should be an IP from your MetalLB address pool
# "Pending" = no IPs available in the pool or pool not configured

# Step 2: Check which node is announcing the IP
# In Layer 2 mode, one node "owns" the IP and responds to ARP requests
kubectl logs -n metallb-system -l component=speaker | grep <external-ip>

# Step 3: Check ARP table on the router/switch
# From the upstream router, verify ARP entry for the MetalLB IP
arp -n | grep <metallb-ip>
# If absent: the speaker is not sending GARP, or ARP is blocked

# Step 4: Verify ARP announcement
# MetalLB sends GARP when it takes ownership of an IP
tcpdump -ni <uplink-interface> arp | grep <metallb-ip>

# Step 5: Check iptables rules on the speaker node
# The speaker node must have iptables rules allowing inbound traffic to the VIP
iptables -L INPUT -n -v | grep <metallb-ip>
```

**BGP mode debugging:**
```bash
# Step 1: Verify BGP session status
kubectl logs -n metallb-system -l component=speaker | grep "BGP session"

# Step 2: Check BGP peers from the speaker
kubectl exec -n metallb-system <speaker-pod> -- metallb-bgp-session-status

# Step 3: Verify upstream router has the route
# From upstream router: show bgp <metallb-ip>/32
# Expected: route learned from one (or multiple for ECMP) speaker nodes
```

**Common failure modes:**
1. **Layer 2 mode + VLAN segregation:** MetalLB ARP is L2 — if the upstream router is on a different VLAN/segment, ARP responses don't reach it. Use BGP mode for multi-segment networks.
2. **BGP mode + wrong ASN:** MetalLB's ASN in the ConfigMap must match what the router expects.
3. **Firewall blocking the VIP:** The node iptables or an external firewall may block inbound traffic to the MetalLB IP. Check: is the traffic arriving at the node (tcpdump) but being dropped (iptables -L)?
4. **gratuitous ARP not reaching upstream:** Cloud environments (even private cloud) often block GARP. Use BGP mode in environments with controlled L2 topology.

**Follow-up Questions:**
- How does MetalLB Layer 2 mode handle node failover when the speaker node goes down?
- What is the difference between MetalLB and a cloud provider LoadBalancer?
- How do you configure MetalLB to use multiple address pools for different services?

**Connecting Points:**
- `01-networking-fundamentals/` for ARP and L2 networking
- `04-kubernetes-networking/` for Kubernetes Service types

---

## Q: An EKS pod can't reach RDS. Walk through every layer that could be causing this.

**What the interviewer is testing:** Whether you can systematically eliminate failure points across VPC networking, security, DNS, and TLS.

**Model Answer:**

This is the quintessential cross-domain debugging scenario. Cover every layer:

**Layer 1 — Pod networking (is the pod connected?):**
```bash
kubectl exec -n <ns> <pod> -- ip route show          # does pod have a default route?
kubectl exec -n <ns> <pod> -- ip addr show           # does pod have an IP?
kubectl exec -n <ns> <pod> -- ping <node-ip>         # can pod reach its node?
```

**Layer 2 — DNS (is RDS endpoint resolving?):**
```bash
kubectl exec -n <ns> <pod> -- nslookup <rds-endpoint>
# Expected: returns a private IP (10.x.x.x)
# NXDOMAIN: DNS misconfiguration or wrong endpoint string
# Timeout: CoreDNS not reachable, or conntrack race condition

# Test with ndots bypass (add trailing dot)
kubectl exec <pod> -- nslookup <rds-endpoint>.
```

**Layer 3 — VPC routing (is there a route from the pod's subnet to RDS?):**
```bash
# Check VPC route table for the pod's subnet
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=<pod-subnet-id>"

# Test network reachability
aws ec2 start-network-insights-analysis \
  --network-insights-path-id <path-id>
# Path: pod ENI → RDS endpoint
```

**Layer 4 — Security Groups (is the connection allowed?):**
```bash
# Pod's SG must have outbound rule to RDS port (5432/3306)
aws ec2 describe-security-groups --group-ids <pod-sg-id> \
  --query 'SecurityGroups[].IpPermissionsEgress'

# RDS's SG must have inbound rule from pod's SG (or CIDR)
aws ec2 describe-security-groups --group-ids <rds-sg-id> \
  --query 'SecurityGroups[].IpPermissions'

# Verify SG allows: inbound port 5432 from pod SG (or CIDR)
```

**Layer 5 — NACLs (is subnet-level firewall blocking it?):**
```bash
# Check NACL for pod subnet
aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=<pod-subnet-id>"
# Verify: outbound NACL allows TCP to RDS port
# Verify: inbound NACL allows TCP from ephemeral port range (1024-65535) for return traffic

# Common mistake: NACL allows outbound 5432 but doesn't allow return traffic (stateless!)
```

**Layer 6 — TCP connectivity test:**
```bash
# Test raw TCP connection (bypasses TLS/application errors)
kubectl exec -n <ns> <pod> -- nc -zv <rds-endpoint> 5432
# "Connected" = TCP works; "Connection refused" = wrong port; "Timeout" = SG/NACL/routing
```

**Layer 7 — TLS/RDS authentication:**
```bash
# If TCP connects but application can't authenticate:
kubectl exec -n <ns> <pod> -- openssl s_client -connect <rds-endpoint>:5432 \
  -starttls postgres
# Check: certificate validity, CA bundle in pod, RDS CA certificate

# Check: RDS requires SSL? (PostgreSQL default)
# Connect string: postgresql://user:pass@host/db?sslmode=require  # require TLS
# vs sslmode=disable (for testing)
```

**Summary diagnostic:**
```
1. kubectl exec → ip route/addr → pod has network ✓
2. kubectl exec → nslookup → RDS endpoint resolves to 10.x.x.x ✓
3. VPC route tables → path exists ✓
4. Security Groups → outbound 5432 from pod, inbound 5432 to RDS ✓
5. NACLs → stateless outbound + return traffic ✓
6. nc -zv RDS:5432 → TCP connects ✓
7. psql/application → authentication/TLS
```

**Follow-up Questions:**
- What is VPC Flow Logs and how do you use it to determine if traffic is being rejected at the NACL vs SG level?
- How does RDS IAM authentication differ from password authentication, and how does it interact with VPC networking?
- What is RDS Proxy and how does it affect connection pooling between EKS pods and RDS?

**Connecting Points:**
- `03-cloud-networking/` for VPC Security Groups, NACLs, routing
- `05-network-security/` for TLS client authentication

---

## Q: Kubernetes cluster is experiencing high DNS latency (500ms+ for external domains). What are all possible root causes?

**What the interviewer is testing:** Whether you know the multiple layers that can introduce DNS latency in K8s.

**Model Answer:**

**Root cause 1 — ndots:5 query amplification:**
```bash
kubectl exec <pod> -- cat /etc/resolv.conf
# options ndots:5 + 3 search domains = 4 queries per external name
# At 10ms per query: 40ms added latency (not 500ms — look elsewhere if this severe)
```

**Root cause 2 — conntrack UDP race condition:**
```bash
conntrack -S | grep insert_failed    # on any cluster node
# Incrementing = simultaneous A+AAAA queries cause conntrack insertion race
# Dropped queries → 5s timeout before retry → 500ms+ latency at high rates
```

**Root cause 3 — CoreDNS overloaded:**
```bash
kubectl top pods -n kube-system -l k8s-app=kube-dns
# CPU near limit = CoreDNS is the bottleneck

kubectl logs -n kube-system -l k8s-app=kube-dns | grep -i "error\|timeout"

# Check CoreDNS Prometheus metrics:
# coredns_dns_request_duration_seconds{quantile="0.99"} > 0.5 = 500ms P99 latency
# coredns_forward_request_duration_seconds{quantile="0.99"} = upstream latency
# If forward latency is high → upstream DNS is slow, not CoreDNS itself
```

**Root cause 4 — Upstream DNS slow (corporate DNS, Route 53):**
```bash
# Test upstream resolver directly
dig @<coredns-ip> external.domain.com              # via CoreDNS
dig @<upstream-resolver-ip> external.domain.com    # direct to upstream
# If upstream is slow but CoreDNS is fast → upstream resolver issue
```

**Root cause 5 — NodeLocal DNSCache misconfigured:**
```bash
# If NodeLocal DNSCache is deployed but misconfigured:
# Check if pods are using the link-local address
kubectl exec <pod> -- cat /etc/resolv.conf
# Should show 169.254.20.10 (link-local cache address)
# If showing CoreDNS IP instead: DaemonSet injection failed

# Check cache hit rate
kubectl logs -n kube-system node-local-dns-<hash> | grep -i "cache\|hit"
```

**Root cause 6 — DNS response truncation + TCP retry:**
```bash
# Large DNS responses (many A records) may be truncated in UDP
# Client retries with TCP — adds 10-50ms
dig <domain> | grep "MSG SIZE"    # if > 512 bytes, truncation risk
```

**Recommended fix stack:**
```
1. Deploy NodeLocal DNSCache (eliminates conntrack, reduces CoreDNS load)
2. Set ndots:2 for external-heavy workloads
3. Increase CoreDNS replicas and memory limit if CPU-constrained
4. Enable cache plugin in CoreDNS Corefile (30-300 second TTL for positive responses)
```

**Follow-up Questions:**
- How does NodeLocal DNSCache interact with the conntrack table?
- What Prometheus alerts would you set up for DNS health?
- How do you handle DNS for pods that need to resolve both cluster-internal and external names efficiently?

**Connecting Points:**
- `04-kubernetes-networking/` for CoreDNS and NodeLocal DNSCache
- `02-linux-networking/` for conntrack UDP race condition

---

## Q: Service mesh mTLS is blocking traffic after certificate rotation. How do you diagnose and fix this without restarting all pods?

**What the interviewer is testing:** Whether you understand certificate rotation in a service mesh and the failure modes.

**Model Answer:**

In Istio, certificates are issued by istiod and rotated every 24 hours by default. After rotation, Envoy proxies reload certificates via SDS (Secret Discovery Service) without restart. If this fails, mTLS connections are rejected with TLS handshake errors.

**Diagnosis:**
```bash
# Step 1: Check istiod is healthy
kubectl get pods -n istio-system
kubectl logs -n istio-system istiod-<hash> | grep -i "error\|cert"

# Step 2: Check Envoy sync status
istioctl proxy-status
# "SYNCED" = Envoy has current config from istiod
# "STALE" = Envoy hasn't received updates — investigate connectivity to istiod

# Step 3: Check certificate expiry on affected pods
kubectl exec -n <ns> <pod> -c istio-proxy -- \
  openssl x509 -in /etc/certs/cert-chain.pem -noout -dates -subject
# Look for: expired certificate (notAfter in the past)

# Step 4: Check Envoy TLS handshake errors
kubectl exec -n <ns> <pod> -c istio-proxy -- \
  curl localhost:15000/stats | grep -E "ssl.handshake_error|ssl.connection_error"

# Step 5: Check if peer certificate is trusted
# The new certificate may be signed by a new intermediate CA
# that the peer Envoy hasn't yet loaded
kubectl exec <pod> -c istio-proxy -- \
  curl localhost:15000/config_dump | jq '.configs[] |
  select(.["@type"] | contains("SecretsConfigDump"))'
```

**Root cause — certificate rotation race:**

When istiod rotates its intermediate CA, it issues new leaf certificates. However, there's a brief window where:
- New certificates are signed by the new intermediate CA
- Some Envoy proxies haven't yet loaded the new CA bundle in their trust store
- Result: TLS handshake fails because the peer's new certificate is signed by an unknown CA

**Fixes:**
```bash
# Option 1: Force certificate reload (no pod restart)
# Trigger SDS update by touching the certificate secret
kubectl rollout restart deployment istiod -n istio-system
# istiod will re-push all certificates to all Envoy proxies

# Option 2: If specific pods are stuck, restart them (fastest)
kubectl rollout restart deployment <affected-deployment>

# Option 3: Temporarily set PERMISSIVE mode (traffic continues while debugging)
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: temp-permissive
  namespace: <affected-namespace>
spec:
  mtls:
    mode: PERMISSIVE    # allows both mTLS and plaintext during recovery
EOF
# Fix the rotation issue, then remove this override
```

**Prevention:**
```yaml
# Use a longer certificate TTL to reduce rotation frequency (trade-off: longer revocation window)
# In istiod configuration:
env:
  - name: CITADEL_SELF_SIGNED_CA_CERT_TTL
    value: "87600h"    # 10 years for root; rotate more carefully
  - name: CITADEL_WORKLOAD_CERT_TTL
    value: "24h"       # leaf certs: 24 hours

# Enable cert manager integration for external CA
# (removes dependency on istiod for certificate management)
```

**Follow-up Questions:**
- How do you rotate the Istio root CA without service disruption?
- What is the SPIFFE federation and how does it enable cross-cluster mTLS?
- What is the certificate grace period in Istio and how does it prevent rotation failures?

**Connecting Points:**
- `05-network-security/` for mTLS and PKI management
- `10-interview-prep/05-network-security-qa.md` for certificate expiry prevention

---

## Key Takeaways

- Calico BGP failures follow a clear investigation path: BGP session → route propagation → anti-spoofing → IP forwarding — each step eliminates a class of failure
- Docker DNS SERVFAIL is almost always the conntrack UDP race condition — check `conntrack -S | grep insert_failed` before anything else
- EKS pod → RDS debugging crosses 7 layers: pod networking, DNS, VPC routing, Security Groups, NACLs, TCP, and TLS/auth — work bottom-up
- Kubernetes DNS latency has 6 distinct root causes; NodeLocal DNSCache fixes 3 of them simultaneously (conntrack, CoreDNS load, ndots amplification) — it's the highest-leverage change
- Istio mTLS rotation failures require: identify which certs are expired/mismatched, use PERMISSIVE mode as a safety valve, force SDS reload — never skip the observability step
