# Cloud Networking — Interview Q&A

## Quick Reference

Cloud networking interviews test whether you understand that AWS VPCs, Azure VNets, and GCP VPCs are software-defined overlays with specific constraints that don't exist in physical networks. The most common failure patterns are: treating cloud networking like on-premises networking (wrong mental model), not knowing the service limits (NAT Gateway 55K connections, VPC peering 125 peers, etc.), and not understanding the stateful vs stateless distinction for Security Groups vs NACLs. See `03-cloud-networking/` for deeper reading.

---

## Q: VPC Peering vs Transit Gateway vs PrivateLink — when do you use each and what are the hard limits?

**What the interviewer is testing:** Whether you understand the architectural trade-offs, not just that all three "connect VPCs."

**Model Answer:**

**VPC Peering:**
- Direct, private, non-transitive connection between two VPCs
- Non-transitive: if VPC A peers with B, and B peers with C, A cannot reach C through B
- No bandwidth limit, no gateway to manage
- Cost: $0.01/GB data transfer (cross-AZ or cross-region)
- Limit: 125 peering connections per VPC (soft limit, can request increase)
- **Use when:** Small number of VPCs (< 10) that need full connectivity, or hub-and-spoke isn't needed

**Transit Gateway (TGW):**
- Regional hub router connecting multiple VPCs, VPNs, and Direct Connect
- Transitive: all attached VPCs can route to each other (unless blocked by route tables)
- Cost: $0.05/attachment/hour + $0.02/GB data transfer — gets expensive
- Throughput: 50 Gbps per VPC attachment
- Supports BGP for dynamic routing from on-premises
- **Use when:** 10+ VPCs, hub-and-spoke architecture, need to route to on-premises

**PrivateLink (VPC Endpoint Services):**
- Creates an elastic network interface (ENI) in the consumer VPC that routes to a service in the provider VPC via AWS private network
- Works even with overlapping CIDRs (the ENI gets an IP from the consumer VPC)
- Unidirectional: consumer can reach provider service, not the other way around
- Requires an NLB in the provider VPC
- Cost: $0.01/hour per AZ + $0.01/GB data transfer
- **Use when:** Exposing a specific service to many consumers (SaaS product), overlapping CIDRs, zero-trust service isolation

**Decision tree:**
```
Need to expose a specific service to consumers with unknown/overlapping CIDRs?
  → PrivateLink

Need full network connectivity between few VPCs in same region?
  → VPC Peering

Need hub-and-spoke, VPN integration, or many VPCs?
  → Transit Gateway

Need full mesh connectivity at 100+ VPCs?
  → Transit Gateway (peering mesh is n(n-1)/2 connections — 100 VPCs = 4,950 peers, exceeds limit)
```

**Follow-up Questions:**
- How do you handle PrivateLink in a multi-region architecture?
- What is TGW multicast support and when is it used?
- Why can't you do transitive routing with VPC peering?

**Connecting Points:**
- `03-cloud-networking/` for detailed TGW route table configuration
- `10-interview-prep/07-cross-domain-scenarios.md` for CIDR overlap PrivateLink scenario

---

## Q: Why do you need explicit route table entries for VPC peering? Why isn't it automatic?

**What the interviewer is testing:** Whether you understand VPC routing is software-defined and intentionally explicit, not automatic like ARP learning.

**Model Answer:**

VPC peering creates a **network path** (a private link between VPCs), but it does NOT add routes. Routes are always explicit in AWS VPC — there is no automatic route learning like in traditional LANs (no ARP flooding, no spanning tree).

**Why explicit is correct:**

1. **Security by default:** Automatic routes would mean any VPC you peer with immediately has routes to all your subnets. Explicit routes let you control which subnets in your VPC can communicate with which subnets in the peer VPC.

