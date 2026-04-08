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
| **What** | Isolated virtual network in AWS with custom CIDR blocks, subnets, route tables, and gateways |
| **Problem Solved** | Network isolation for workloads; prevents uncontrolled access between environments |
| **Usage** | Define CIDR (e.g., `10.0.0.0/16`), create public/private subnets across AZs, attach IGW or NAT GW |
| **Scenarios** | Secure multi-tier architecture (01), PCI-DSS CDE isolation (08), multi-region failover (03) |

#### Security Groups

| Attribute | Detail |
|---|---|
| **What** | Stateful instance-level firewalls with ALLOW-only rules; referenced by ID for tier chaining |
| **Problem Solved** | Network-level access control without hardcoded IPs; auto-scales with ASG membership |
| **Usage** | `ALB-SG → App-SG (allow 8080 from ALB-SG) → DB-SG (allow 5432 from App-SG)` |
| **Scenarios** | Three-tier SG chaining (02-aws-security), PCI CDE segmentation (08-compliance), HA networking (05) |

#### NACLs (Network Access Control Lists)

| Attribute | Detail |
|---|---|
| **What** | Stateless subnet-level ACLs with explicit ALLOW and DENY rules evaluated in numbered order |
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
| **What** | Regional hub-and-spoke network connector supporting VPC peering, VPN, and Direct Connect attachments |
| **Problem Solved** | Eliminates N×N VPC peering; centralizes routing and firewall inspection between VPCs |
| **Usage** | Attach CDE and non-CDE VPCs; route inter-VPC traffic through Transit Gateway Firewall for L7 inspection |
| **Scenarios** | PCI CDE-to-non-CDE segmentation (08-compliance), multi-account networking (01-multi-cloud-patterns) |

#### AWS Direct Connect

| Attribute | Detail |
|---|---|
| **What** | Dedicated private network connection from on-premise to AWS (1 Gbps / 10 Gbps / 100 Gbps) |
| **Problem Solved** | Consistent latency and bandwidth for hybrid workloads; avoids internet for sensitive traffic |
| **Usage** | Provision via Equinix/Megaport interconnect fabric; pair with VPN as backup |
| **Scenarios** | M&A network integration (02-multi-cloud-scenarios), hybrid connectivity (01-multi-cloud-patterns) |

---

### Azure Networking

#### NSGs (Network Security Groups)

| Attribute | Detail |
|---|---|
| **What** | Stateful network filtering at subnet and NIC level with priority-ordered rules |
| **Problem Solved** | Subnet-level traffic control; flow logging for forensics and anomaly detection |
| **Usage** | Apply to subnets; enable NSG Flow Logs → Storage Account → Traffic Analytics → Sentinel |
| **Scenarios** | Lateral movement containment (03-azure-security), VM quarantine during incident response |

#### Azure Firewall

| Attribute | Detail |
|---|---|
| **What** | Managed L7 FQDN-filtering firewall with threat intelligence integration |
| **Problem Solved** | Centralized egress filtering; blocks known-malicious domains at the network edge |
| **Usage** | Deploy in hub VNet; route spoke traffic through firewall; apply FQDN rules |
| **Scenarios** | Azure network plane security (03-azure-security), CDE egress control |

#### Azure Private Endpoints

| Attribute | Detail |
|---|---|
| **What** | Private IP addresses for PaaS services (Key Vault, Storage, SQL) within your VNet |
| **Problem Solved** | Removes public endpoints from PaaS services; traffic stays on Azure backbone |
| **Usage** | Deploy Private Endpoint for Key Vault; disable public access; access via private IP only |
| **Scenarios** | Key Vault without public endpoint (03-azure-security), HIPAA ePHI protection |

#### Azure ExpressRoute

| Attribute | Detail |
|---|---|
| **What** | Dedicated private connection from on-premise or other clouds to Azure (up to 100 Gbps) |
| **Problem Solved** | Private, low-latency connectivity; required for compliance workloads avoiding public internet |
| **Usage** | Provision via Equinix/Megaport; peer with Azure VNets via ExpressRoute Gateway |
| **Scenarios** | M&A Azure integration (02-multi-cloud-scenarios), cross-cloud connectivity (01-multi-cloud-patterns) |

---

### Cross-Cloud & Multi-Cloud Connectivity

#### WireGuard

| Attribute | Detail |
|---|---|
| **What** | Modern, lightweight VPN protocol using Noise Protocol Framework with ~4,000 lines of code |
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
| **What** | Software-defined overlay network with centralized policy and application-aware routing |
| **Problem Solved** | Multi-WAN path selection; automatic failover between internet, MPLS, and cloud circuits |
| **Usage** | Deploy SD-WAN edge appliances in each cloud VPC/VNet; centrally define routing policies |
| **Scenarios** | Multi-cloud overlay networking (01-multi-cloud-patterns), branch-to-cloud connectivity |

