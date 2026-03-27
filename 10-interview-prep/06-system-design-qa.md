# System Design — Interview Q&A

## Quick Reference

System design questions test whether you can apply networking and security knowledge at architectural scale. The evaluation criteria: do you frame the problem before diving into solutions, do you name the key trade-offs explicitly, and do you know what changes at 10x scale? A good system design answer takes 15-20 minutes and covers: requirements clarification, high-level design, key technical decisions with justification, failure modes, and scaling considerations. See `08-system-design/` for deeper reading.

---

## Q: Design a multi-region active-active system with RTO < 1 minute.

**What the interviewer is testing:** Whether you understand what "active-active" really requires — not just traffic distribution, but conflict resolution and health detection at sub-minute granularity.

**Model Answer:**

**Requirements clarification:**
- Active-active means BOTH regions serve production traffic simultaneously (vs active-passive where backup is warm but idle)
- RTO < 1 minute means full traffic restoration within 60 seconds of any single-region failure
- This implies: health detection < 30s, traffic failover < 30s

**High-level architecture:**
```
Region US-EAST-1                    Region EU-WEST-1
  ├── EKS cluster                     ├── EKS cluster
  ├── RDS Multi-AZ                    ├── RDS Multi-AZ
  ├── ElastiCache                     ├── ElastiCache
  ├── ALB                             ├── ALB
  └── Route53 Health Check            └── Route53 Health Check
         ↓                                   ↓
         └──────── Route53 (Global) ─────────┘
                   Latency-based routing (active-active)
                   Health check: 10s interval, 3-failure threshold = 30s detection
```

**Traffic distribution — Route 53 latency-based routing:**
Users are routed to the nearest region. Both regions serve traffic concurrently. If one region fails health checks, Route 53 removes it and all traffic goes to the remaining region.

**Data layer — the hard problem:**
Active-active requires resolving write conflicts. Options:

1. **Read/write splitting (most common):**
   - Writes to single "primary" region; reads from either region
   - Cross-region replication: RDS Aurora Global Database (replication lag: ~1-5ms)
   - On primary region failure: promote secondary as new primary (Aurora Global fast failover: < 1 minute)
   - Trade-off: writes still have a single region bottleneck

2. **Multi-master (CRDTs or conflict resolution):**
   - DynamoDB Global Tables: multi-master replication with "last writer wins" conflict resolution
   - Trade-off: data model must handle conflicts (no strongly consistent global writes)
   - Use when: write volume requires horizontal scaling across regions

**Health check and failover sequence:**
```
t=0:    Region failure begins
t=10s:  Route 53 Fast Health Check detects failure (10s interval)
t=30s:  3 consecutive failures → Route53 marks region unhealthy
t=60s:  DNS TTL expires at all resolvers → all traffic to healthy region
Total: ~60s = RTO 1 minute (barely achievable)
```

**To achieve RTO < 1 minute reliably:**
- Use Route 53 Application Recovery Controller (ARC) instead of basic health checks: provides zonal shift and readiness checks
- Set DNS TTL = 60s (minimum practical value before excessive DNS load)
- Pre-warm the standby region: keep it at 50% capacity minimum (double capacity cost)
- Test quarterly: run a Chaos Engineering game day where you fail an entire region and measure actual RTO

**At 10x scale (10x traffic):**
- Route 53 stays the same — it scales automatically
- ALB and EKS autoscale within minutes — pre-warming is critical
- Database: Aurora Global I/O-Optimized mode for high write throughput; DynamoDB Global Tables for massive scale
- Add AWS Global Accelerator (static anycast IPs instead of DNS-based routing — eliminates DNS TTL from failover time)

**Follow-up Questions:**
- How does AWS Global Accelerator reduce failover time below 1 minute?
- What is the difference between ARC routing control and Route 53 health checks?
- How do you handle session state in active-active (user's session was on Region A, now they're routed to Region B)?

**Connecting Points:**
- `08-system-design/` for multi-region architecture patterns
- `03-cloud-networking/` for Route 53 health check timing

---

## Q: Design a secure 3-tier architecture on AWS that is PCI-DSS compliant.

**What the interviewer is testing:** Whether you can translate compliance requirements into specific AWS services and configurations.

**Model Answer:**