2. **Non-transitive enforcement:** VPC A peers with B and C. If routes were auto-added, B might accidentally be able to route to C through A (which isn't allowed but could happen with auto-learned routes). Explicit routes make you deliberately express each allowed path.

3. **Subnet-level granularity:** You can add a route for only the peer VPC's private subnet, not its public subnet — even if both are in the same peered VPC.

**After creating a peering, you must:**
```bash
# In VPC A's route table, add route for VPC B's CIDR
aws ec2 create-route \
  --route-table-id rtb-XXXXXXXX \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-XXXXXXXX

# In VPC B's route table, add route for VPC A's CIDR (bidirectional)
aws ec2 create-route \
  --route-table-id rtb-YYYYYYYY \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-XXXXXXXX
```

**Common mistakes:**
- Adding route only one direction — traffic flows A→B but not B→A
- Adding route but forgetting Security Groups — route exists but SG blocks the connection
- Expecting routes to update automatically when subnets are added to a VPC

**Follow-up Questions:**
- What happens to existing connections if you delete a VPC peering route table entry?
- How does TGW differ — does it add routes automatically?
- What is the maximum number of routes in an AWS VPC route table?

**Connecting Points:**
- `03-cloud-networking/` for VPC route table architecture
- `10-interview-prep/07-cross-domain-scenarios.md` Q3 for overlapping CIDR scenarios

---

## Q: You have Direct Connect for primary connectivity and a VPN as backup. How does BGP route preference control failover, and what are the timing considerations?

**What the interviewer is testing:** Whether you understand BGP metrics in AWS context and can reason about failover timing.

**Model Answer:**

AWS uses BGP for both Direct Connect (via Virtual Private Gateway or TGW) and Site-to-Site VPN. Route preference is controlled through standard BGP attributes:

**AWS BGP path selection order (for routes learned from customer):**
1. Longest prefix match (most specific route always wins)
2. **Local Preference** — AWS sets this on routes learned from DX vs VPN: DX gets higher LOCAL_PREF by default
3. **AS_PATH prepending** — you can influence AWS's preference by prepending ASes to VPN routes

**Recommended architecture for DX + VPN failover:**
```
Primary path: Direct Connect
  Advertise: 10.0.0.0/16 via DX (no AS_PATH prepend)
  AWS sees: short AS_PATH, higher LOCAL_PREF from DX

Backup path: VPN
  Advertise: 10.0.0.0/16 via VPN (with 2-3 AS_PATH prepends)
  aws sees: longer AS_PATH, lower LOCAL_PREF from VPN
  Only selected when DX route is unavailable
```

**For outbound from AWS (AWS → your network):**
```
AWS advertises VPC CIDR to both DX and VPN customer routers.
You control path preference with:
- MED (Multi-Exit Discriminator): lower MED = preferred
- LOCAL_PREF on your side: set higher on DX-learned routes
```

**Failover timing:**
- BGP hold timer: default 90 seconds (AWS to customer). DX failure detected in up to 90s.
- With BFD (Bidirectional Forwarding Detection): failure detected in < 1 second on DX
```bash
# Enable BFD on DX virtual interface (AWS Console or API)
# BFD min-interval: 300ms, multiplier: 3 = 900ms detection time
```
- VPN path convergence: 60-90 seconds after DX BGP session drops
- Total failover time without BFD: 90s + 60s = ~3 minutes
- With BFD: < 2 seconds

**Follow-up Questions:**
- How do you test DX failover without actually failing the connection?
- What is the difference between Active/Active and Active/Passive DX configuration?
- How does TGW ECMP work with multiple DX connections?

**Connecting Points:**
- `03-cloud-networking/` for DX and VPN architecture
- `01-networking-fundamentals/` for BGP path selection details

---

## Q: EKS pods are getting EADDRNOTAVAIL errors connecting to RDS. You suspect pod IP exhaustion. Walk through diagnosis and WARM_IP_TARGET tuning.

**What the interviewer is testing:** Whether you understand AWS VPC CNI's IP pre-allocation model and its interaction with subnet sizing.

**Model Answer:**

AWS VPC CNI pre-allocates IPs on each node. When a pod is scheduled, it gets one of these pre-allocated IPs immediately. The CNI then requests more IPs from the subnet to refill the warm pool.

**The problem:** If the subnet runs out of IPs (subnet exhaustion), the CNI cannot allocate IPs for new pods. EADDRNOTAVAIL is the error when no local IP is available to bind a source address.

**Diagnosis:**
```bash
# Check how many IPs are available in the node's ENIs
kubectl describe node <node-name> | grep -A 10 "Allocatable"
# Look for "vpc.amazonaws.com/pod-eni" capacity

# Check CNI logs for IP exhaustion
kubectl logs -n kube-system aws-node-<pod-id> | grep -i "exhausted\|failed\|no.*ip"

# Check actual subnet IP usage
aws ec2 describe-subnets --subnet-ids <subnet-id> \
  --query 'Subnets[].{Available:AvailableIpAddressCount, CIDR:CidrBlock}'

# Check current pod IP allocations
kubectl get pods -A -o wide | grep <node-name> | wc -l    # pods on this node
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=<instance-id>" \
  --query 'NetworkInterfaces[].PrivateIpAddresses[].PrivateIpAddress'
```

**Understanding WARM_IP_TARGET:**

```
WARM_IP_TARGET: minimum IPs to keep "warm" (pre-allocated) on each node
MINIMUM_IP_TARGET: minimum IPs regardless of pod count
WARM_PREFIX_TARGET: for prefix delegation mode (/28 blocks)
```

```bash
# Current settings in aws-node DaemonSet
kubectl describe daemonset -n kube-system aws-node | grep -A 5 "WARM_IP"

# Edit to tune
kubectl edit daemonset -n kube-system aws-node
# Set WARM_IP_TARGET=3 (keep 3 warm IPs per node)
# Set MINIMUM_IP_TARGET=10 (always keep 10 IPs per node)
```

**The tension:**
- High WARM_IP_TARGET: fast pod startup (no waiting for IP allocation), but wastes IPs when nodes are under-utilized — accelerates subnet exhaustion
- Low WARM_IP_TARGET: slow pod startup (wait for IP allocation from ENI), but conserves IPs

**Solutions when subnet is exhausted:**

1. Add secondary CIDR to VPC:
```bash
aws ec2 associate-vpc-cidr-block --vpc-id vpc-XXXXXXXX --cidr-block 100.64.0.0/16
aws ec2 create-subnet --vpc-id vpc-XXXXXXXX --cidr-block 100.64.0.0/18 --availability-zone us-east-1a
```

2. Enable prefix delegation (assigns /28 prefixes instead of individual IPs — each /28 = 16 IPs from one ENI slot):
```bash
kubectl set env daemonset -n kube-system aws-node ENABLE_PREFIX_DELEGATION=true
```

3. Switch to CNI with separate pod CIDR (Cilium with overlay mode or AWS CNI Custom Networking):
```bash
# Custom Networking: pod IPs come from a different subnet than node IPs
kubectl set env daemonset -n kube-system aws-node AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
```

**Follow-up Questions:**
- How does node instance type affect max pod count in EKS?
- What is the formula for maximum pods per node with AWS VPC CNI?
- How does prefix delegation change the subnet size requirements?

**Connecting Points:**
- `04-kubernetes-networking/` for EKS CNI configuration
- `10-interview-prep/04-kubernetes-networking-qa.md` for CNI scaling

---

## Q: Security Group vs NACL — what is the fundamental difference and when does it matter?

**What the interviewer is testing:** Whether you understand stateful vs stateless as a fundamental property, not just a checkbox.

**Model Answer:**

The fundamental difference is **stateful vs stateless**, which determines how return traffic is handled.

**Security Groups (stateful):**
- Stateful: if you allow inbound TCP on port 443, the return traffic (SYN-ACK, established connection data) is AUTOMATICALLY allowed outbound — no return rule needed.
- Instance-level: attached to ENIs, applies to all traffic to/from that ENI
- Allow-only: no explicit DENY rules, only ALLOW. Traffic not matching any rule is implicitly denied.
- Evaluation: ALL rules evaluated, most permissive match wins

**NACLs (stateless):**
- Stateless: you must explicitly allow BOTH inbound AND return outbound traffic
- Subnet-level: applies to all traffic entering/leaving the subnet
- Allow and Deny: can explicitly deny specific IPs/ports
- Evaluation: rules processed in number order, first match wins (like iptables)

**The return traffic problem with NACLs:**

A client connects from ephemeral port 52341 to your server on port 443. The NACL inbound rule allows TCP port 443. But the return traffic goes from the server to the client's ephemeral port (1024-65535). You must have an outbound NACL rule allowing TCP to ports 1024-65535 (or the specific client IP):

```
Inbound NACL rule:  Allow TCP from 0.0.0.0/0 to port 443  ✓
Outbound NACL rule: Allow TCP from 0.0.0.0/0 to ports 1024-65535  ✓ (required!)
# Without the outbound rule, replies are dropped silently
```

This is the most common NACL misconfiguration: blocking all outbound traffic and wondering why established connections fail.

**When NACLs are useful (despite stateless complexity):**
1. **Emergency IP block:** Add a DENY rule to block an attacker's IP across ALL instances in a subnet — Security Groups would require updating every SG
2. **Defense in depth:** An extra layer of protection for compliance (NACLs are subnet-level, hard to accidentally remove)
3. **Blocking by subnet:** Control which subnets can communicate (tier separation) without per-instance SG rules

**Debugging NACL issues:**
```bash
# Enable VPC Flow Logs and look for REJECT on the subnet boundary
aws ec2 create-flow-logs --resource-type Subnet \
  --resource-ids subnet-XXXXXXXX \
  --traffic-type REJECT \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-vpc-flow-logs

# Filter for REJECT action (NACLs and SGs both show as REJECT in flow logs)
# If you see REJECT on outbound: likely NACL blocking return traffic
```

**Follow-up Questions:**
- What is VPC Flow Logs and what does it NOT capture?
- How does a Security Group reference another Security Group (and why is this useful)?
- What happens to existing connections when you modify a Security Group vs a NACL?

**Connecting Points:**
- `05-network-security/` for Security Group design patterns
- `06-cloud-security/` for PCI-DSS compliant SG configuration

---

## Q: Route 53 health check failover — how does it work, what triggers it, and what are the timing gaps?

**What the interviewer is testing:** Whether you understand DNS-based failover as a distributed system with TTL, health check interval, and DNS propagation delays.

**Model Answer:**

Route 53 health checks poll your endpoint from ~15 global checker locations. When a threshold of checkers fail, R53 marks the record unhealthy and stops returning it in DNS responses. This triggers DNS-based failover.

**Default timing:**
```
Health check interval: 30 seconds (Fast: 10 seconds)
Failure threshold: 3 consecutive failures
Time to mark unhealthy: 30s × 3 = 90 seconds (Fast: 10s × 3 = 30 seconds)
DNS TTL for failover records: typically 60 seconds
DNS propagation to resolvers: 60 seconds + resolver TTL caching
```

**Total failover time (default):**
```
90s (health check) + 60s (TTL) + caching latency = 2-4 minutes
```

**With Fast Health Checks:**
```
30s (health check) + 60s (TTL) = 90 seconds
```

**What triggers health check failure:**
- TCP connection failure (no response within 10-second timeout)
- HTTP 2xx not returned (4xx, 5xx, timeout all count as failure)
- HTTPS certificate validation failure (optional)
- String matching failure (body check)

**What it does NOT detect:**
- Application returning 200 but serving errors (unless you configure string matching)
- Slow responses that don't timeout (a service responding in 9.9 seconds looks healthy)
- Internal service dependencies failing (your app is up but database is down — unless your health check checks the DB)

**Failback is also slow:**
- Healthy threshold: 3 consecutive successes = 90 seconds (same as failure)
- TTL: 60 seconds
- Total failback: 2-4 minutes of healthy region being unused

**Recommendations for lower RTO:**
```bash
# Use Fast Health Checks ($1/month per check)
# Set TTL to 60 seconds (lowest that avoids excessive DNS load)
# Use health check alarm (CloudWatch) to trigger other remediation
# Design for graceful degradation: primary down → secondary handles all traffic
# vs hard failover → minutes of errors
```

**Follow-up Questions:**
- What is the difference between active-active and active-passive Route 53 configurations?
- How does latency-based routing interact with health checks?
- What is Route 53 Application Recovery Controller and how does it improve on basic health checks?

**Connecting Points:**
- `08-system-design/` for multi-region active-active design
- `10-interview-prep/06-system-design-qa.md` for RTO < 1 minute system design

---

## Q: AWS NAT Gateway has a per-destination connection limit of 55,000. How does this cause problems in production and how do you architect around it?

**What the interviewer is testing:** Whether you know the specific limits and can reason about the architecture implications.

**Model Answer:**

AWS NAT Gateway limits simultaneous connections to a single destination IP:port combination to 55,000 (5-tuple: srcIP, srcPort, dstIP, dstPort, protocol — but srcIP is always the NAT GW's IP, so effectively limited by srcPort range).

**When this causes problems:**
- Many EC2 instances or pods sharing one NAT Gateway connecting to the same external endpoint
- Scenarios: all pods connecting to a shared database endpoint, all nodes pulling from the same container registry, bulk data transfer to a single S3 endpoint (before VPC Endpoints)

**Symptoms:**
- Connection errors to a specific external IP when traffic is high
- `EADDRNOTAVAIL` or connection timeouts in application logs
- NAT Gateway metrics: `ErrorPortAllocation` counter incrementing

```bash
# Check NAT Gateway metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name ErrorPortAllocation \
  --dimensions Name=NatGatewayId,Value=nat-XXXXXXXX \
  --start-time 2026-01-01T00:00:00Z \
  --end-time 2026-01-02T00:00:00Z \
  --period 60 \
  --statistics Sum
```

**Architectural solutions:**

1. **VPC Endpoints (most effective for AWS services):**
   - S3, DynamoDB, and 100+ AWS services: use Gateway or Interface VPC Endpoints
   - Traffic stays within AWS network, bypasses NAT Gateway entirely
   - No connection limit, lower latency, lower cost
   ```bash
   aws ec2 create-vpc-endpoint --vpc-id vpc-XXX --service-name com.amazonaws.us-east-1.s3 --route-table-ids rtb-XXX
   ```

2. **Multiple NAT Gateways with traffic distribution:**
   - Each subnet in different AZ gets its own NAT Gateway
   - 3 AZs × 55,000 = 165,000 connections to same destination
   - Requires distributing pods/instances across subnets/AZs

3. **Persistent connections (HTTP/2, gRPC, connection pooling):**
   - Reduces connection churn — fewer simultaneous connections to same destination
   - A connection pool of 100 persistent connections uses 100 NAT ports vs 10,000 short-lived connections using up to 10,000

4. **PrivateLink for internal services:**
   - If the "external" destination is actually another VPC, use PrivateLink — no NAT Gateway involved

**Follow-up Questions:**
- What is the NAT Gateway bandwidth limit and how do you exceed it?
- How does NAT Gateway differ from a self-managed NAT instance for this problem?
- What metrics would you alert on to detect approaching the connection limit?

**Connecting Points:**
- `03-cloud-networking/` for NAT Gateway architecture
- `10-interview-prep/01-networking-fundamentals-qa.md` for port exhaustion fundamentals

---

## Q: Azure CNI vs Azure CNI Overlay — what is the difference and how would you migrate?

**What the interviewer is testing:** Whether you understand Azure's specific IP exhaustion problem and the overlay trade-off.

**Model Answer:**

**Azure CNI (legacy):**
- Pods get IPs directly from the Azure VNet subnet
- Pod IPs are routable within the VNet and to on-premises via ExpressRoute
- Problem: each pod consumes a VNet IP — a 512-node cluster × 30 pods/node = 15,360 VNet IPs
- A /24 subnet (251 usable) runs out at ~8 nodes
- Forces large subnets (/16+) and wastes IPs on idle nodes

**Azure CNI Overlay:**
- Pods get IPs from a private CIDR (typically 192.168.0.0/16) that is NOT routable in the VNet
- Node IPs remain VNet-native; pod IPs are overlay-only
- Eliminates IP exhaustion for large clusters
- Traffic: pod → node SNAT → VNet; external traffic is NATed at the node level
- Pods are NOT directly reachable from outside the cluster without a Load Balancer

**Trade-offs:**

| Aspect | Azure CNI | Azure CNI Overlay |
|--------|-----------|------------------|
| IP efficiency | Poor (VNet IPs) | Good (overlay IPs) |
| Pod reachability from VNet | Direct (pod IP visible) | No (SNAT required) |
| On-prem access to pods | Direct via ExpressRoute | Via LB/Service only |
| Network Policy | Azure NPM (iptables) | Cilium recommended |
| Latency | Lower (no encap) | Slightly higher (encap) |

**Migration strategy:**

Zero-downtime migration from Azure CNI to Azure CNI Overlay requires a new cluster (in-place CNI migration is not supported by Azure):

```
Phase 1: Deploy new AKS cluster with Azure CNI Overlay
  - Same VNet, new node subnet
  - Deploy all workloads to new cluster
  - Validate functionality (especially workloads that rely on pod IPs being routable)

Phase 2: Traffic migration
  - Point DNS/load balancers to new cluster
  - Monitor error rates

Phase 3: Decommission old cluster
```

For workloads relying on pod IP reachability (e.g., on-premises services connecting directly to pod IPs): must migrate to Service-based access before switching CNI.

**Follow-up Questions:**
- How does Cilium's overlay mode compare to Azure CNI Overlay?
- What is Azure NPM and how does it implement NetworkPolicy?
- How does ExpressRoute route advertisement change between Azure CNI and Overlay?

**Connecting Points:**
- `03-cloud-networking/` for Azure networking architecture
- `04-kubernetes-networking/` for CNI plugin comparison

---

## Key Takeaways

- VPC Peering is non-transitive and has no automatic routes — both are intentional design decisions for security and control
- Security Groups are stateful (return traffic is automatic); NACLs are stateless (you must allow ephemeral port range outbound — the most common NACL mistake)
- Route 53 health check failover takes 2-4 minutes by default; Fast Health Checks + 60s TTL reduces this to ~90 seconds
- AWS NAT Gateway's 55K per-destination limit is solved by VPC Endpoints (best for AWS services), not by larger NAT Gateways
- Azure CNI Overlay trades pod IP routability for IP efficiency — migration requires a new cluster, not an in-place change