#### IPAM (IP Address Management)

| Attribute | Detail |
|---|---|
| **What** | Centralized IP address allocation strategy preventing CIDR overlaps across clouds |
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
| **What** | AWS L7 load balancer supporting path/host-based routing, WebSocket, gRPC, and WAF integration |
| **Problem Solved** | Distributes HTTP/HTTPS traffic across targets with content-based routing |
| **Usage** | Target groups per microservice; health checks; WAF association for L7 filtering |
| **Scenarios** | Multi-tier architecture (01-secure-multi-tier), API gateway frontend (06-api-gateway) |

#### NLB (Network Load Balancer)

| Attribute | Detail |
|---|---|
| **What** | AWS L4 load balancer with static IPs, ultra-low latency, and millions of RPS capacity |
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
| **What** | Identifies resources accessible from outside your AWS Organization; detects unused permissions |
| **Problem Solved** | Finds accidental public exposure; generates least-privilege policies from CloudTrail activity |
| **Usage** | `aws iam generate-service-last-accessed-details` → generates minimum-privilege policy from 90 days of activity |
| **Scenarios** | Least privilege enforcement (02-aws-security), SOC 2 quarterly access reviews (08-compliance) |

#### IRSA (IAM Roles for Service Accounts)

| Attribute | Detail |
|---|---|
| **What** | Maps Kubernetes ServiceAccounts to AWS IAM Roles via OIDC federation — no static credentials |
| **Problem Solved** | Eliminates AWS access keys in pods; per-pod IAM role scoping |
| **Usage** | Annotate SA with `eks.amazonaws.com/role-arn`; pod assumes role via projected token |
| **Scenarios** | ESO accessing Secrets Manager (06-secrets-mgmt), Zero Trust workload identity (01-zero-trust) |

#### STS (Security Token Service)

| Attribute | Detail |
|---|---|
| **What** | Issues temporary security credentials for assuming roles and cross-account access |
| **Problem Solved** | Short-lived credentials reduce blast radius of compromise; enables cross-account patterns |
| **Usage** | `aws sts assume-role` with short TTL; session policies limit assumed role permissions further |
| **Scenarios** | Cross-account access (02-aws-security), incident response credential rotation |

#### SCPs (Service Control Policies)

| Attribute | Detail |
|---|---|
| **What** | Organization-level guardrails that override account-level IAM; cannot be bypassed by account admins |
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
| **What** | Auto-managed identity for Azure resources; certificate rotated automatically by Azure — no static credentials |
| **Problem Solved** | Eliminates client secrets and API keys for Azure service-to-service authentication |
| **Usage** | VM/App Service calls IMDS (`169.254.169.254/metadata`) → receives short-lived access token → accesses Key Vault |
| **Scenarios** | AKS Workload Identity (03-azure-security), Key Vault access without secrets |

#### Azure PIM (Privileged Identity Management)

| Attribute | Detail |
|---|---|
| **What** | Just-in-time role elevation with time limits, justification, and approval workflows |
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
| **What** | OIDC federation mapping Kubernetes ServiceAccounts to Azure Managed Identities |
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
| **What** | Records resource configuration changes over time; evaluates against managed and custom rules |
| **Problem Solved** | Compliance drift detection; automated remediation via SSM Automation when rules are violated |
| **Usage** | PCI Conformance Pack: `restricted-ssh`, `vpc-flow-logs-enabled`, `cloud-trail-enabled` + auto-remediation |
| **Scenarios** | PCI compliance automation (08-compliance), CDE SG monitoring (08-compliance), SOC 2 evidence |

#### AWS Macie

| Attribute | Detail |
|---|---|
| **What** | ML-powered S3 data classification service detecting PII, PHI, and sensitive data |
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
| **What** | Unified CSPM + CWP across Azure, AWS, and GCP; Secure Score, attack path analysis, and regulatory compliance |
| **Problem Solved** | Multi-cloud security posture visibility; prioritized remediation recommendations |
| **Usage** | Enable CSPM + workload protection plans; review Secure Score; use attack path analysis for exposure |
| **Scenarios** | Azure posture management (03-azure-security), multi-cloud CSPM (03-azure-security) |

#### Microsoft Sentinel

| Attribute | Detail |
|---|---|
| **What** | Cloud-native SIEM + SOAR with 200+ data connectors, KQL analytics rules, and Logic App playbooks |
| **Problem Solved** | Threat detection, investigation, and automated response across Azure, AWS, M365, and third-party |
| **Usage** | KQL analytics rules for impossible travel; Playbooks auto-disable user + revoke tokens + isolate VM |
| **Scenarios** | Lateral movement containment (03-azure-security), impossible travel detection, SOC operations |