**Architecture:**
```
Internet
  ↓
AWS Shield Advanced (DDoS protection, required for PCI)
AWS WAF (OWASP rule set, required for cardholder data)
  ↓
[DMZ / Public Tier]  ← Public Subnet (3 AZs)
  ALB (HTTPS only, TLS 1.2+ enforced)
  Security Group: allow 443 from 0.0.0.0/0, allow 80 redirect only
  NACL: allow 443/80 inbound, allow ephemeral ports outbound
  ↓
[Application Tier]   ← Private Subnet (3 AZs)
  ECS/EKS application containers
  Security Group: allow 8080 from ALB security group ONLY
  NACL: allow from public subnet CIDR, deny all else
  No direct internet access — use VPC Endpoints for AWS services
  ↓
[Data Tier / CDE]    ← Isolated Subnet (3 AZs) — THIS IS THE CDE
  RDS (PostgreSQL Multi-AZ)
  Security Group: allow 5432 from application tier SG ONLY
  NACL: allow only from private subnet CIDR
  KMS encryption at rest (aws:kms CMK, not default AWS key)
  ↓
[Management]
  VPN/Direct Connect for admin access (NO public SSH/RDP)
  Bastion host in DMZ: allow SSH from corporate IP range only
  All admin sessions logged to CloudTrail
```

**PCI-DSS specific controls:**

Requirement 1 (Firewall):
- Every Security Group rule documented and justified
- Review and re-approve all rules every 6 months
- AWS Config rule: any SG change triggers SNS notification for review

Requirement 2 (Secure configuration):
- All instances use hardened AMIs (CIS Benchmark)
- No default passwords, SSH key rotation, no password auth

Requirement 3 (Protect stored card data):
- No card numbers stored in plaintext — tokenization (pay.i/Stripe tokens)
- If stored: AES-256 encryption with KMS CMK
- Key rotation: KMS automatic annual rotation

Requirement 4 (Encrypt transmission):
- TLS 1.2+ enforced on ALB: `ELBSecurityPolicy-TLS13-1-2-Res-2021-06`
- Mutual TLS for internal service-to-service

Requirement 10 (Audit logs):
- CloudTrail enabled for all regions and all management events
- VPC Flow Logs enabled for all CDE subnets
- CloudWatch Logs retention: 1 year minimum
- Log integrity: CloudTrail log file validation enabled

**At 10x scale:**
- Application tier: EKS with cluster autoscaler and HPA
- Database tier: Aurora Serverless v2 auto-scales, or Aurora Global for read scaling
- Cost implication: Multi-AZ + Shield Advanced + WAF + CloudTrail = ~$5,000/month baseline

**Follow-up Questions:**
- What is scope reduction and how does tokenization reduce the PCI audit surface?
- What does PCI-DSS say about container security in Requirement 6?
- How do you demonstrate continuous compliance vs point-in-time audit?

**Connecting Points:**
- `05-network-security/` for PCI-DSS network segmentation requirements
- `06-cloud-security/` for AWS PCI-DSS architecture

---

## Q: How would you implement zero-trust for a 50-service Kubernetes platform?

**What the interviewer is testing:** Whether you understand zero-trust as an operational posture, not just a buzzword.

**Model Answer:**

Zero-trust principle: "Never trust, always verify." In Kubernetes, this means: every service must authenticate, every connection must be authorized, regardless of whether the caller is on the same node/cluster.

**Step 1 — Identity (who is this workload?):**
```yaml
# SPIFFE workload identity via cert-manager + SPIRE
# Every pod gets a certificate with its identity:
# spiffe://cluster.local/ns/payments/sa/payment-service
# This is the workload's identity — not its IP, not its hostname

# In Kubernetes: service accounts + projected tokens
volumes:
  - name: spiffe-svid
    projected:
      sources:
        - serviceAccountToken:
            audience: spiffe://cluster.local
            expirationSeconds: 3600
            path: token
```

**Step 2 — mTLS everywhere (verify the identity):**
```yaml
# Cilium NetworkPolicy with identity-based rules (not IP-based)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: checkout-service   # only checkout can call payments
      toPorts:
        - ports:
            - port: "8443"          # only HTTPS
              protocol: TCP
```

**Step 3 — Default deny all (no implicit trust):**
```yaml
# Deny all traffic by default in every namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}       # matches all pods
  policyTypes:
    - Ingress
    - Egress
  # No ingress or egress rules = deny all
```

