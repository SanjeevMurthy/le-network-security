# Security & Cloud Infrastructure Technology Cheatsheet

> Comprehensive reference of tools, technologies, and services extracted from cloud networking, cloud security, and system design documentation. Each entry includes: what it is, what problem it solves, how it is used, and which scenarios reference it.

---

## Table of Contents

- [1. Networking & Connectivity](#1-networking--connectivity)
  - [AWS VPC & Networking](#aws-vpc--networking)
  - [Azure Networking](#azure-networking)
  - [Cross-Cloud & Multi-Cloud Connectivity](#cross-cloud--multi-cloud-connectivity)
  - [DNS & Traffic Routing](#dns--traffic-routing)
  - [CDN & Edge Networking](#cdn--edge-networking)
  - [Load Balancing](#load-balancing)
  - [Service Mesh & Service-to-Service Communication](#service-mesh--service-to-service-communication)
- [2. Identity & Access Management](#2-identity--access-management)
  - [AWS IAM & Identity](#aws-iam--identity)
  - [Azure Identity](#azure-identity)
  - [Workload Identity & Federation](#workload-identity--federation)
- [3. Security & Threat Detection](#3-security--threat-detection)
  - [AWS Security Services](#aws-security-services)
  - [Azure Security Services](#azure-security-services)
  - [Web Application Security](#web-application-security)
  - [DDoS Protection](#ddos-protection)
- [4. Container & Kubernetes Security](#4-container--kubernetes-security)
  - [Image Security](#image-security)
  - [Runtime Security](#runtime-security)
  - [Admission Control & Policy Enforcement](#admission-control--policy-enforcement)
  - [Pod Security](#pod-security)
- [5. Supply Chain Security](#5-supply-chain-security)
  - [Signing & Provenance](#signing--provenance)
  - [SBOM & Dependency Management](#sbom--dependency-management)
- [6. Secrets Management & Encryption](#6-secrets-management--encryption)
  - [Secrets Providers](#secrets-providers)
  - [Kubernetes Secrets Delivery](#kubernetes-secrets-delivery)
  - [Encryption & Key Management](#encryption--key-management)
- [7. Observability & Monitoring](#7-observability--monitoring)
  - [Logging & Auditing](#logging--auditing)
  - [Metrics & Tracing](#metrics--tracing)
  - [SIEM & SOAR](#siem--soar)
- [8. Platform Engineering & IaC](#8-platform-engineering--iac)
  - [GitOps & Deployment](#gitops--deployment)
  - [Infrastructure as Code](#infrastructure-as-code)
  - [Data Replication & Databases](#data-replication--databases)
- [9. API Gateway & Traffic Management](#9-api-gateway--traffic-management)
- [10. Compliance & Governance](#10-compliance--governance)

---

## 1. Networking & Connectivity

### AWS VPC & Networking

#### VPC (Virtual Private Cloud)

| Attribute | Detail |
|---|---|
| **What** | Isolated virtual network in AWS with custom CIDR blocks (e.g., `10.0.0.0/16`), subnets, route tables, and gateways.<br/><br/>A VPC spans all Availability Zones in a region and serves as the foundational networking construct for all AWS resources. Supports VPC peering for cross-VPC connectivity, VPN/Direct Connect for hybrid access, and flow logs for traffic visibility. Example: a production VPC with public subnets (ALB), private subnets (app tier), and isolated subnets (RDS) across 3 AZs. |
| **Problem Solved** | Network isolation for workloads; prevents uncontrolled access between environments |
| **Usage** | Define CIDR (e.g., `10.0.0.0/16`), create public/private subnets across AZs, attach IGW or NAT GW |
| **Scenarios** | Secure multi-tier architecture (01), PCI-DSS CDE isolation (08), multi-region failover (03) |

#### Security Groups

| Attribute | Detail |
|---|---|
| **What** | Stateful instance-level firewalls with ALLOW-only rules (no explicit DENY); return traffic is automatically permitted regardless of outbound rules.<br/><br/>Security Groups are referenced by ID rather than IP, enabling dynamic tier chaining — e.g., `App-SG` allows port 8080 only from `ALB-SG`, so any new ALB instance is automatically trusted. Up to 5 SGs per ENI with rules evaluated as a union (any match = allow). Example: `DB-SG` allows `5432/tcp` only from `App-SG`, ensuring no direct access from the internet tier. |
| **Problem Solved** | Network-level access control without hardcoded IPs; auto-scales with ASG membership |
| **Usage** | `ALB-SG → App-SG (allow 8080 from ALB-SG) → DB-SG (allow 5432 from App-SG)` |
| **Scenarios** | Three-tier SG chaining (02-aws-security), PCI CDE segmentation (08-compliance), HA networking (05) |

#### NACLs (Network Access Control Lists)

| Attribute | Detail |
|---|---|
| **What** | Stateless subnet-level ACLs with explicit ALLOW and DENY rules, evaluated in ascending numbered order (lowest number wins).<br/><br/>Unlike Security Groups, NACLs are stateless — you must define both inbound and outbound rules (including ephemeral port ranges 1024-65535 for return traffic). Rules are processed in order; first match applies and remaining rules are skipped. Example: Rule 100 DENY `192.168.1.50/32` on all ports blocks a specific attacker IP, even if Rule 200 ALLOWs the broader subnet. |
| **Problem Solved** | Subnet-level defense-in-depth; IP-based blocking during incident response |
| **Usage** | `aws ec2 create-network-acl-entry --rule-action deny --cidr-block "$ATTACKER_IP/32"` |
| **Scenarios** | S3 exfiltration containment (02-aws-security), CDE segmentation test (08-compliance) |

#### VPC Endpoints (Gateway & Interface / PrivateLink)

| Attribute | Detail |
|---|---|
| **What** | Gateway Endpoints route S3/DynamoDB traffic via AWS backbone (free); Interface Endpoints create ENIs with private IPs for 100+ services |
| **Problem Solved** | Keeps AWS API traffic off the public internet; enforces endpoint policies restricting actions/principals |
| **Usage** | `aws ec2 create-vpc-endpoint --service-name com.amazonaws.us-east-1.secretsmanager --vpc-endpoint-type Interface --private-dns-enabled` |
| **Scenarios** | Secrets Manager private access (01-secure-multi-tier), S3 bucket policy with VPC endpoint condition (02-aws-security) |

#### Transit Gateway

| Attribute | Detail |
|---|---|
| **What** | Regional hub-and-spoke network router that connects up to 5,000 VPCs, VPN tunnels, and Direct Connect gateways through a single managed attachment point.<br/><br/>Each attachment is associated with a route table, enabling segmented routing domains — e.g., a "Security" route table inspects all inter-VPC traffic via a firewall appliance, while a "Shared Services" table allows direct connectivity. Supports cross-region peering for global transit networks. Example: CDE VPC traffic routes through TGW → Network Firewall for L7 inspection before reaching the non-CDE VPC. |
| **Problem Solved** | Eliminates N×N VPC peering; centralizes routing and firewall inspection between VPCs |
| **Usage** | Attach CDE and non-CDE VPCs; route inter-VPC traffic through Transit Gateway Firewall for L7 inspection |
| **Scenarios** | PCI CDE-to-non-CDE segmentation (08-compliance), multi-account networking (01-multi-cloud-patterns) |

#### AWS Direct Connect

| Attribute | Detail |
|---|---|
| **What** | Dedicated private network connection from on-premise to AWS, available at 1 Gbps, 10 Gbps, or 100 Gbps, bypassing the public internet entirely.<br/><br/>Supports three virtual interface (VIF) types: Private VIF (access VPCs), Public VIF (access AWS public services like S3), and Transit VIF (attach to Transit Gateway for multi-VPC access). Link Aggregation Groups (LAGs) bond multiple connections for redundancy. Example: a financial institution provisions two DX connections via Equinix in active/passive LAG, with a Site-to-Site VPN as tertiary backup. |
| **Problem Solved** | Consistent latency and bandwidth for hybrid workloads; avoids internet for sensitive traffic |
| **Usage** | Provision via Equinix/Megaport interconnect fabric; pair with VPN as backup |
| **Scenarios** | M&A network integration (02-multi-cloud-scenarios), hybrid connectivity (01-multi-cloud-patterns) |

---

### Azure Networking

#### NSGs (Network Security Groups)

| Attribute | Detail |
|---|---|
| **What** | Stateful network filtering applied at both subnet and NIC level, with priority-ordered rules (lower number = higher priority, range 100-4096).<br/><br/>Supports augmented security rules with Service Tags (e.g., `AzureCloud`, `Internet`, `Sql`) for dynamic IP management and Application Security Groups (ASGs) for role-based grouping without hardcoded IPs. NSG Flow Logs (v2) capture 5-tuple metadata and byte counts for forensic analysis. Example: an NSG rule allows `Tcp/443` from ASG `web-tier` to ASG `api-tier`, automatically applying to any VM added to those groups. |
| **Problem Solved** | Subnet-level traffic control; flow logging for forensics and anomaly detection |
| **Usage** | Apply to subnets; enable NSG Flow Logs → Storage Account → Traffic Analytics → Sentinel |
| **Scenarios** | Lateral movement containment (03-azure-security), VM quarantine during incident response |

#### Azure Firewall

| Attribute | Detail |
|---|---|
| **What** | Fully managed, cloud-native L4-L7 firewall with FQDN-based filtering, threat intelligence feeds, and centralized policy management across hub-spoke topologies.<br/><br/>The Premium SKU adds TLS inspection (terminating and re-encrypting TLS to inspect encrypted traffic), IDPS with signature-based detection, URL categorization, and Web categories filtering. Also functions as a DNS proxy, enabling FQDN rules for network rules (not just application rules). Example: in a hub VNet, Azure Firewall inspects all spoke-to-internet egress, blocking traffic to known C2 domains via Microsoft Threat Intelligence. |
| **Problem Solved** | Centralized egress filtering; blocks known-malicious domains at the network edge |
| **Usage** | Deploy in hub VNet; route spoke traffic through firewall; apply FQDN rules |
| **Scenarios** | Azure network plane security (03-azure-security), CDE egress control |

#### Azure Private Endpoints

| Attribute | Detail |
|---|---|
| **What** | Private IP addresses injected into your VNet for Azure PaaS services (Key Vault, Storage, SQL, App Service), making them accessible only via internal networking.<br/><br/>Each Private Endpoint creates a NIC in your subnet with a private IP from your CIDR range. Combined with Azure Private DNS Zones, the service FQDN (e.g., `myvault.vault.azure.net`) resolves to the private IP instead of the public one — transparent to applications with zero code changes. Public access can then be fully disabled. Example: Key Vault accessible at `10.0.2.5` via Private Endpoint; public endpoint returns 403. |
| **Problem Solved** | Removes public endpoints from PaaS services; traffic stays on Azure backbone |
| **Usage** | Deploy Private Endpoint for Key Vault; disable public access; access via private IP only |
| **Scenarios** | Key Vault without public endpoint (03-azure-security), HIPAA ePHI protection |

#### Azure ExpressRoute

| Attribute | Detail |
|---|---|
| **What** | Dedicated private connection from on-premise or other clouds to Azure, available at speeds up to 100 Gbps via ExpressRoute Direct.<br/><br/>Supports two peering types: Azure Private Peering (access VNets) and Microsoft Peering (access M365/Dynamics). Global Reach enables site-to-site connectivity through the Microsoft backbone (e.g., London office ↔ Frankfurt DC routed via Azure). FastPath bypasses the ExpressRoute Gateway for latency-sensitive workloads, routing directly to VNet VMs. Example: two ExpressRoute circuits from different providers in active-active for carrier-diverse redundancy. |
| **Problem Solved** | Private, low-latency connectivity; required for compliance workloads avoiding public internet |
| **Usage** | Provision via Equinix/Megaport; peer with Azure VNets via ExpressRoute Gateway |
| **Scenarios** | M&A Azure integration (02-multi-cloud-scenarios), cross-cloud connectivity (01-multi-cloud-patterns) |

---

### Cross-Cloud & Multi-Cloud Connectivity

#### WireGuard

| Attribute | Detail |
|---|---|
| **What** | Modern, lightweight VPN protocol built on the Noise Protocol Framework, using ChaCha20-Poly1305 encryption in just ~4,000 lines of kernel code.<br/><br/>Significantly faster than OpenVPN (userspace) and simpler than IPSec (no IKE negotiation phase — peers exchange static public keys). Operates over UDP with built-in roaming support, making it ideal for cloud-to-cloud tunnels where endpoints may change IPs. Example: an AWS VPN gateway VM (`10.0.1.5`) peers with an Azure VM (`172.16.1.5`), encrypting cross-cloud traffic at near-wire speed with <1ms overhead. |
| **Problem Solved** | Encrypted cross-cloud tunnels with minimal overhead; faster than IPSec for cloud-to-cloud |
| **Usage** | Deploy on VPN gateway VMs in each cloud; configure peer endpoints and allowed IPs |
| **Scenarios** | AWS-to-Azure encrypted tunnel (02-multi-cloud-scenarios), cross-cloud DR connectivity |

#### Equinix Fabric / Megaport

| Attribute | Detail |
|---|---|
| **What** | Cloud-neutral interconnect platforms providing private circuits between AWS, Azure, GCP, and colocation |
| **Problem Solved** | Deterministic latency and bandwidth for cross-cloud traffic; avoids public internet |
| **Usage** | Provision virtual cross-connect from Equinix to AWS Direct Connect + Azure ExpressRoute |
| **Scenarios** | M&A dual-cloud connectivity (02-multi-cloud-scenarios), financial services cross-cloud (01-multi-cloud-patterns) |

#### SD-WAN (Cisco Viptela / VMware VeloCloud)

| Attribute | Detail |
|---|---|
| **What** | Software-defined overlay network that decouples network control from the physical transport, providing centralized policy management and application-aware routing across WAN links.<br/><br/>Unlike traditional MPLS networks, SD-WAN dynamically selects the best path (internet, MPLS, LTE, cloud interconnect) per application based on real-time metrics (latency, jitter, loss). Modern platforms integrate ZTNA (Zero Trust Network Access) for per-user, per-app access control. Example: Cisco Viptela routes video conferencing traffic over the low-latency DX circuit while sending email traffic over broadband, with automatic failover in <1 second. |
| **Problem Solved** | Multi-WAN path selection; automatic failover between internet, MPLS, and cloud circuits |
| **Usage** | Deploy SD-WAN edge appliances in each cloud VPC/VNet; centrally define routing policies |
| **Scenarios** | Multi-cloud overlay networking (01-multi-cloud-patterns), branch-to-cloud connectivity |

#### IPAM (IP Address Management)

| Attribute | Detail |
|---|---|
| **What** | Centralized IP address allocation strategy and tooling that prevents CIDR overlaps and routing conflicts when interconnecting multi-cloud, hybrid, and multi-account networks.<br/><br/>Tools like NetBox (open-source), Infoblox (enterprise), or AWS VPC IPAM provide automated CIDR allocation, conflict detection, and subnet planning. The core challenge is ensuring RFC 1918 ranges don't collide when VPCs/VNets are peered or connected via VPN. Example: assigning AWS `10.0.0.0/8`, Azure `172.16.0.0/12`, GCP `192.168.0.0/16` as supernets, then carving `/20` blocks per environment per region from each supernet. |
| **Problem Solved** | Prevents routing conflicts when connecting AWS (`10.0.0.0/8`), Azure (`172.16.0.0/12`), GCP VPCs |
| **Usage** | Assign non-overlapping supernets per cloud: AWS `10.0.0.0/8`, Azure `172.16.0.0/12`, GCP `192.168.0.0/16` |
| **Scenarios** | Multi-cloud CIDR planning (01-multi-cloud-patterns), M&A network merge (02-multi-cloud-scenarios) |

---

### DNS & Traffic Routing

#### Route 53

| Attribute | Detail |
|---|---|
| **What** | AWS managed DNS with health-check-based failover, latency-based routing, and weighted routing |
| **Problem Solved** | Global traffic steering; automated failover from primary to secondary region on health check failure |
| **Usage** | Primary-failover record set with health checks; latency-based routing for multi-region |
| **Scenarios** | Multi-region failover (03-multi-region-failover), CDN origin routing (04-cdn-edge), HA networking (05) |

#### Anycast Routing

| Attribute | Detail |
|---|---|
| **What** | Multiple geographically distributed hosts sharing the same IP address; BGP routes clients to nearest |
| **Problem Solved** | Automatic geo-proximity routing; DDoS absorption by distributing attack traffic across locations |
| **Usage** | Advertise same /24 prefix from multiple PoPs via BGP; clients route to nearest PoP |
| **Scenarios** | CDN edge architecture (04-cdn-edge), HA networking (05-high-availability-networking) |

---

### CDN & Edge Networking

#### CloudFront

| Attribute | Detail |
|---|---|
| **What** | AWS CDN with 400+ edge locations; supports Lambda@Edge, CloudFront Functions, and origin protection |
| **Problem Solved** | Reduces latency via edge caching; absorbs DDoS at the edge; offloads origin servers |
| **Usage** | Distribution → Origin (ALB/S3) with OAC; custom headers for origin protection |
| **Scenarios** | CDN architecture (04-cdn-edge), video streaming delivery, static asset caching, DDoS resilience |

#### Lambda@Edge vs CloudFront Functions

| Attribute | Detail |
|---|---|
| **What** | Lambda@Edge: Node.js/Python at 13 regional edge caches (up to 10s, 10MB). CloudFront Functions: JS at 400+ PoPs (sub-ms, 10KB) |
| **Problem Solved** | Edge compute for request/response manipulation without origin round-trips |
| **Usage** | CloudFront Functions: header manipulation, URL rewrites. Lambda@Edge: A/B testing, auth, geo-routing |
| **Scenarios** | Token validation at edge (04-cdn-edge), geo-based content routing, security header injection |

#### Origin Access Control (OAC)

| Attribute | Detail |
|---|---|
| **What** | CloudFront mechanism restricting S3 bucket access to only CloudFront distribution requests |
| **Problem Solved** | Prevents direct S3 access bypassing CDN caching and WAF protections |
| **Usage** | Configure OAC on distribution; S3 bucket policy allows only CloudFront service principal |
| **Scenarios** | S3 origin protection (04-cdn-edge), static site security |

---

### Load Balancing

#### ALB (Application Load Balancer)

| Attribute | Detail |
|---|---|
| **What** | AWS Layer 7 load balancer that routes HTTP/HTTPS requests based on host headers, URL paths, query strings, and HTTP methods to target groups of EC2, ECS, Lambda, or IP targets.<br/><br/>Supports native OIDC/Cognito authentication at the LB layer (offloading auth from backends), WebSocket and gRPC protocols, slow-start mode for warming up new targets, and sticky sessions via app cookies. Integrates directly with AWS WAF for L7 filtering. Example: `/api/*` routes to ECS Fargate target group, `/static/*` routes to S3 via fixed-response, and `/health` returns a 200 directly from the ALB. |
| **Problem Solved** | Distributes HTTP/HTTPS traffic across targets with content-based routing |
| **Usage** | Target groups per microservice; health checks; WAF association for L7 filtering |
| **Scenarios** | Multi-tier architecture (01-secure-multi-tier), API gateway frontend (06-api-gateway) |

#### NLB (Network Load Balancer)

| Attribute | Detail |
|---|---|
| **What** | AWS Layer 4 load balancer operating at the TCP/UDP/TLS level, capable of handling millions of requests per second with ultra-low latency (~100µs added) and static Elastic IPs per AZ.<br/><br/>Preserves client source IP natively (no X-Forwarded-For needed), supports TLS passthrough (backend terminates TLS), and serves as the entry point for AWS PrivateLink services. Cross-zone load balancing distributes traffic evenly across AZs. Example: NLB with static IPs fronts a gRPC service on EKS, enabling firewall allowlisting by IP; also exposes the service via PrivateLink to partner VPCs without VPC peering. |
| **Problem Solved** | TCP/UDP load balancing for non-HTTP protocols; preserves source IP |
| **Usage** | Front gRPC services, database proxies, or VPN endpoints |
| **Scenarios** | HA networking (05-high-availability), cross-AZ database routing |

---

### Service Mesh & Service-to-Service Communication

#### Istio

| Attribute | Detail |
|---|---|
| **What** | Service mesh providing mTLS, traffic management, and observability via Envoy sidecar proxies |
| **Problem Solved** | Zero-trust service-to-service communication; automatic certificate rotation; L7 traffic policies |
| **Usage** | `PeerAuthentication: STRICT` for mesh-wide mTLS; `AuthorizationPolicy` for per-service access control |
| **Scenarios** | Zero Trust design (02-zero-trust-design), API gateway backend (06-api-gateway), container security (04) |

#### Envoy Proxy

| Attribute | Detail |
|---|---|
| **What** | High-performance L7 proxy used as Istio sidecar and standalone API gateway (100K+ RPS/node) |
| **Problem Solved** | Transparent mTLS, circuit breaking, retry with budget, gRPC-JSON transcoding |
| **Usage** | Sidecar in service mesh; standalone edge gateway for API routing at scale |
| **Scenarios** | API gateway design (06-api-gateway), HA networking (05), Zero Trust enforcement |

#### Cilium

| Attribute | Detail |
|---|---|
| **What** | eBPF-based CNI providing networking, security, and observability for Kubernetes |
| **Problem Solved** | Kernel-level network policy enforcement without iptables overhead; native Hubble observability |
| **Usage** | Deploy as CNI; use CiliumNetworkPolicy for L3-L7 enforcement; Tetragon for runtime security |
| **Scenarios** | Container networking (04-container-security), eBPF enforcement via Tetragon |

#### Calico

| Attribute | Detail |
|---|---|
| **What** | Kubernetes CNI with NetworkPolicy enforcement, BGP routing, and eBPF dataplane option |
| **Problem Solved** | Default-deny network policies per namespace; pod-level microsegmentation |
| **Usage** | `NetworkPolicy: default-deny ingress/egress` per namespace; explicit allows for required flows |
| **Scenarios** | AKS lateral movement prevention (03-azure-security), PCI CDE namespace isolation (08-compliance) |

#### SPIFFE / SPIRE

| Attribute | Detail |
|---|---|
| **What** | SPIFFE: universal workload identity standard (SVID). SPIRE: production implementation issuing x509 SVIDs |
| **Problem Solved** | Cryptographic workload identity without static credentials; enables zero-trust service authentication |
| **Usage** | SPIRE Agent on each node → issues SVIDs to pods → SVIDs used as mTLS certificates |
| **Scenarios** | Zero Trust architecture (01-zero-trust, 02-zero-trust-design), cross-cloud identity (02-multi-cloud-scenarios) |

---

## 2. Identity & Access Management

### AWS IAM & Identity

#### IAM Policies (Identity-Based & Resource-Based)

| Attribute | Detail |
|---|---|
| **What** | JSON policy documents controlling API-level access. Identity-based attach to users/roles; resource-based attach to S3/SQS/KMS |
| **Problem Solved** | Fine-grained API authorization; resource-based policies restrict access by VPC endpoint or source IP |
| **Usage** | Bucket policy with `StringNotEquals: aws:SourceVpce` condition denies all non-VPC-endpoint access |
| **Scenarios** | S3 VPC endpoint restriction (02-aws-security), PCI least privilege (08-compliance), CDE IAM scoping |

#### IAM Access Analyzer

| Attribute | Detail |
|---|---|
| **What** | AWS service that continuously analyzes resource policies (S3, SQS, KMS, Lambda, IAM roles) to identify resources shared with external entities outside your AWS Organization.<br/><br/>Operates in two modes: External Access Analysis (finds resources accessible to external principals — other accounts, public) and Unused Access Analysis (identifies IAM roles/users with permissions not exercised in 90+ days). Can auto-generate least-privilege policies from CloudTrail activity. Example: detects an S3 bucket policy granting `s3:GetObject` to `Principal: "*"` and raises a finding for immediate remediation. |
| **Problem Solved** | Finds accidental public exposure; generates least-privilege policies from CloudTrail activity |
| **Usage** | `aws iam generate-service-last-accessed-details` → generates minimum-privilege policy from 90 days of activity |
| **Scenarios** | Least privilege enforcement (02-aws-security), SOC 2 quarterly access reviews (08-compliance) |

#### IRSA (IAM Roles for Service Accounts)

| Attribute | Detail |
|---|---|
| **What** | EKS-native identity mechanism that maps Kubernetes ServiceAccounts to AWS IAM Roles via OIDC federation, eliminating the need for static AWS access keys in pods.<br/><br/>Works by configuring an OIDC identity provider in IAM, then annotating a K8s ServiceAccount with the IAM role ARN. The pod receives a projected service account token (JWT) that STS exchanges for temporary AWS credentials via `AssumeRoleWithWebIdentity`. The IAM role's trust policy validates the token's `iss` (cluster OIDC URL) and `sub` (namespace:serviceaccount). Example: a pod running ESO assumes `arn:aws:iam::123:role/eso-secrets-reader` scoped to only `secretsmanager:GetSecretValue`. |
| **Problem Solved** | Eliminates AWS access keys in pods; per-pod IAM role scoping |
| **Usage** | Annotate SA with `eks.amazonaws.com/role-arn`; pod assumes role via projected token |
| **Scenarios** | ESO accessing Secrets Manager (06-secrets-mgmt), Zero Trust workload identity (01-zero-trust) |

#### STS (Security Token Service)

| Attribute | Detail |
|---|---|
| **What** | AWS service that issues temporary, limited-privilege security credentials (access key + secret key + session token) for assuming IAM roles and cross-account access.<br/><br/>Supports multiple federation types: `AssumeRole` (cross-account), `AssumeRoleWithSAML` (enterprise SSO), and `AssumeRoleWithWebIdentity` (OIDC/Cognito). Session policies can further restrict the assumed role's permissions. Uses External ID in trust policies to prevent the confused deputy problem in third-party integrations. Example: a CI/CD pipeline in Account A calls `sts:AssumeRole` to deploy to Account B with a 15-minute session and external ID `deploy-prod-2024`. |
| **Problem Solved** | Short-lived credentials reduce blast radius of compromise; enables cross-account patterns |
| **Usage** | `aws sts assume-role` with short TTL; session policies limit assumed role permissions further |
| **Scenarios** | Cross-account access (02-aws-security), incident response credential rotation |

#### SCPs (Service Control Policies)

| Attribute | Detail |
|---|---|
| **What** | Organization-level permission guardrails applied to OUs and accounts that set the maximum available permissions, overriding any account-level IAM policy — even for account root users.<br/><br/>Typically implemented as deny-list strategy (allow `*`, then explicitly deny dangerous actions) rather than allow-list (which is harder to maintain). SCPs follow the organizational hierarchy: a deny at the OU level applies to all child accounts. They do not grant permissions — they only restrict. Example: an SCP on the Production OU denies `ec2:RunInstances` unless the instance type is in an approved list and the region is `us-east-1` or `eu-west-1`. |
| **Problem Solved** | Prevents disabling security controls (CloudTrail, GuardDuty); enforces mandatory guardrails |
| **Usage** | Deny: `guardduty:DeleteDetector`, `cloudtrail:StopLogging`, `iam:CreateAccessKey` for root |
| **Scenarios** | PCI defense-in-depth (02-aws-security), multi-account governance (08-compliance) |

---

### Azure Identity

#### Entra ID (Azure AD)

| Attribute | Detail |
|---|---|
| **What** | Microsoft's identity platform providing SSO, MFA, Conditional Access, and PIM for human and workload identity |
| **Problem Solved** | Centralized identity control plane; highest-value target in Azure — compromise = tenant ownership |
| **Usage** | Conditional Access policies evaluate user, location, device compliance, and risk score before granting tokens |
| **Scenarios** | Lateral movement detection (03-azure-security), impossible travel detection via Sentinel |

#### Managed Identity (System-Assigned & User-Assigned)

| Attribute | Detail |
|---|---|
| **What** | Azure-native identity automatically managed by the platform, where certificate credentials are rotated every 46 days with zero operator intervention.<br/><br/>Two types: System-Assigned (lifecycle tied to the resource — deleted when VM/App Service is deleted) and User-Assigned (independent lifecycle, shareable across multiple resources). The identity obtains tokens via the Instance Metadata Service (IMDS) at `169.254.169.254` — no credentials are ever stored in code or config. Tokens are short-lived (default 24h) with proactive refresh. Example: an AKS pod with User-Assigned Managed Identity requests a token for Key Vault scope, receives a JWT, and reads secrets — all without any client secret or certificate in the pod spec. |
| **Problem Solved** | Eliminates client secrets and API keys for Azure service-to-service authentication |
| **Usage** | VM/App Service calls IMDS (`169.254.169.254/metadata`) → receives short-lived access token → accesses Key Vault |
| **Scenarios** | AKS Workload Identity (03-azure-security), Key Vault access without secrets |

#### Azure PIM (Privileged Identity Management)

| Attribute | Detail |
|---|---|
| **What** | Just-in-time privileged role activation with configurable time limits (1-24 hours), mandatory justification, optional approvals, and MFA enforcement.<br/><br/>Distinguishes between Eligible Assignments (user can activate when needed) and Active Assignments (permanently assigned — should be minimized). Includes built-in Access Reviews that periodically require managers or resource owners to re-certify continued access need. All activations produce audit logs for SOC 2/ISO 27001 evidence. Example: a DevOps engineer has an eligible "Contributor" role on `prod-rg`; they activate it for 2 hours with justification "deploy hotfix CR-4521", requiring Security Lead approval. |
| **Problem Solved** | Reduces standing privilege exposure; creates audit trail for all elevations |
| **Usage** | Engineer requests Owner on prod-rg for 1 hour → Security Lead approves → role auto-expires |
| **Scenarios** | Break-glass procedures (03-azure-security), SOC 2 access control evidence (08-compliance) |

#### Conditional Access

| Attribute | Detail |
|---|---|
| **What** | Context-aware policy engine evaluating user, location, device, app sensitivity, and sign-in risk |
| **Problem Solved** | Blocks access from non-approved countries; requires MFA for elevated-risk sign-ins |
| **Usage** | Policy: block all locations except approved named locations; require compliant device for sensitive apps |
| **Scenarios** | Azure identity defense (03-azure-security), impossible travel response |

---

### Workload Identity & Federation

#### AKS Workload Identity

| Attribute | Detail |
|---|---|
| **What** | OIDC federation mechanism that maps Kubernetes ServiceAccounts to Azure User-Assigned Managed Identities, enabling per-pod Azure RBAC without any static credentials.<br/><br/>Works through a three-step token exchange: the pod receives a projected K8s service account JWT → the Azure Identity SDK exchanges it via Azure AD's OIDC endpoint → receives an Azure access token scoped to the Managed Identity's RBAC assignments. Requires a Federated Credential configured on the Managed Identity that trusts the AKS cluster's OIDC issuer URL and specific namespace:serviceaccount. Example: the payments pod in `ns:payments/sa:payments-api` exchanges its K8s token for an Azure token and reads secrets from Key Vault — no connection string or client secret needed. |
| **Problem Solved** | Per-pod Azure identity without static credentials; scoped RBAC per workload |
| **Usage** | SA annotation `azure.workload.identity/client-id` + pod label `azure.workload.identity/use: "true"` |
| **Scenarios** | AKS pod identity (03-azure-security), payments API accessing Key Vault |

#### Crossplane

| Attribute | Detail |
|---|---|
| **What** | Kubernetes-native infrastructure provisioning using CRDs to manage cloud resources (RDS, Azure SQL) |
| **Problem Solved** | Cloud-agnostic infrastructure abstraction; unified platform management across AWS and Azure |
| **Usage** | Define `CompositeResourceDefinition` mapping to AWS RDS + Azure SQL; teams create resources via `kubectl` |
| **Scenarios** | Unified Kubernetes platform (02-multi-cloud-scenarios), multi-cloud resource provisioning |

---

## 3. Security & Threat Detection

### AWS Security Services

#### AWS GuardDuty

| Attribute | Detail |
|---|---|
| **What** | Managed threat detection analyzing CloudTrail, VPC Flow Logs, DNS logs, S3 data events, and EKS audit logs |
| **Problem Solved** | Detects in-progress threats: crypto mining, credential theft, port scanning, data exfiltration |
| **Usage** | Enable org-wide; EventBridge triggers Lambda for automated response (quarantine EC2, block IP) |
| **Scenarios** | S3 exfiltration detection (02-aws-security), SSRF credential theft (07-owasp-api), HA monitoring (05) |

#### AWS Security Hub

| Attribute | Detail |
|---|---|
| **What** | CSPM aggregating findings from GuardDuty, Inspector, Config, Macie, IAM Access Analyzer in ASFF format |
| **Problem Solved** | Single security score across 350+ controls; maps to CIS, PCI, and AWS Security Best Practices |
| **Usage** | Enable standards; remediate findings; cross-account aggregation via Organization |
| **Scenarios** | PCI compliance dashboard (08-compliance), SOC 2 evidence (08-compliance), S3 exfiltration triage |

#### AWS Inspector

| Attribute | Detail |
|---|---|
| **What** | Continuous vulnerability scanning for EC2 (via SSM Agent), ECR images (on push + new CVE), and Lambda functions |
| **Problem Solved** | OS-level CVE detection; blocks CI/CD if CRITICAL vulnerabilities found in container images |
| **Usage** | `aws inspector2 list-findings --filter-criteria severity=CRITICAL` in CD pipeline to gate deployment |
| **Scenarios** | ECR image scanning (02-aws-security), container vulnerability management (04-container-security) |

#### AWS Config

| Attribute | Detail |
|---|---|
| **What** | AWS service that continuously records resource configuration state and changes over time, evaluating them against managed rules, custom rules (Lambda-backed), and conformance packs.<br/><br/>Config Aggregator provides a multi-account, multi-region single-pane view of compliance status. The Configuration Timeline shows exactly what changed, when, and by whom (via CloudTrail correlation). Supports auto-remediation via SSM Automation documents — e.g., a non-compliant SG rule is automatically revoked. Example: the PCI Conformance Pack evaluates 40+ rules (`restricted-ssh`, `encrypted-volumes`, `vpc-flow-logs-enabled`) and triggers SSM remediation within minutes of drift detection. |
| **Problem Solved** | Compliance drift detection; automated remediation via SSM Automation when rules are violated |
| **Usage** | PCI Conformance Pack: `restricted-ssh`, `vpc-flow-logs-enabled`, `cloud-trail-enabled` + auto-remediation |
| **Scenarios** | PCI compliance automation (08-compliance), CDE SG monitoring (08-compliance), SOC 2 evidence |

#### AWS Macie

| Attribute | Detail |
|---|---|
| **What** | ML-powered data classification service that automatically discovers, classifies, and protects sensitive data stored in Amazon S3 buckets across your organization.<br/><br/>Uses ML models and pattern matching to identify sensitive data types: PII (names, SSNs, credit cards), PHI (medical records), credentials (API keys, private keys), and financial data. Supports custom data identifiers using regex + proximity keywords for organization-specific patterns. Scheduled discovery jobs can scan all buckets or target specific prefixes. Example: Macie scans 500 S3 buckets nightly, finds 12 buckets containing unencrypted credit card numbers, and sends findings to Security Hub for SOC triage. |
| **Problem Solved** | Identifies which S3 buckets contain sensitive data; detects accidental public exposure |
| **Usage** | Enable on S3 buckets; findings feed Security Hub; cross-reference with PII classification during breach |
| **Scenarios** | S3 data exfiltration scope assessment (02-aws-security), GDPR/CCPA notification determination |

#### AWS CloudTrail

| Attribute | Detail |
|---|---|
| **What** | API audit log recording every AWS API call — who, what, when, where — with SHA-256 integrity validation |
| **Problem Solved** | Primary forensic data source for incident investigation; proves compliance for audit evidence |
| **Usage** | Multi-region org trail; data events for S3/Lambda; query via Athena or `lookup-events` CLI |
| **Scenarios** | S3 exfiltration investigation (02-aws-security), IAM credential compromise (02), SOC 2 evidence (08) |

---

### Azure Security Services

#### Microsoft Defender for Cloud

| Attribute | Detail |
|---|---|
| **What** | Unified Cloud Security Posture Management (CSPM) and Cloud Workload Protection (CWP) platform covering Azure, AWS, and GCP from a single console.<br/><br/>CSPM provides Secure Score (0-100%), attack path analysis (visualizing multi-step exploit chains), and regulatory compliance dashboards. CWP plans include Defender for Servers (vulnerability assessment, adaptive application controls), Defender for Containers (runtime threat detection, admission control), Defender for SQL (anomalous query detection), and Defender for Key Vault (suspicious access patterns). Agentless scanning discovers vulnerabilities without deploying agents. Example: attack path analysis reveals Internet → exposed VM → managed identity → Key Vault → database credentials as a 4-step exploit chain requiring immediate remediation. |
| **Problem Solved** | Multi-cloud security posture visibility; prioritized remediation recommendations |
| **Usage** | Enable CSPM + workload protection plans; review Secure Score; use attack path analysis for exposure |
| **Scenarios** | Azure posture management (03-azure-security), multi-cloud CSPM (03-azure-security) |

#### Microsoft Sentinel

| Attribute | Detail |
|---|---|
| **What** | Cloud-native SIEM (Security Information and Event Management) and SOAR (Security Orchestration, Automation, and Response) platform with 200+ built-in data connectors.<br/><br/>Ingests logs from Azure (Activity, Entra ID, NSG), AWS (CloudTrail, GuardDuty), M365, and third-party sources. Analytics rules written in KQL detect threats like impossible travel, brute force, and lateral movement. UEBA (User and Entity Behavior Analytics) baselines normal behavior and flags anomalies. Workbooks provide interactive investigation dashboards. Logic App Playbooks automate response — disable user, revoke sessions, isolate VM — in seconds. Example: a KQL rule detects sign-ins from two countries within 10 minutes, triggers a Playbook that disables the account, revokes all refresh tokens, and creates a ServiceNow incident. |
| **Problem Solved** | Threat detection, investigation, and automated response across Azure, AWS, M365, and third-party |
| **Usage** | KQL analytics rules for impossible travel; Playbooks auto-disable user + revoke tokens + isolate VM |
| **Scenarios** | Lateral movement containment (03-azure-security), impossible travel detection, SOC operations |

#### Azure Policy

| Attribute | Detail |
|---|---|
| **What** | Azure Resource Manager-level policy enforcement engine that evaluates every resource create/update/delete against policy rules, with effects including Audit, Deny, Modify, DeployIfNotExists, and Disabled.<br/><br/>Policies are individual rules; Initiatives (formerly Policy Sets) group related policies for compliance standards like HIPAA/HITRUST, CIS, or NIST. Remediation Tasks can retroactively fix non-compliant existing resources. Exemptions allow time-limited exceptions with justification. Example: a HIPAA initiative applied at subscription scope includes 120+ policies; a `DeployIfNotExists` policy automatically enables encryption on any new Storage Account that lacks it. |
| **Problem Solved** | Prevents non-compliant resource creation; auto-deploys required configurations |
| **Usage** | HIPAA/HITRUST initiative applied at subscription scope; AKS security baseline initiative |
| **Scenarios** | HIPAA compliance (08-compliance), AKS security enforcement (03-azure-security) |

---

### Web Application Security

#### AWS WAF

| Attribute | Detail |
|---|---|
| **What** | AWS Layer 7 web application firewall that inspects HTTP/HTTPS requests using managed rule groups (AWS Managed Rules, OWASP Core Rule Set, Bot Control, Account Takeover Prevention) and custom rules.<br/><br/>Supports rate-based rules (auto-block IPs exceeding thresholds), IP reputation lists (Amazon IP threat intelligence), geo-match conditions, and regex pattern matching. Logs to S3, CloudWatch, or Kinesis Firehose for analysis. Operates at both CloudFront (edge) and ALB/API Gateway (regional) attachment points. Example: a WAF WebACL on CloudFront blocks SQL injection via managed OWASP rules, rate-limits to 2000 RPM per IP, and uses a custom rule to deny requests to `/actuator/*` and `/admin/*` paths from external IPs. |
| **Problem Solved** | Blocks SQL injection, XSS, SSRF patterns, bot traffic, and debug endpoint access |
| **Usage** | Associate with ALB/CloudFront; managed OWASP Core Rule Set + custom rules for `/actuator/*` blocking |
| **Scenarios** | CDN edge security (04-cdn-edge), API security misconfiguration (07-owasp-api), business flow abuse |

#### OPA (Open Policy Agent) / Cedar

| Attribute | Detail |
|---|---|
| **What** | General-purpose policy engine for authorization decisions. OPA uses the Rego declarative language and runs as a sidecar/library evaluating policies against structured data (JSON). Cedar (by AWS) uses a typed, human-readable policy language with schema validation.<br/><br/>OPA is widely used for Kubernetes admission control (via Gatekeeper), API authorization, and Terraform plan validation. Cedar's advantage is its typed schemas — policies are validated against an entity schema at authoring time, catching errors before deployment. Both support sub-millisecond evaluation at 10K+ RPS when embedded as an in-process library. Example: an OPA Rego policy maps each API route + HTTP method to required roles: `allow { input.method == "DELETE"; input.path == "/api/users"; "admin" in input.user.roles }`. |
| **Problem Solved** | Centralized default-deny authorization; replaces scattered `if (user.role == "admin")` checks |
| **Usage** | Maps routes + methods to required roles; in-process SDK for sub-ms evaluation at 10K+ RPS |
| **Scenarios** | BFLA prevention (07-owasp-api), Zero Trust policy enforcement (01-zero-trust), PCI RBAC (08-compliance) |

---

### DDoS Protection

#### AWS Shield (Standard & Advanced)

| Attribute | Detail |
|---|---|
| **What** | AWS DDoS protection service in two tiers: Shield Standard (free, automatic L3/L4 protection on all AWS resources) and Shield Advanced ($3K/month, adds L7 protection, real-time visibility, cost protection, and access to the DDoS Response Team).<br/><br/>Shield Advanced provides automatic engagement of the AWS Shield Response Team (SRT) during detected events, proactive engagement for known large-scale events, and cost protection that refunds scaling charges caused by DDoS attacks. Includes DDoS event visibility dashboards and integration with WAF for L7 mitigation. Example: during a 2 Tbps volumetric attack on CloudFront, Shield Standard absorbs the L3/L4 flood; Shield Advanced's SRT deploys custom WAF rules to filter the L7 application-layer attack, and the customer is refunded for the CloudFront traffic spike. |
| **Problem Solved** | Absorbs volumetric DDoS at the edge; Advanced provides real-time attack visibility and response team |
| **Usage** | Shield Advanced on CloudFront + ALB; DDoS Response Team (SRT) for active attacks |
| **Scenarios** | CDN DDoS resilience (04-cdn-edge), HA networking (05-high-availability) |

#### Azure DDoS Protection

| Attribute | Detail |
|---|---|
| **What** | Azure-managed DDoS mitigation service with two plans: DDoS Network Protection (per-VNet, adaptive tuning based on traffic patterns) and DDoS IP Protection (per-public-IP, cost-effective for smaller deployments).<br/><br/>Automatically profiles normal traffic patterns for each public IP and mitigates anomalies in real-time without impacting legitimate traffic. Includes 24/7 Rapid Response support during active attacks, DDoS protection telemetry in Azure Monitor, and cost guarantee credits if DDoS causes resource scale-out charges. Example: a public-facing AKS cluster with DDoS Network Protection auto-mitigates a SYN flood attack within 30 seconds, with attack telemetry streamed to Sentinel for SOC visibility. |
| **Problem Solved** | Protects Azure public endpoints from volumetric and protocol attacks |
| **Usage** | Enable on VNet; auto-mitigates; logs available in Azure Monitor |
| **Scenarios** | Azure network security (03-azure-security) |

---

## 4. Container & Kubernetes Security

### Image Security

#### Trivy

| Attribute | Detail |
|---|---|
| **What** | Open-source vulnerability scanner for container images, filesystems, IaC, and SBOM generation |
| **Problem Solved** | Detects OS and language-level CVEs in container images; integrates into CI/CD as gate |
| **Usage** | `trivy image --severity CRITICAL,HIGH --exit-code 1 myimage:latest` in CI pipeline |
| **Scenarios** | CI/CD image scanning (04-container-security), ECR pipeline gate (05-supply-chain) |

#### Distroless Images

| Attribute | Detail |
|---|---|
| **What** | Google-maintained minimal container images containing only the application and runtime — no shell, package manager, or OS utilities |
| **Problem Solved** | Reduces attack surface by 90%+; eliminates shell-based exploitation (reverse shells, command injection) |
| **Usage** | `FROM gcr.io/distroless/java21-debian12:nonroot` as final stage in multi-stage Dockerfile |
| **Scenarios** | Image hardening (04-container-security), PCI container standards (08-compliance) |

#### Multi-Stage Builds

| Attribute | Detail |
|---|---|
| **What** | Docker build pattern separating build environment from runtime; only final stage ships to production |
| **Problem Solved** | Excludes compilers, build tools, and source code from production images |
| **Usage** | Stage 1: `FROM golang:1.22 AS builder` → Stage 2: `FROM gcr.io/distroless/static-debian12` COPY binary only |
| **Scenarios** | Image size reduction (04-container-security), supply chain surface minimization (05-supply-chain) |

---

### Runtime Security

#### Falco

| Attribute | Detail |
|---|---|
| **What** | Cloud-native runtime security tool using kernel syscall monitoring to detect anomalous container behavior |
| **Problem Solved** | Detects runtime threats: shell spawned in container, sensitive file read, unexpected network connections |
| **Usage** | Deploy as DaemonSet; custom rules alert on `execve(bash)` in production containers |
| **Scenarios** | Container runtime detection (04-container-security), PCI intrusion detection (08-compliance) |

#### Tetragon

| Attribute | Detail |
|---|---|
| **What** | eBPF-based security observability and runtime enforcement engine from the Cilium project, operating at the Linux kernel level without requiring kernel modules or application changes.<br/><br/>Unlike Falco (which primarily detects and alerts), Tetragon can enforce responses directly in the kernel — killing processes, sending signals, or overriding syscall return values — with near-zero latency. Uses TracingPolicy CRDs to define match criteria (binary path, arguments, capabilities) and enforcement actions. Hooks into kprobes, tracepoints, and LSM hooks for deep kernel visibility. Example: a TracingPolicy matches any execution of `xmrig` (cryptominer) binary and immediately sends SIGKILL, preventing the process from running even a single instruction after `execve`. |
| **Problem Solved** | Kernel-level process, file, and network monitoring with enforcement (kill, signal) — not just alerting |
| **Usage** | TracingPolicy CRD defines: match on binary + enforce action (sigkill); blocks cryptominer execution |
| **Scenarios** | eBPF runtime enforcement (04-container-security), advanced threat response |

#### seccomp

| Attribute | Detail |
|---|---|
| **What** | Linux kernel security facility (Secure Computing Mode) that restricts which system calls a containerized process is allowed to make, acting as a syscall-level firewall.<br/><br/>Three action types: `SCMP_ACT_ALLOW` (permit), `SCMP_ACT_ERRNO` (deny with error), and `SCMP_ACT_LOG` (audit mode). The `RuntimeDefault` profile (Docker/containerd default) blocks ~44 dangerous syscalls including `unshare` (namespace escape), `mount` (filesystem manipulation), `ptrace` (process debugging), and `reboot`. Custom profiles can be generated from production behavior using tools like `oci-seccomp-bpf-hook` or Inspektor Gadget. Example: a Java application's custom profile allows `clone3` (needed for JVM threads) but blocks `personality` and `keyctl` (not needed, reduces attack surface). |
| **Problem Solved** | Prevents container escape via dangerous syscalls like `unshare`, `mount`, `ptrace` |
| **Usage** | `RuntimeDefault` profile blocks ~44 dangerous syscalls; custom profiles for app-specific allowlists |
| **Scenarios** | Pod hardening (04-container-security), PCI container runtime controls (08-compliance) |

#### AppArmor

| Attribute | Detail |
|---|---|
| **What** | Linux Security Module (LSM) that enforces mandatory access control via per-program profiles, restricting file reads/writes, network access, capability usage, and mount operations.<br/><br/>Profiles operate in two modes: Complain (log violations but allow) for profiling, and Enforce (block violations) for production. Unlike SELinux (which uses labels/contexts and is more complex), AppArmor uses path-based rules that are easier to write and audit. Profiles are loaded into the kernel and applied per-container via pod annotations. Example: a profile for a web server allows read access to `/app/**` and `/etc/ssl/**`, network TCP on port 8080, and denies all writes except to `/tmp/**` — preventing an attacker from writing a webshell to the application directory. |
| **Problem Solved** | Constrains container filesystem access and network operations beyond what namespaces provide |
| **Usage** | Pod annotation `container.apparmor.security.beta.kubernetes.io/app: runtime/default` |
| **Scenarios** | Container confinement (04-container-security), defense-in-depth layering |

---

### Admission Control & Policy Enforcement

#### Kyverno

| Attribute | Detail |
|---|---|
| **What** | Kubernetes-native policy engine using YAML (no Rego); validates, mutates, and generates resources |
| **Problem Solved** | Enforces security baselines: require non-root, block latest tags, mandate resource limits |
| **Usage** | `ClusterPolicy: validate → deny pods with runAsRoot=true; mutate → inject seccomp profile` |
| **Scenarios** | Supply chain admission (05-supply-chain), container policy enforcement (04-container-security) |

#### OPA Gatekeeper

| Attribute | Detail |
|---|---|
| **What** | OPA-based Kubernetes admission controller using ConstraintTemplates (Rego) for policy enforcement |
| **Problem Solved** | Shift-left security: blocks non-compliant resources at admission before they reach the cluster |
| **Usage** | `ConstraintTemplate` + `Constraint` CRDs; auditFromCache for existing resource compliance checks |
| **Scenarios** | Zero Trust admission (02-zero-trust-design), PCI workload policies (08-compliance) |

#### Pod Security Admission (PSA)

| Attribute | Detail |
|---|---|
| **What** | Built-in Kubernetes admission controller (GA since v1.25) that enforces Pod Security Standards at the namespace level using three modes that can operate simultaneously on the same namespace.<br/><br/>The three modes are: Enforce (reject non-compliant pods), Audit (allow but log violations in audit logs), and Warn (allow but return a warning to the user). This enables a staged rollout: start with `warn: restricted` to discover violations, then escalate to `audit`, and finally `enforce`. Three security levels: Privileged (unrestricted), Baseline (prevents known privilege escalations), and Restricted (hardened, requires non-root, drops capabilities, read-only root filesystem). Example: namespace label `pod-security.kubernetes.io/enforce: restricted, pod-security.kubernetes.io/warn: restricted` rejects non-compliant pods and warns on `kubectl apply`. |
| **Problem Solved** | Baseline security without external tools; three levels: Privileged, Baseline, Restricted |
| **Usage** | Label namespace: `pod-security.kubernetes.io/enforce: restricted` |
| **Scenarios** | Namespace security baseline (04-container-security), PCI pod hardening (08-compliance) |

---

### Pod Security

#### Kubernetes NetworkPolicy

| Attribute | Detail |
|---|---|
| **What** | Kubernetes-native L3/L4 network access control between pods, using label selectors and namespace selectors to define allowed ingress and egress traffic flows.<br/><br/>Without any NetworkPolicy, all pods can freely communicate with all other pods (flat network). Applying a policy to a pod immediately makes it default-deny for the selected direction (ingress/egress). Rules use `podSelector` (select pods in same namespace), `namespaceSelector` (select pods in other namespaces), and `ipBlock` (external CIDRs). DNS egress (UDP/53) must be explicitly allowed when egress is restricted. Example: a payment namespace policy denies all ingress except from pods with label `app: api-gateway` on port 8443, and denies all egress except to pods with label `app: postgres` on port 5432 and DNS on UDP/53. |
| **Problem Solved** | East-west microsegmentation; default-deny prevents lateral movement between namespaces |
| **Usage** | Default deny all ingress/egress → explicit allow rules for required flows (e.g., frontend → backend on port 8080) |
| **Scenarios** | Zero Trust pod isolation (02-zero-trust-design), PCI CDE segmentation (08-compliance) |

#### Pod Security Context

| Attribute | Detail |
|---|---|
| **What** | Kubernetes spec fields controlling container runtime privileges: runAsUser, capabilities, readOnlyRootFilesystem |
| **Problem Solved** | Prevents privilege escalation; enforces principle of least privilege at the container level |
| **Usage** | `runAsNonRoot: true, readOnlyRootFilesystem: true, allowPrivilegeEscalation: false, drop: [ALL]` |
| **Scenarios** | Container hardening (04-container-security), Kyverno enforcement (04), PCI containers (08) |

---

## 5. Supply Chain Security

### Signing & Provenance

#### Sigstore (Cosign / Fulcio / Rekor)

| Attribute | Detail |
|---|---|
| **What** | Cosign: container image signing. Fulcio: keyless CA issuing short-lived certs from OIDC. Rekor: immutable transparency log |
| **Problem Solved** | Proves image provenance without managing long-lived signing keys; tamper-evident audit trail |
| **Usage** | `cosign sign --identity-token=$(gcloud auth print-identity-token) ${IMAGE}` → signature stored in OCI registry |
| **Scenarios** | Keyless signing in CI/CD (05-supply-chain), admission verification via Kyverno |

#### SLSA Framework (Supply-chain Levels for Software Artifacts)

| Attribute | Detail |
|---|---|
| **What** | Security framework with 4 levels (L1-L4) defining build integrity, provenance, and hardening requirements |
| **Problem Solved** | Protects against compromised build systems, tampered source code, and dependency attacks |
| **Usage** | L1: provenance exists. L2: hosted build service. L3: hardened, isolated builds. L4: two-person review + hermetic |
| **Scenarios** | Build provenance (05-supply-chain), SLSA L3 CI/CD pipeline design (05) |

#### in-toto

| Attribute | Detail |
|---|---|
| **What** | Framework for securing the entire software supply chain by defining a Layout (expected steps) and collecting signed Links (evidence of each step) from authorized Functionaries (actors/CI systems).<br/><br/>The Layout specifies which steps must occur, in what order, by which functionary, and what artifacts each step consumes/produces. After all steps complete, the final verification checks the full chain: every expected step was performed, by the authorized functionary, with cryptographic signatures. This prevents skipped steps (e.g., bypassing code review) and unauthorized modifications. Example: a Layout requires steps [checkout, build, test, scan, sign] performed by functionaries [github-actions, trivy-ci, cosign-ci]; verification fails if the scan step is missing or performed by an unauthorized identity. |
| **Problem Solved** | End-to-end supply chain integrity from source checkout to deployment |
| **Usage** | Each CI step produces a signed link; final verification checks that all expected steps were performed by authorized actors |
| **Scenarios** | Supply chain attestation (05-supply-chain), SLSA L3+ evidence |

---

### SBOM & Dependency Management

#### SBOM (CycloneDX / SPDX)

| Attribute | Detail |
|---|---|
| **What** | Machine-readable inventory of all components in a software artifact. CycloneDX: security-focused. SPDX: license-focused |
| **Problem Solved** | Enables rapid CVE impact assessment: "which production images contain log4j?" — answered in seconds vs. hours |
| **Usage** | `syft image:tag -o cyclonedx-json > sbom.json` in CI; store in registry alongside image |
| **Scenarios** | Vulnerability inventory (05-supply-chain), HIPAA software inventory (08-compliance) |

#### Dependabot / Renovate

| Attribute | Detail |
|---|---|
| **What** | Automated dependency update tools that monitor project manifests (package.json, go.mod, requirements.txt, Dockerfile FROM) and create pull requests when new versions or security patches are published.<br/><br/>Dependabot (GitHub-native) scans daily and auto-creates standalone PRs per dependency. Renovate (open-source, more configurable) supports grouping updates (e.g., all patch updates in one PR), automerge for patch/minor versions with passing CI, scheduled merge windows, and custom managers for non-standard package files. Renovate's noise reduction features (group, schedule, automerge) are critical for large monorepos. Example: Renovate groups all `@aws-sdk/*` package updates into a single weekly PR, automerges if CI passes, and assigns the platform team for major version bumps requiring review. |
| **Problem Solved** | Prevents dependency rot; ensures security patches are applied before exploitation |
| **Usage** | Configure in `.github/dependabot.yml`; auto-merge patch updates; require review for major versions |
| **Scenarios** | Dependency management (05-supply-chain), SOC 2 patch management evidence |

---

## 6. Secrets Management & Encryption

### Secrets Providers

#### HashiCorp Vault

| Attribute | Detail |
|---|---|
| **What** | Centralized secrets management with dynamic secret generation, auto-rotation, encryption-as-a-service, and audit logging |
| **Problem Solved** | Eliminates static credentials; generates short-lived database credentials on demand |
| **Usage** | Database secrets engine: `vault read database/creds/app-role` returns unique user/pass with 1h TTL |
| **Scenarios** | Dynamic secrets (06-secrets-mgmt), cross-cloud secrets (02-multi-cloud-scenarios), Zero Trust token management |

#### AWS Secrets Manager

| Attribute | Detail |
|---|---|
| **What** | AWS-managed secrets store with automatic rotation via Lambda, RDS integration, and cross-region replication |
| **Problem Solved** | Automated credential rotation; native RDS password rotation without application changes |
| **Usage** | Secret + rotation Lambda on 30-day schedule; SDK retrieves current version; application uses cached credentials |
| **Scenarios** | RDS credential rotation (06-secrets-mgmt), secure multi-tier secrets (01-secure-multi-tier) |

#### Azure Key Vault

| Attribute | Detail |
|---|---|
| **What** | Azure-managed secrets, keys, and certificates with RBAC, HSM-backed keys, and Private Endpoint support |
| **Problem Solved** | Centralized secret storage for Azure workloads; HSM-backed keys for compliance (FIPS 140-2 L2) |
| **Usage** | Access via Managed Identity; Private Endpoint for VNet-only access; Key Vault references in App Service |
| **Scenarios** | Azure secrets management (03-azure-security), HIPAA key management (08-compliance) |

---

### Kubernetes Secrets Delivery

#### External Secrets Operator (ESO)

| Attribute | Detail |
|---|---|
| **What** | Kubernetes operator syncing secrets from external providers (Vault, AWS SM, Azure KV, GCP SM) into K8s Secrets |
| **Problem Solved** | Single source of truth for secrets outside K8s; automatic sync and refresh; no secrets in Git |
| **Usage** | `ExternalSecret` CRD references SecretStore → syncs to K8s Secret → mounted as volume or env var |
| **Scenarios** | Multi-cloud secrets sync (06-secrets-mgmt), ESO with Vault and AWS SM |

#### Sealed Secrets

| Attribute | Detail |
|---|---|
| **What** | Bitnami controller encrypting secrets client-side using cluster's public key; only the cluster can decrypt |
| **Problem Solved** | Enables secrets in Git (GitOps-safe); encrypted at rest in repository |
| **Usage** | `kubeseal` encrypts → commit `SealedSecret` to Git → controller decrypts → creates K8s Secret |
| **Scenarios** | GitOps secrets (06-secrets-mgmt), ArgoCD-compatible secret management |

#### CSI Secrets Store Driver

| Attribute | Detail |
|---|---|
| **What** | Kubernetes CSI (Container Storage Interface) driver that mounts secrets from external providers (Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager) directly as tmpfs volumes in pods.<br/><br/>Secrets are fetched from the provider at pod start and mounted as files — bypassing the K8s Secret object entirely, so secrets never transit etcd. Supports optional rotation (re-fetching secrets periodically) and optional sync-as-K8s-Secret for workloads that require environment variables instead of files. Each provider has a dedicated SecretProviderClass driver plugin. Example: a `SecretProviderClass` references Azure Key Vault, specifying secret names `db-password` and `api-key`; the pod mounts them as files at `/mnt/secrets/db-password` and `/mnt/secrets/api-key`, refreshed every 2 minutes. |
| **Problem Solved** | Bypasses K8s Secret object entirely; secrets never stored in etcd; direct provider-to-pod delivery |
| **Usage** | `SecretProviderClass` CRD specifies provider + secret paths; pod mounts via CSI volume |
| **Scenarios** | Direct secret mounting (06-secrets-mgmt), high-security secret delivery |

---

### Encryption & Key Management

#### AWS KMS (Key Management Service)

| Attribute | Detail |
|---|---|
| **What** | Managed key service for envelope encryption; integrates with S3, EBS, RDS, Secrets Manager, and Lambda |
| **Problem Solved** | Centralized key management; key policies restrict which IAM principals can encrypt/decrypt |
| **Usage** | CMK with key policy → S3 SSE-KMS; Secrets Manager uses KMS for secret encryption at rest |
| **Scenarios** | PCI encryption at rest (08-compliance), data residency pseudonymization (02-multi-cloud-scenarios) |

#### etcd Encryption at Rest

| Attribute | Detail |
|---|---|
| **What** | Kubernetes API server `EncryptionConfiguration` encrypting Secret resources in etcd using AES-CBC or KMS providers |
| **Problem Solved** | Protects secrets if etcd backup is compromised; required for compliance (PCI, SOC 2) |
| **Usage** | KMS provider preferred: `EncryptionConfiguration → kmsv2 provider → AWS/Azure/GCP KMS` |
| **Scenarios** | K8s secrets at rest (06-secrets-mgmt), PCI etcd protection (08-compliance) |

---

## 7. Observability & Monitoring

### Logging & Auditing

#### VPC Flow Logs

| Attribute | Detail |
|---|---|
| **What** | AWS network-level logs capturing source/dest IP, port, protocol, bytes, and accept/reject actions for all ENIs |
| **Problem Solved** | Network forensics; detects lateral movement, data exfiltration patterns, and unauthorized traffic flows |
| **Usage** | Enable on VPC/subnet; send to CloudWatch Logs or S3; Athena queries for investigation |
| **Scenarios** | S3 exfiltration detection (02-aws-security), PCI network monitoring (08-compliance), HA troubleshooting (05) |

#### Kubernetes Audit Logs

| Attribute | Detail |
|---|---|
| **What** | API server logs recording every K8s API request with timestamp, user, verb, resource, and response code |
| **Problem Solved** | Detects unauthorized K8s operations: secret reads, RBAC escalations, exec into pods |
| **Usage** | Configure audit policy levels (Metadata, Request, RequestResponse); send to SIEM for alerting |
| **Scenarios** | Zero Trust audit (02-zero-trust-design), SOC 2 K8s evidence (08-compliance) |

#### NSG Flow Logs

| Attribute | Detail |
|---|---|
| **What** | Azure-native network traffic logs captured at the NSG level, recording 5-tuple flow information (src/dst IP, src/dst port, protocol), byte counts, packet counts, and the NSG rule that matched each flow.<br/><br/>Version 2 format adds additional fields: flow state tracking (begin/continuing/end), throughput in bytes/packets per interval, and encryption status. Logs are stored in Storage Accounts with configurable retention (1-365 days). Traffic Analytics (built on Log Analytics workspace) provides geo-visualization, traffic hotspot detection, and anomaly identification. Example: during a lateral movement investigation, Traffic Analytics reveals a compromised VM sending 50GB to an internal database server over port 1433 at 3 AM — flagged as anomalous by the baseline model. |
| **Problem Solved** | Azure network forensics; Traffic Analytics provides visualization and anomaly detection |
| **Usage** | Enable on NSG → Storage Account → Traffic Analytics workspace → Sentinel integration |
| **Scenarios** | Azure incident investigation (03-azure-security), VM quarantine validation |

---

### Metrics & Tracing

#### Prometheus / Grafana

| Attribute | Detail |
|---|---|
| **What** | Prometheus: pull-based time-series metrics collection. Grafana: visualization and alerting dashboard |
| **Problem Solved** | Real-time infrastructure and application monitoring; capacity planning and SLO tracking |
| **Usage** | ServiceMonitor CRDs for auto-discovery; Grafana dashboards for API gateway latency, error rates, saturation |
| **Scenarios** | API gateway observability (06-api-gateway), Istio mesh metrics (02-zero-trust-design), HA monitoring (05) |

#### Jaeger / OpenTelemetry

| Attribute | Detail |
|---|---|
| **What** | Jaeger: distributed tracing backend. OpenTelemetry: vendor-neutral instrumentation SDK and collector |
| **Problem Solved** | End-to-end request tracing across microservices; identifies latency bottlenecks and failure points |
| **Usage** | OTel SDK auto-instruments apps; OTel Collector routes traces to Jaeger; correlate with Prometheus metrics |
| **Scenarios** | API request tracing (06-api-gateway), service mesh observability (02-zero-trust-design) |

#### Hubble (Cilium)

| Attribute | Detail |
|---|---|
| **What** | eBPF-based network observability for Kubernetes providing flow visibility, DNS inspection, and HTTP metrics |
| **Problem Solved** | Network-level observability without proxy overhead; sees traffic that service meshes miss |
| **Usage** | `hubble observe --namespace payments --verdict DROPPED` for real-time network policy debugging |
| **Scenarios** | Cilium observability (04-container-security), NetworkPolicy validation (02-zero-trust-design) |

---

### SIEM & SOAR

#### Amazon EventBridge

| Attribute | Detail |
|---|---|
| **What** | Serverless event bus routing AWS service events to Lambda, SNS, SQS, or Step Functions based on rules |
| **Problem Solved** | Automated incident response: GuardDuty finding → EventBridge rule → Lambda quarantines resource |
| **Usage** | Rule: `source=aws.guardduty, detail-type=finding, severity>=7 → invoke containment Lambda` |
| **Scenarios** | Automated threat response (02-aws-security), S3 exfiltration containment (02), EC2 quarantine |

#### AWS Step Functions

| Attribute | Detail |
|---|---|
| **What** | Serverless orchestration service for multi-step workflows with retry, parallel execution, and error handling |
| **Problem Solved** | Complex multi-step incident response and failover orchestration without custom orchestration code |
| **Usage** | Failover workflow: check health → promote Aurora replica → update Route 53 → validate → notify |
| **Scenarios** | Multi-region failover automation (03-multi-region-failover), incident response orchestration |

---

## 8. Platform Engineering & IaC

### GitOps & Deployment

#### ArgoCD

| Attribute | Detail |
|---|---|
| **What** | Kubernetes-native GitOps continuous delivery tool that syncs cluster state from Git repositories |
| **Problem Solved** | Declarative deployment; drift detection and auto-remediation; fleet-wide rollouts across clusters |
| **Usage** | ApplicationSet generator for multi-cluster deployment; sync from Git → K8s with automatic pruning |
| **Scenarios** | Multi-cloud fleet management (02-multi-cloud-scenarios), GitOps deployment (05-supply-chain) |

#### Flux CD

| Attribute | Detail |
|---|---|
| **What** | CNCF-graduated GitOps toolkit providing a set of composable controllers (Source Controller, Kustomize Controller, Helm Controller, Notification Controller, Image Automation Controller) for progressive delivery.<br/><br/>Unlike ArgoCD's single-binary approach, Flux's modular architecture lets you adopt only the controllers you need. The Notification Controller dispatches alerts to Slack/Teams/PagerDuty on reconciliation failures. Image Automation Controller watches container registries for new tags matching a policy (e.g., semver `>=1.0.0 <2.0.0`) and auto-commits updated image references to Git. Supports multi-tenancy via RBAC-scoped `Kustomization` resources per team namespace. Example: Flux watches a Helm chart in an OCI registry, auto-updates the Git manifest when a new patch version is published, then reconciles the HelmRelease in the cluster — fully automated CD without any CI pipeline trigger. |
| **Problem Solved** | Pull-based deployment; Helm chart reconciliation; image automation for CD pipelines |
| **Usage** | `GitRepository` + `Kustomization` CRDs; automated image updates via ImagePolicy |
| **Scenarios** | GitOps alternative to ArgoCD (05-supply-chain), Helm-based deployments |

---

### Infrastructure as Code

#### Terraform / OpenTofu

| Attribute | Detail |
|---|---|
| **What** | Declarative IaC tool provisioning cloud infrastructure via HCL; OpenTofu is the open-source fork |
| **Problem Solved** | Reproducible infrastructure; state tracking; plan/apply workflow for safe changes |
| **Usage** | Define VPC, subnets, security groups, EKS clusters; `terraform plan` before `apply` |
| **Scenarios** | Multi-cloud infrastructure (01-multi-cloud-patterns), PCI CDE deployment (08-compliance) |

#### Crossplane (XRDs)

| Attribute | Detail |
|---|---|
| **What** | Kubernetes-based IaC using Composite Resource Definitions (XRDs) to abstract multi-cloud provisioning |
| **Problem Solved** | Platform teams define composites; development teams consume cloud resources via `kubectl` |
| **Usage** | `CompositeResourceDefinition: DatabaseClaim` → provisions RDS on AWS or Azure SQL on Azure |
| **Scenarios** | Unified platform management (02-multi-cloud-scenarios), self-service infrastructure |

---

### Data Replication & Databases

#### Aurora Global Database

| Attribute | Detail |
|---|---|
| **What** | AWS multi-region database with <1 second cross-region replication and managed failover |
| **Problem Solved** | RPO <1 second for multi-region DR; secondary regions provide read-scale and automatic promotion |
| **Usage** | Primary writes in us-east-1; secondary in eu-west-1; failover via `failover-global-cluster` API |
| **Scenarios** | Multi-region failover (03-multi-region-failover), PCI data redundancy (08-compliance) |

#### Debezium + Kafka (CDC)

| Attribute | Detail |
|---|---|
| **What** | Debezium: Change Data Capture from databases to Kafka. Kafka: distributed event streaming platform |
| **Problem Solved** | Low-RPO cross-cloud data replication without application changes; captures row-level changes |
| **Usage** | Debezium connector reads MySQL/Postgres WAL → publishes to Kafka topic → consumer writes to target DB |
| **Scenarios** | Cross-cloud DR replication (02-multi-cloud-scenarios), data residency pipeline |

#### AWS Glue

| Attribute | Detail |
|---|---|
| **What** | Serverless ETL service for data transformation, cataloging, and cross-region analytics |
| **Problem Solved** | Data pseudonymization for cross-border compliance; tokenizes PII before cross-region transfer |
| **Usage** | Glue job reads from source → KMS-based tokenization → writes pseudonymized data to analytics region |
| **Scenarios** | Data residency compliance (02-multi-cloud-scenarios), GDPR pseudonymization pipeline |

---

## 9. API Gateway & Traffic Management

#### API Gateway Patterns (Edge / BFF / Sidecar)

| Attribute | Detail |
|---|---|
| **What** | Three-tier gateway architecture: Edge Gateway (external), BFF Gateway (client-specific), Sidecar Proxy (per-service) |
| **Problem Solved** | Separates concerns: Edge handles DDoS/auth, BFF handles client optimization, Sidecar handles mTLS/retry |
| **Usage** | Kong/AWS API GW at edge → BFF per client type (mobile/web) → Envoy sidecar per microservice |
| **Scenarios** | API gateway design (06-api-gateway), multi-tier routing architecture |

#### Rate Limiting & Throttling

| Attribute | Detail |
|---|---|
| **What** | Request rate control using token bucket, sliding window, or fixed window algorithms |
| **Problem Solved** | Prevents resource exhaustion (OWASP API4); protects backend services from abuse |
| **Usage** | Tiered limits: free (100 RPM), basic (1K RPM), enterprise (10K RPM); Redis-backed distributed counter |
| **Scenarios** | API resource consumption (07-owasp-api), API gateway throttling (06-api-gateway) |

#### Circuit Breaker

| Attribute | Detail |
|---|---|
| **What** | Resilience pattern for distributed systems with three states: Closed (requests flow normally, failures are counted), Open (requests are immediately rejected/fail-fast for a cooldown period), and Half-Open (limited probe requests test if the backend has recovered).<br/><br/>In Envoy/Istio, this is implemented via outlier detection: if an upstream host returns 5 consecutive 5xx errors, it is ejected from the load balancing pool for 30 seconds (configurable). Istio's `DestinationRule` supports both consecutive errors and success-rate-based ejection. Unlike client-library approaches (Hystrix — now deprecated), service mesh circuit breaking is transparent to application code and applies uniformly across all services. Example: an `DestinationRule` ejects any backend pod returning >10% errors over a 10-second window for 60 seconds, with max 30% of pods ejectable to prevent full service unavailability. |
| **Problem Solved** | Prevents cascade failures; stops requests to failing backends; enables graceful degradation |
| **Usage** | Envoy circuit breaker: 5 consecutive 5xx → open for 30s → half-open probe → reset on success |
| **Scenarios** | API gateway resilience (06-api-gateway), HA networking (05-high-availability) |

#### JWT / OAuth2 / mTLS Authentication

| Attribute | Detail |
|---|---|
| **What** | JWT: stateless bearer tokens. OAuth2: authorization framework. mTLS: mutual TLS with client certificates |
| **Problem Solved** | API authentication at the gateway layer; JWT for external clients, mTLS for service-to-service |
| **Usage** | Gateway validates JWT signature + claims (iss, aud, exp); extracts user context; passes to backend |
| **Scenarios** | Broken Auth prevention (07-owasp-api), Zero Trust authentication (01-zero-trust), API security |

---

## 10. Compliance & Governance

#### PCI-DSS v4.0

| Attribute | Detail |
|---|---|
| **What** | Payment card industry standard with 12 requirements covering network segmentation, encryption, access control, monitoring |
| **Problem Solved** | Protects cardholder data (CHD) and sensitive authentication data (SAD); required for payment processing |
| **Key Controls** | CDE network isolation (Transit GW), SG chaining, encryption at rest (KMS) and in transit (TLS 1.2+), CloudTrail logging |
| **Scenarios** | Secure multi-tier PCI architecture (01-secure-multi-tier), compliance automation (08-compliance) |

#### HIPAA

| Attribute | Detail |
|---|---|
| **What** | Healthcare data protection law governing ePHI with Security Rule, Privacy Rule, and Breach Notification |
| **Problem Solved** | Protects electronic Protected Health Information; requires encryption, access control, audit trails |
| **Key Controls** | VPC isolation, KMS encryption, Private Endpoints, MFA, audit logging, BAA with cloud providers |
| **Scenarios** | HIPAA compliance mapping (08-compliance), healthcare multi-tier architecture |

#### SOC 2 Type II

| Attribute | Detail |
|---|---|
| **What** | AICPA framework evaluating controls over 6-12 months across Trust Service Criteria: Security, Availability, Confidentiality |
| **Problem Solved** | Proves operational security to enterprise customers; continuous evidence vs. point-in-time |
| **Key Controls** | IAM Access Analyzer, Config rules, GuardDuty, quarterly access reviews, change management evidence |
| **Scenarios** | SOC 2 evidence collection (08-compliance), continuous compliance automation |

#### NIST 800-53

| Attribute | Detail |
|---|---|
| **What** | Comprehensive security and privacy control catalog with 20 families; basis for FedRAMP and many frameworks |
| **Problem Solved** | Risk-based security control selection; maps to nearly all other compliance frameworks |
| **Key Controls** | AC (Access Control), AU (Audit), CM (Configuration Management), IA (Identification), SC (System Communications) |
| **Scenarios** | Control family mapping (08-compliance), GovCloud requirements |

#### OWASP API Security Top 10

| Attribute | Detail |
|---|---|
| **What** | Top 10 API-specific security risks: BOLA, Broken Auth, BOPLA, Resource Consumption, BFLA, SSRF, Misconfiguration |
| **Problem Solved** | Identifies most critical API attack vectors; guides API security testing and architecture |
| **Key Controls** | Object-level authorization, rate limiting, field-level ACLs, SSRF allowlists, API inventory management |
| **Scenarios** | API security design (07-owasp-api), API gateway security controls (06-api-gateway) |

---

> **Source Files:** This cheatsheet synthesizes information from 18 markdown files across `03-cloud-networking/multi-cloud/`, `06-cloud-security/`, and `08-system-design/` directories.