#### Azure Policy

| Attribute | Detail |
|---|---|
| **What** | ARM-level policy enforcement with effects: Audit, Deny, Modify, DeployIfNotExists |
| **Problem Solved** | Prevents non-compliant resource creation; auto-deploys required configurations |
| **Usage** | HIPAA/HITRUST initiative applied at subscription scope; AKS security baseline initiative |
| **Scenarios** | HIPAA compliance (08-compliance), AKS security enforcement (03-azure-security) |

---

### Web Application Security

#### AWS WAF

| Attribute | Detail |
|---|---|
| **What** | L7 web application firewall with managed rule groups (OWASP, Bot Control) and custom rules |
| **Problem Solved** | Blocks SQL injection, XSS, SSRF patterns, bot traffic, and debug endpoint access |
| **Usage** | Associate with ALB/CloudFront; managed OWASP Core Rule Set + custom rules for `/actuator/*` blocking |
| **Scenarios** | CDN edge security (04-cdn-edge), API security misconfiguration (07-owasp-api), business flow abuse |

#### OPA (Open Policy Agent) / Cedar

| Attribute | Detail |
|---|---|
| **What** | General-purpose policy engine (Rego language) for authorization decisions across API, K8s, and infrastructure |
| **Problem Solved** | Centralized default-deny authorization; replaces scattered `if (user.role == "admin")` checks |
| **Usage** | Maps routes + methods to required roles; in-process SDK for sub-ms evaluation at 10K+ RPS |
| **Scenarios** | BFLA prevention (07-owasp-api), Zero Trust policy enforcement (01-zero-trust), PCI RBAC (08-compliance) |

---

### DDoS Protection

#### AWS Shield (Standard & Advanced)

| Attribute | Detail |
|---|---|
| **What** | DDoS protection: Standard (free, auto L3/L4) and Advanced ($3K/mo, L7, cost protection, SRT access) |
| **Problem Solved** | Absorbs volumetric DDoS at the edge; Advanced provides real-time attack visibility and response team |
| **Usage** | Shield Advanced on CloudFront + ALB; DDoS Response Team (SRT) for active attacks |
| **Scenarios** | CDN DDoS resilience (04-cdn-edge), HA networking (05-high-availability) |

#### Azure DDoS Protection

| Attribute | Detail |
|---|---|
| **What** | Managed DDoS mitigation with adaptive tuning and integration with Azure Monitor |
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
| **What** | eBPF-based security observability and runtime enforcement from Cilium project |
| **Problem Solved** | Kernel-level process, file, and network monitoring with enforcement (kill, signal) — not just alerting |
| **Usage** | TracingPolicy CRD defines: match on binary + enforce action (sigkill); blocks cryptominer execution |
| **Scenarios** | eBPF runtime enforcement (04-container-security), advanced threat response |

#### seccomp

| Attribute | Detail |
|---|---|
| **What** | Linux kernel feature restricting which syscalls a process can make |
| **Problem Solved** | Prevents container escape via dangerous syscalls like `unshare`, `mount`, `ptrace` |
| **Usage** | `RuntimeDefault` profile blocks ~44 dangerous syscalls; custom profiles for app-specific allowlists |
| **Scenarios** | Pod hardening (04-container-security), PCI container runtime controls (08-compliance) |

#### AppArmor

| Attribute | Detail |
|---|---|
| **What** | Linux security module enforcing per-program profiles restricting file, network, and capability access |
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
| **What** | Built-in Kubernetes admission controller enforcing Pod Security Standards at namespace level |
| **Problem Solved** | Baseline security without external tools; three levels: Privileged, Baseline, Restricted |
| **Usage** | Label namespace: `pod-security.kubernetes.io/enforce: restricted` |
| **Scenarios** | Namespace security baseline (04-container-security), PCI pod hardening (08-compliance) |

---

### Pod Security

#### Kubernetes NetworkPolicy

| Attribute | Detail |
|---|---|
| **What** | L3/L4 network access control between pods using label selectors |
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
| **What** | Framework for securing the entire software supply chain by signing each step and verifying the chain |
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
| **What** | Automated dependency update tools creating PRs when new versions or security patches are available |
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
| **What** | Kubernetes CSI driver mounting secrets from external providers (Vault, AWS SM, Azure KV) directly as volumes |
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
| **What** | Azure NSG-level traffic logs capturing 5-tuple flows, byte counts, and rule evaluation results |
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
| **What** | CNCF GitOps toolkit with Kustomization and HelmRelease controllers for progressive delivery |
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
| **What** | Resilience pattern with three states: Closed (normal), Open (fail-fast), Half-Open (probe) |
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