**Step 4 — Authorization policy (what is this workload allowed to do?):**
```yaml
# Istio AuthorizationPolicy: payment-service can only GET /payments/*, not POST /admin/*
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
spec:
  selector:
    matchLabels:
      app: payment-service
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/checkout/sa/checkout-service"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/payments/*"]
```

**Step 5 — Observability (verify zero-trust is working):**
```bash
# Hubble (Cilium): see all allowed and denied flows
hubble observe --type drop --namespace payments
# Every denied connection is logged with source, destination, reason

# Istio metrics
# istio_requests_total{response_code="403",source_workload="unknown"}
# High 403s from unknown sources: someone is trying to call without mTLS
```

**Migration strategy (can't do all 50 services at once):**
```
Phase 1: Deploy Cilium/Istio; set mode=PERMISSIVE (mTLS optional, traffic observed)
Phase 2: Enable Hubble/observability; map all actual service-to-service traffic
Phase 3: Generate NetworkPolicy from observed flows (Cilium auto-generate)
Phase 4: Apply policies in AUDIT mode, check for false positives
Phase 5: Enable STRICT mode one namespace at a time
Phase 6: Remove all IP-based rules; enforce identity-based only
```

**At 10x scale (500 services):**
- Cilium hub scales better than Istio for policy enforcement at this count
- Replace istiod with cert-manager + SPIRE for lighter-weight certificate management
- Consider Cilium's identity model (label-based, not certificate-based) for internal cluster traffic

**Follow-up Questions:**
- How do you handle zero-trust for a service that calls external APIs (not in the cluster)?
- What is the performance overhead of mTLS per connection?
- How does zero-trust interact with debugging tools that inspect traffic?

**Connecting Points:**
- `04-kubernetes-networking/` for Istio mTLS and Cilium policy
- `10-interview-prep/07-cross-domain-scenarios.md` for mTLS implementation scenario

---

## Q: Design the networking for an EKS cluster — CNI choice, ingress, egress, observability.

**What the interviewer is testing:** Whether you can make opinionated CNI choices and justify them.

**Model Answer:**

**CNI choice decision tree:**

```
Cluster size < 2,000 pods, simple networking needs?
  → AWS VPC CNI (default) — simplest, no overlay, pod IPs routable in VPC

Cluster size > 5,000 pods OR need NetworkPolicy at scale OR need eBPF observability?
  → Cilium — eBPF, replaces kube-proxy, best network policy performance

Need Calico-compatible BGP routing to on-premises physical network?
  → Calico BGP mode

Need Windows node support?
  → AWS VPC CNI (only CNI with Windows EKS support)
```

**For a production EKS cluster (recommended: Cilium):**
```yaml
# Install Cilium with kube-proxy replacement
# eni mode: pods get VPC IPs (native routing, no overlay)
helm install cilium cilium/cilium --version 1.15.x \
  --set eni.enabled=true \
  --set ipam.mode=eni \
  --set egressMasqueradeInterfaces=eth0 \
  --set tunnel=disabled \            # native routing (no VXLAN overhead)
  --set kubeProxyReplacement=true    # replace kube-proxy entirely
```

**Ingress design:**
```
External traffic:
  Route53 → AWS ALB (Ingress Class: alb, uses AWS Load Balancer Controller)
    → EKS Service (ClusterIP)
    → Pod

Internal/API traffic:
  AWS API Gateway (if external) → VPC Link → NLB → EKS Service

gRPC traffic:
  NLB (Layer 4, no HTTP2 termination issues) → EKS pods
```

```yaml
# AWS Load Balancer Controller Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip    # pod IPs directly
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-Res-2021-06
```

**Egress design:**
```
Pod → NAT Gateway (for general internet) — simple but adds cost per connection
Pod → VPC Endpoint (for AWS services — S3, ECR, DynamoDB) — no NAT Gateway, lower latency
Pod → Egress Gateway (Cilium) — for compliance: route all pod egress through specific IP
```

```bash
# Cilium Egress Gateway: all pods in "payments" namespace exit through specific IP
# This provides a predictable egress IP for allowlisting in external firewalls
```

**Observability:**
```bash
# Hubble UI: real-time flow visualization
cilium hubble enable
cilium hubble ui

# Prometheus metrics
# Container metrics: kube-state-metrics, metrics-server
# Network metrics: cilium_drop_count, cilium_forward_bytes, coredns_dns_request_duration_seconds

# VPC Flow Logs for all ENIs (compliance, security analysis)
# CloudWatch Container Insights for EKS
```

**Follow-up Questions:**
- How do you handle EKS cluster upgrades without downtime to CNI configuration?
- What is Cilium's node IPAM mode vs ENI mode and when do you choose?
- How does the AWS Load Balancer Controller interact with ExternalDNS?

**Connecting Points:**
- `04-kubernetes-networking/` for EKS CNI configuration
- `03-cloud-networking/` for VPC Endpoints and NAT Gateway

---

## Q: How would you migrate from a monolithic VPC to micro-segmented VPCs without downtime?

**What the interviewer is testing:** Whether you can design complex migrations with safety properties — reversibility, incremental validation, zero downtime.

**Model Answer:**

**Current state:** All services in one large VPC (10.0.0.0/16). All services can talk to each other. No meaningful segmentation.

**Target state:** Each service tier in its own VPC, connected via Transit Gateway, with explicit security group and NetworkPolicy controls.

**Migration principles:**
1. Never cut over everything at once — each phase is independently reversible
2. Run in parallel, not in series (new VPC exists before old is decommissioned)
3. Network connectivity is established before services migrate (strangler fig pattern)
4. Validate each phase before proceeding

**Phase 1 — Establish connectivity (no disruption):**
```bash
# Create segmented VPCs
aws ec2 create-vpc --cidr-block 10.1.0.0/16  # payment-vpc
aws ec2 create-vpc --cidr-block 10.2.0.0/16  # inventory-vpc
aws ec2 create-vpc --cidr-block 10.3.0.0/16  # frontend-vpc

# Connect to Transit Gateway
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-XXXXXXXX \
  --vpc-id vpc-payment \
  --subnet-ids subnet-XXXXXXXX

# Verify connectivity between new VPCs (no services yet)
# Test from a temporary EC2 in payment-vpc to inventory-vpc
```

**Phase 2 — Mirror traffic (validate networking):**
```bash
# Deploy new service in new VPC while old service still serves traffic
# Use weighted DNS or ALB routing to send 1% of traffic to new VPC
# Monitor error rates — if >0 errors: rollback (just shift weight back to 0%)
```

**Phase 3 — Service migration (one service at a time):**
```bash
# Migrate payment-service:
# 1. Deploy payment-service in payment-vpc
# 2. Update DNS/load balancer to point to new VPC (Route 53 weighted: old 100%, new 0%)
# 3. Shift 5% traffic to new VPC
# 4. Monitor for 24 hours
# 5. Shift to 50%, 100%
# 6. Decommission old payment-service instances
# Rollback: shift weight to 0% for new VPC

# For each service migration:
# - Update any hardcoded IPs to DNS names (required pre-work)
# - Update Security Groups to allow cross-VPC traffic
# - Validate data tier access (RDS Security Group update)
```

**Phase 4 — Tighten security (after migration complete):**
```bash
# Enable strict deny-all in old VPC
# Verify no traffic flows through old VPC
# Decommission old VPC

# Add explicit allow rules in new VPCs
# Remove overly broad rules (the migration may have temporarily opened wide rules)
```

**Risk mitigation:**
- Blue-green deployment at each phase: old and new coexist until validation complete
- Rollback plan at each phase: must be < 5 minutes
- Data migration is the hardest part: database cannot be in two VPCs simultaneously — use database proxy (RDS Proxy) accessible from both VPCs during transition

**Follow-up Questions:**
- How do you handle a service that uses hardcoded IPs instead of DNS names?
- What is the role of AWS Network Firewall in the segmented architecture?
- How does this migration change if you're doing it across cloud providers?

**Connecting Points:**
- `03-cloud-networking/` for Transit Gateway architecture
- `08-system-design/` for migration patterns

---

## Key Takeaways

- Active-active RTO < 1 minute requires pre-warming both regions, Route 53 Fast Health Checks (10s interval), and 60s DNS TTL — DNS propagation is the limiting factor
- PCI-DSS compliance translates to specific AWS controls: Shield Advanced, WAF, private subnets, Security Groups with justified rules, CloudTrail for all management events
- Zero-trust migration is phased: observe first (PERMISSIVE mode), generate policies from observed traffic, then enforce — never go straight to STRICT without observation data
- EKS CNI choice is an inflection point: AWS VPC CNI for simplicity, Cilium for scale (>5K pods) and eBPF observability
- VPC migration must be incremental and reversible at each step — weighted DNS routing enables 1% canary validation before full cutover
