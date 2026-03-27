# Networking + Security Knowledge Base — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a production-grade, 10+ years SRE-level networking + security knowledge base by synthesizing three existing documentation repos, filling critical gaps, and organizing content for progressive mastery.

**Architecture:** Content organized into 11 themed sections with a strict progression from fundamentals → Linux → Cloud → Kubernetes → Security → Debugging → Design → Advanced → Interview Prep → Labs. Every file integrates real-world scenarios, failure modes, Mermaid diagrams, and security implications. Existing docs are extended and enriched, not rewritten.

**Tech Stack:** Markdown + Mermaid diagrams, organized under `/le-network-security/`

---

## Phase 1: Structure Proposal (AWAITING APPROVAL)

---

## Source Analysis + Gap Matrix

### What Exists

| Source | Files | Strengths | Gaps |
|--------|-------|-----------|------|
| `le-study-notes/networking` | 6 files (DNS, TLS, HTTP/2-3, LB, Traefik, Service Mesh) | Excellent depth on application-layer networking, Kubernetes-oriented | Missing: IP fundamentals, TCP internals, Linux kernel stack, BGP/routing, NAT, cloud-native VPC architecture |
| `le-study-notes/security` | 6 files (OWASP, Container, ZTA, Supply Chain, Kyverno, ESO) | Excellent cloud-native security coverage | Missing: network firewall architecture, DDoS/WAF, VPN technologies, PKI deep dive, compliance frameworks, cloud IAM+network integration |
| `le-interview-prep/networking-linux` | 10 files (kernel internals, performance, troubleshooting, system design, Q&A × 5) | Outstanding kernel internals, Linux networking, interview scenarios, runbooks | Missing: cloud networking Q&A, security scenarios, multi-cloud, IPv6, WireGuard, structured labs |

### Critical Gaps (Must Fill)

| Gap Area | Priority | Notes |
|----------|----------|-------|
| Networking fundamentals (OSI, TCP flags, subnetting, ARP, ICMP) | HIGH | Foundation for everything else |
| AWS VPC deep dive (Transit Gateway, PrivateLink, Direct Connect) | HIGH | Core SRE cloud skill |
| Azure VNet + AKS networking | HIGH | Explicitly required by spec |
| Kubernetes CNI internals (Calico, Cilium, Flannel) | HIGH | Partial in interview prep, needs synthesis |
| Kubernetes NetworkPolicy + Gateway API | HIGH | Missing dedicated treatment |
| eBPF / Cilium comprehensive guide | HIGH | Fragmented across 3 files |
| Debugging playbooks (structured, decision-tree format) | HIGH | Scattered, needs unified playbook section |
| DDoS mitigation + WAF architecture | HIGH | Only mentioned in passing |
| VPN technologies (IPSec, WireGuard) | HIGH | Completely absent |
| Observability for networks (Prometheus, tracing, Grafana) | MEDIUM | Mentioned but no guide |
| NAT exhaustion + ephemeral ports | MEDIUM | In interview prep Q&A only |
| Multi-region failover design | MEDIUM | Only covered abstractly |
| Compliance frameworks (PCI-DSS, SOC 2, HIPAA) | MEDIUM | Missing entirely |
| BGP at scale (route leaks, RPKI) | MEDIUM | In interview prep Q&A only |
| Hands-on labs (Break/Fix) | MEDIUM | No labs exist |
| IPv6 networking | LOW | Mostly absent |

---

## Proposed Folder + File Structure

```
/le-network-security/
│
├── README.md                          ← Master index, learning paths, how to use this repo
│
├── 01-networking-fundamentals/        ← Build the mental model from ground up
│   ├── 01-osi-tcpip-model.md
│   ├── 02-ip-addressing-subnetting.md
│   ├── 03-tcp-udp-deep-dive.md
│   ├── 04-arp-icmp-ndp.md
│   ├── 05-routing-protocols.md
│   ├── 06-nat-pat-ephemeral-ports.md
│   ├── 07-vlan-vxlan-overlay-networks.md
│   └── 08-dns-comprehensive.md
│
├── 02-linux-networking/               ← Kernel-level mastery
│   ├── 01-linux-network-stack.md
│   ├── 02-virtual-networking.md
│   ├── 03-iptables-nftables.md
│   ├── 04-tc-traffic-control.md
│   ├── 05-network-namespaces.md
│   ├── 06-linux-routing-policy.md
│   ├── 07-socket-tuning-performance.md
│   └── 08-ebpf-xdp-comprehensive.md
│
├── 03-cloud-networking/               ← AWS + Azure production patterns
│   ├── aws/
│   │   ├── 01-vpc-fundamentals.md
│   │   ├── 02-vpc-connectivity.md
│   │   ├── 03-aws-load-balancers.md
│   │   ├── 04-route53-dns.md
│   │   └── 05-eks-networking.md
│   ├── azure/
│   │   ├── 01-vnet-fundamentals.md
│   │   ├── 02-azure-connectivity.md
│   │   ├── 03-azure-load-balancers.md
│   │   └── 04-aks-networking.md
│   └── multi-cloud/
│       └── 01-multi-cloud-patterns.md
│
├── 04-kubernetes-networking/          ← K8s networking from pod to mesh
│   ├── 01-k8s-networking-model.md
│   ├── 02-service-types.md
│   ├── 03-ingress-gateway-api.md
│   ├── 04-network-policies.md
│   ├── 05-cni-comparison.md
│   ├── 06-cilium-ebpf.md
│   ├── 07-service-mesh.md
│   └── 08-coredns-externaldns.md
│
├── 05-network-security/               ← Defensive network architecture
│   ├── 01-firewall-architectures.md
│   ├── 02-tls-mtls-pki.md
│   ├── 03-ddos-mitigation.md
│   ├── 04-waf-ids-ips.md
│   ├── 05-vpn-technologies.md
│   └── 06-network-segmentation.md
│
├── 06-cloud-security/                 ← Cloud-native security + compliance
│   ├── 01-zero-trust-architecture.md
│   ├── 02-aws-security.md
│   ├── 03-azure-security.md
│   ├── 04-container-security.md
│   ├── 05-supply-chain-security.md
│   ├── 06-secrets-management.md
│   ├── 07-owasp-api-security.md
│   └── 08-compliance-frameworks.md
│
├── 07-debugging-playbooks/            ← Hypothesis-driven incident response
│   ├── 00-debugging-methodology.md
│   ├── 01-service-not-reachable.md
│   ├── 02-high-latency.md
│   ├── 03-dns-failures.md
│   ├── 04-tls-certificate-issues.md
│   ├── 05-packet-drops.md
│   ├── 06-kubernetes-networking-issues.md
│   ├── 07-cloud-connectivity-issues.md
│   └── 08-ddos-incident-response.md
│
├── 08-system-design/                  ← Architecture + trade-offs at scale
│   ├── 01-secure-multi-tier-architecture.md
│   ├── 02-zero-trust-design.md
│   ├── 03-multi-region-failover.md
│   ├── 04-cdn-edge-architecture.md
│   ├── 05-high-availability-networking.md
│   └── 06-api-gateway-design.md
│
├── 09-advanced-topics/                ← Staff+ and principal-level depth
│   ├── 01-ebpf-internals.md
│   ├── 02-service-mesh-advanced.md
│   ├── 03-network-observability.md
│   ├── 04-performance-engineering-at-scale.md
│   ├── 05-bgp-at-scale.md
│   └── 06-ipv6-production.md
│
├── 10-interview-prep/                 ← Synthesis layer — ready for interviews
│   ├── 00-study-guide.md
│   ├── 01-networking-fundamentals-qa.md
│   ├── 02-linux-networking-qa.md
│   ├── 03-cloud-networking-qa.md
│   ├── 04-kubernetes-networking-qa.md
│   ├── 05-network-security-qa.md
│   ├── 06-system-design-qa.md
│   ├── 07-cross-domain-scenarios.md
│   ├── 08-interview-patterns-pitfalls.md
│   └── 09-exotic-deep-dive-qa.md
│
└── 11-hands-on-labs/                 ← Break/Fix for muscle memory
    ├── 01-linux-networking-lab.md
    ├── 02-kubernetes-networking-lab.md
    ├── 03-aws-vpc-lab.md
    ├── 04-tls-debugging-lab.md
    └── 05-ebpf-xdp-lab.md
```

---

## File-by-File Contents + Learning Progression

### 01 — Networking Fundamentals

**Progression:** Start here if any gaps in fundamentals. Every concept links forward to how it behaves in Linux/Cloud/K8s.

| File | Contents | Source |
|------|----------|--------|
| `01-osi-tcpip-model.md` | Layer-by-layer with real tools per layer (ethtool → ip → ss → curl). How breaks at each layer present in production. | NEW |
| `02-ip-addressing-subnetting.md` | CIDR, subnetting, supernetting, VPC CIDR design trade-offs, IPv6 addressing. RFC1918 and why it matters in cloud. | NEW |
| `03-tcp-udp-deep-dive.md` | TCP flags, state machine, handshake, retransmission, TIME_WAIT/CLOSE_WAIT, congestion control (BBR vs CUBIC). Production failure patterns. | Extends `interview-prep/05_Advanced_Interview_QA.md` |
| `04-arp-icmp-ndp.md` | ARP spoofing, gratuitous ARP, GARP in HA setups, ICMP types in debugging, NDP for IPv6. | NEW |
| `05-routing-protocols.md` | BGP fundamentals → BGP in cloud (AWS TGW, Azure Route Server), OSPF, ECMP, route leaks, RPKI. | Extends `interview-prep/05_Advanced_Interview_QA.md` Q3 |
| `06-nat-pat-ephemeral-ports.md` | SNAT/DNAT/PAT mechanics, conntrack table, NAT exhaustion in production, ephemeral port tuning, cloud NAT. | Extends `interview-prep/05_Advanced_Interview_QA.md` Q1 |
| `07-vlan-vxlan-overlay-networks.md` | 802.1Q VLANs, VXLAN encapsulation, GENEVE, how cloud VPCs use overlays, debugging encapsulation issues. | Extends `interview-prep/05_Advanced_Interview_QA.md` Q9 |
| `08-dns-comprehensive.md` | Full DNS resolution chain, record types, DNSSEC, DoH/DoT, DNS at scale, split-horizon, DNS security attacks (cache poisoning, DNS rebinding). | Extends `le-study-notes/networking/01-dns-basics.md` |

---

### 02 — Linux Networking

**Progression:** Kernel internals → virtual devices → packet filtering → performance. Directly maps to debugging and interview prep.

| File | Contents | Source |
|------|----------|--------|
| `01-linux-network-stack.md` | sk_buff lifecycle, NAPI, softirq budget, ring buffers, NIC → socket full path. Failure modes at each stage. | Extends `interview-prep/01_Architecture_and_Internals.md` |
| `02-virtual-networking.md` | veth pairs, Linux bridge, tun/tap, macvlan, ipvlan, OVS basics. How Docker/K8s use these. | Extends `interview-prep/06_Cross_Domain_Integration.md` |
| `03-iptables-nftables.md` | iptables chains/tables, nftables syntax, migration guide, performance comparison, kube-proxy iptables mode. | Extends `interview-prep/07_Ecosystem_and_Adjacent_Tools.md` Q3 + Q10 |
| `04-tc-traffic-control.md` | qdisc types (pfifo, fq_codel, HTB, TBF), tc commands, bandwidth limiting in practice, CNI use of tc. | Extends `interview-prep/02_Performance_and_Optimization.md` |
| `05-network-namespaces.md` | Namespace creation, veth-pair setup, how containers use them, debugging namespace isolation issues. | Extends `interview-prep/01_Architecture_and_Internals.md` + `09_Exotic_and_Deep_Dive_Questions.md` Q8 |
| `06-linux-routing-policy.md` | ip route, ip rule, multiple routing tables, policy routing, ECMP, OSPF integration. | Extends `interview-prep/07_Ecosystem_and_Adjacent_Tools.md` Q9 |
| `07-socket-tuning-performance.md` | rmem/wmem, tcp_rmem/wmem, window scaling, sysctl reference table, NUMA-aware tuning, benchmarking methodology. | Extends `interview-prep/02_Performance_and_Optimization.md` |
| `08-ebpf-xdp-comprehensive.md` | BPF maps, program types, attach points (XDP, tc, kprobe, uprobe), CO-RE, bpftrace, AF_XDP zero-copy, Cilium eBPF internals. | Extends `interview-prep/09_Exotic_and_Deep_Dive_Questions.md` Q7+Q9 |

---

### 03 — Cloud Networking

**Progression:** AWS first (most common), then Azure, then multi-cloud patterns.

| File | Contents | Source |
|------|----------|--------|
| `aws/01-vpc-fundamentals.md` | VPC design, subnets (public/private/isolated), route tables, Internet Gateway, NAT Gateway, Endpoints (Interface/Gateway). Security Groups vs NACLs. | NEW |
| `aws/02-vpc-connectivity.md` | VPC Peering vs Transit Gateway vs PrivateLink, Direct Connect vs Site-to-Site VPN, overlapping CIDRs, centralized egress. | Extends `interview-prep/06_Cross_Domain_Integration.md` Q3 + Q5 |
| `aws/03-aws-load-balancers.md` | ALB vs NLB vs GLB — when to use each, target group types, connection draining, WAF integration, access logs. | Extends `le-study-notes/networking/04-load-balancing.md` |
| `aws/04-route53-dns.md` | Routing policies, health checks, private hosted zones, resolver inbound/outbound endpoints, DNSSEC, hybrid DNS. | Extends `le-study-notes/networking/01-dns-basics.md` |
| `aws/05-eks-networking.md` | VPC CNI (ENI allocation, prefix mode), CoreDNS in EKS, kube-proxy vs iptables vs IPVS, security groups for pods, AWS Load Balancer Controller. | NEW |
| `azure/01-vnet-fundamentals.md` | VNet address space, subnets, NSGs, ASGs, User-Defined Routes, Azure DNS. | NEW |
| `azure/02-azure-connectivity.md` | VNet Peering vs Virtual WAN vs PrivateLink, ExpressRoute vs VPN Gateway, Azure Route Server, hub-spoke. | NEW |
| `azure/03-azure-load-balancers.md` | Azure Load Balancer (L4) vs Application Gateway (L7) vs Front Door vs Traffic Manager. | NEW |
| `azure/04-aks-networking.md` | kubenet vs Azure CNI vs Azure CNI Overlay, Calico on AKS, Azure Network Policy, Ingress (AGIC). | NEW |
| `multi-cloud/01-multi-cloud-patterns.md` | Cross-cloud connectivity options, DNS federation, latency trade-offs, security perimeter in multi-cloud. | NEW |

---

### 04 — Kubernetes Networking

**Progression:** Understand the model → services → ingress → policies → CNI internals → mesh.

| File | Contents | Source |
|------|----------|--------|
| `01-k8s-networking-model.md` | Pod networking (veth + bridge/eBPF), the 3 K8s networking rules, kube-proxy modes, cluster CIDR, pod CIDR. | Extends `interview-prep/06_Cross_Domain_Integration.md` Q1 |
| `02-service-types.md` | ClusterIP (iptables/IPVS), NodePort, LoadBalancer, ExternalName, headless services, endpoints vs endpointslices. | NEW |
| `03-ingress-gateway-api.md` | Ingress spec, NGINX vs Traefik vs Istio ingress, Gateway API (GatewayClass/Gateway/HTTPRoute), migration path. | Extends `le-study-notes/networking/05-traefik.md` |
| `04-network-policies.md` | Default-deny pattern, ingress/egress policies, Calico GlobalNetworkPolicy, Cilium L7 policies, testing with `kubectl exec`. | Extends `interview-prep/06_Cross_Domain_Integration.md` Q8 |
| `05-cni-comparison.md` | Flannel (VXLAN), Calico (BGP/VXLAN/eBPF), Cilium (eBPF-native), Weave — feature matrix, when to choose each. | Extends `interview-prep/07_Ecosystem_and_Adjacent_Tools.md` Q10 |
| `06-cilium-ebpf.md` | Cilium architecture (agent, operator, Hubble), eBPF datapath, Cilium Network Policy, Hubble observability, Tetragon. | Extends `le-study-notes/networking/06-service-mesh.md` |
| `07-service-mesh.md` | Istio architecture (istiod, envoy sidecar, ambient), Linkerd, SPIFFE/SPIRE, mTLS enforcement, traffic management. | Extends `le-study-notes/networking/06-service-mesh.md` |
| `08-coredns-externaldns.md` | CoreDNS config/plugins, ndots pitfall, stub zones, ExternalDNS providers, DNS-based service discovery. | Extends `interview-prep/07_Ecosystem_and_Adjacent_Tools.md` Q7 |

---

### 05 — Network Security

**Progression:** Defense in depth — from perimeter to protocol to key management.

| File | Contents | Source |
|------|----------|--------|
| `01-firewall-architectures.md` | Stateless vs stateful, next-gen firewalls, east-west vs north-south segmentation, AWS Security Groups, Azure NSGs, iptables vs nftables, cloud-native firewall patterns. | NEW |
| `02-tls-mtls-pki.md` | TLS 1.3 handshake, cipher suites, mTLS for zero-trust, PKI hierarchy, cert-manager, ACM, OCSP stapling, certificate transparency. | Extends `le-study-notes/networking/02-tls-ssl.md` |
| `03-ddos-mitigation.md` | DDoS taxonomy (volumetric, protocol, application), Cloudflare/AWS Shield, BGP blackholing, SYN cookies, rate limiting at each layer, scrubbing centers. | Extends `interview-prep/05_Advanced_Interview_QA.md` Q13 |
| `04-waf-ids-ips.md` | WAF modes (detection/prevention), OWASP ModSecurity ruleset, AWS WAF, Azure WAF, custom rules, Snort/Suricata IDS, bypass techniques and mitigations. | NEW |
| `05-vpn-technologies.md` | IPSec (IKEv2, ESP/AH), WireGuard (Noise protocol, kernel module), OpenVPN, SSL-VPN, always-on VPN for zero-trust, performance comparison. | NEW |
| `06-network-segmentation.md` | Micro-segmentation vs macro, VLAN-based, firewall-based, K8s NetworkPolicy, zero-trust network access (ZTNA), practical segmentation for cloud environments. | NEW |

---

### 06 — Cloud Security

**Progression:** Zero trust principles → cloud-specific implementations → runtime → supply chain → compliance.

| File | Contents | Source |
|------|----------|--------|
| `01-zero-trust-architecture.md` | NIST SP 800-207 tenets, BeyondCorp model, SPIFFE/SPIRE, OPA policy engine, implementation roadmap (crawl→walk→run). | Extends `le-study-notes/security/03-zero-trust.md` |
| `02-aws-security.md` | IAM + networking integration (SCPs, permission boundaries), Security Groups, GuardDuty, Security Hub, Inspector, Macie, CloudTrail analysis. | NEW |
| `03-azure-security.md` | Azure AD + networking, Microsoft Defender for Cloud, Azure Sentinel, PIM, Conditional Access, NSGs + ASGs at scale. | NEW |
| `04-container-security.md` | Image hardening, runtime security (Falco, Tetragon), Pod Security Standards, seccomp/AppArmor, Kyverno enforcement. | Extends `le-study-notes/security/02-container-security.md` + `05-kyverno.md` |
| `05-supply-chain-security.md` | SLSA levels, Sigstore (Cosign, Fulcio, Rekor), SBOM (CycloneDX/SPDX), in-toto, Dependabot, xz Utils incident analysis. | Extends `le-study-notes/security/04-supply-chain-security.md` |
| `06-secrets-management.md` | ESO, Vault, AWS Secrets Manager, Azure Key Vault, secret rotation, GitOps integration (Argo CD + ESO), CSI driver. | Extends `le-study-notes/security/06-external-secrets-operator.md` |
| `07-owasp-api-security.md` | OWASP API Top 10 (2023), auth patterns (OAuth2/OIDC/JWT/mTLS), rate limiting algorithms, GraphQL security, input validation. | Extends `le-study-notes/security/01-owasp-api-security.md` |
| `08-compliance-frameworks.md` | PCI-DSS network controls, HIPAA security rule, SOC 2 CC6/CC7, NIST 800-53 network controls, mapping to K8s/cloud controls. | NEW |

---

### 07 — Debugging Playbooks

**Progression:** Learn the methodology first, then apply to specific failure modes. Each playbook is standalone.

| File | Contents | Format |
|------|----------|--------|
| `00-debugging-methodology.md` | 5-layer diagnostic model, hypothesis-driven debugging, information radiators, escalation criteria, "observe before acting" principle. | Mermaid decision tree |
| `01-service-not-reachable.md` | DNS → firewall → routing → app → TLS decision tree. Commands at each step. Production examples. | Decision tree + runbook |
| `02-high-latency.md` | Layer-by-layer latency diagnosis: NIC drops → TCP retransmits → connection pool → app code → DNS TTL. | Runbook + commands |
| `03-dns-failures.md` | NXDOMAIN vs SERVFAIL vs timeout, ndots exhaustion in K8s, CoreDNS crashes, split-horizon confusion. | Runbook + commands |
| `04-tls-certificate-issues.md` | Certificate expiry, chain validation, mTLS auth failures, cipher mismatch, SNI issues. `openssl s_client` workflow. | Runbook + commands |
| `05-packet-drops.md` | RX drop → ring buffer → conntrack → iptables DROP → routing black hole. `ethtool`, `ss`, `nstat` workflow. | Runbook + commands |
| `06-kubernetes-networking-issues.md` | Pod can't reach service, DNS resolution fails, NetworkPolicy blocking, kube-proxy sync lag, CNI failures. | Runbook + Mermaid flowchart |
| `07-cloud-connectivity-issues.md` | VPC route table missing, Security Group blocking, NAT Gateway exhaustion, Direct Connect failover, PrivateLink DNS. | Runbook + commands |
| `08-ddos-incident-response.md` | Traffic classification, upstream blackholing, rate-limit escalation, Shield/WAF activation, post-incident review. | Incident playbook |

---

### 08 — System Design

**Progression:** Apply all prior knowledge to architectural trade-offs. Interview-ready narratives.

| File | Contents | Source |
|------|----------|--------|
| `01-secure-multi-tier-architecture.md` | 3-tier web app with explicit network boundaries, WAF → ALB → app → RDS, Security Groups per tier, KMS encryption, audit logging. | NEW |
| `02-zero-trust-design.md` | ZTA for a microservices platform: identity (SPIFFE), policy (OPA), device trust, mTLS everywhere, network policies. | NEW |
| `03-multi-region-failover.md` | Active-active vs active-passive, Route 53 health checks, Aurora Global Database, Cross-Region ELB, chaos engineering validation. | NEW |
| `04-cdn-edge-architecture.md` | CDN layers (origin → edge PoP → client), caching strategy, TLS termination at edge, WAF at CDN, DDoS absorption, cache purge strategies. | NEW |
| `05-high-availability-networking.md` | VRRP/keepalived, BGP anycast, ECMP load distribution, health check design, connection draining, split-brain avoidance. | NEW |
| `06-api-gateway-design.md` | API gateway roles (auth, rate limiting, routing, transformation), placement patterns, mTLS to backend services, observability. | NEW |

---

### 09 — Advanced Topics

**Progression:** Staff/Principal depth. These are separators in senior interviews.

| File | Contents | Source |
|------|----------|--------|
| `01-ebpf-internals.md` | BPF verifier, map types (hash, array, LRU, ring buffer), program types, CO-RE, BTF, tail calls, pinning, bpftrace scripting. | Extends `interview-prep/09_Exotic_and_Deep_Dive_Questions.md` Q7+Q9 |
| `02-service-mesh-advanced.md` | Istio Ambient mode (ztunnel + waypoint), Linkerd vs Istio performance, multi-cluster mesh patterns, sidecar resource prediction. | Extends `le-study-notes/networking/06-service-mesh.md` |
| `03-network-observability.md` | RED metrics for networks, Prometheus node_exporter network metrics, Grafana dashboards, distributed tracing (Jaeger/Tempo), Hubble for K8s, structured logging for network events. | NEW |
| `04-performance-engineering-at-scale.md` | RSS/RPS/RFS tuning, XPS, interrupt affinity, NUMA topology, TCP BBR at scale, GSO/GRO/TSO offloads, benchmarking methodology (netperf, iperf3). | Extends `interview-prep/02_Performance_and_Optimization.md` |
| `05-bgp-at-scale.md` | BGP internals (UPDATE/KEEPALIVE/NOTIFICATION), BGP communities, RPKI/ROA, route reflectors, BGP in the data center (Clos fabric), MetalLB BGP mode. | Extends `interview-prep/05_Advanced_Interview_QA.md` Q3 |
| `06-ipv6-production.md` | IPv6 addressing, dual-stack deployment, NDP vs ARP, IPv6-specific attack surface, AWS dual-stack VPC, K8s IPv6 support. | NEW |

---

### 10 — Interview Prep

**Progression:** Synthesis layer. Read sections 01–09 first. Files here are Q&A, not tutorials.

| File | Contents | Source |
|------|----------|--------|
| `00-study-guide.md` | 4-week study plan, prerequisite map, Q&A methodology, critical commands cheatsheet, topic priority for different role levels. | Extends `interview-prep/00_Study_Guide_Overview.md` |
| `01-networking-fundamentals-qa.md` | TCP state machine, subnetting, BGP, NAT, DNS, ARP Q&A (basic → senior level). | NEW |
| `02-linux-networking-qa.md` | Kernel path, conntrack, iptables, eBPF, socket tuning, namespace isolation — 17 deep scenarios. | Extends `interview-prep/05_Advanced_Interview_QA.md` |
| `03-cloud-networking-qa.md` | VPC design, TGW vs Peering, EKS CNI, Route 53, Direct Connect failover, AWS networking incidents. | NEW |
| `04-kubernetes-networking-qa.md` | CNI internals, kube-proxy modes, NetworkPolicy debugging, DNS ndots, service mesh trade-offs. | Extends `interview-prep/06_Cross_Domain_Integration.md` |
| `05-network-security-qa.md` | TLS handshake failures, certificate chain issues, DDoS response, WAF bypass, mTLS debugging. | NEW |
| `06-system-design-qa.md` | Multi-region LB, ZTA for microservices, CDN design, API gateway, HA network architecture. | Extends `interview-prep/04_System_Design_and_Scaling.md` |
| `07-cross-domain-scenarios.md` | 10 scenarios bridging Linux + K8s + Cloud (e.g. EKS pod can't reach RDS, Calico BGP in AWS). | Extends `interview-prep/06_Cross_Domain_Integration.md` |
| `08-interview-patterns-pitfalls.md` | 60-second triage, trade-off articulation, OSI model the right way, staff-level failure patterns. | Extends `interview-prep/08_Interview_Patterns_and_Pitfalls.md` |
| `09-exotic-deep-dive-qa.md` | NAPI internals, BBR congestion, AF_XDP, SO_REUSEPORT, TCP_FASTOPEN, connection tracking hash tables. | Extends `interview-prep/09_Exotic_and_Deep_Dive_Questions.md` |

---

### 11 — Hands-on Labs

**Progression:** After reading a topic, do the lab. Break/Fix approach reinforces understanding.

| File | Contents | Approach |
|------|----------|----------|
| `01-linux-networking-lab.md` | Build namespaces + veth + bridge manually. Reproduce conntrack exhaustion. Debug iptables DROP. | Break/Fix + step-by-step |
| `02-kubernetes-networking-lab.md` | Deploy pods, trace CNI path, apply NetworkPolicy, reproduce DNS ndots issue, debug kube-proxy. | Break/Fix + kubectl |
| `03-aws-vpc-lab.md` | Create VPC with public/private subnets, set up TGW, reproduce Security Group blocking, simulate NAT exhaustion. | AWS CLI + CloudFormation |
| `04-tls-debugging-lab.md` | Generate CA + server cert, reproduce cert expiry, mTLS handshake failure, OCSP debugging with openssl. | Shell commands |
| `05-ebpf-xdp-lab.md` | Write minimal XDP packet counter, attach to veth, use bpftrace to trace TCP connections, Hubble observation. | C + Go + bpftrace |

---

## Structure Rationale

1. **Dependency-ordered sections** — Fundamentals (01) → Linux (02) → Cloud (03) → K8s (04) flow naturally. Each section builds on the previous. A reader can start at their knowledge level.

2. **Security is woven throughout** — Security is NOT isolated to sections 05–06. Every networking section includes security implications. Sections 05–06 go deeper on security-specific architectures.

3. **Debugging playbooks are standalone** — Section 07 is the on-call reference. Engineers will return here during incidents. Each playbook is self-contained with commands, decision trees, and Mermaid flowcharts.

4. **Interview prep is synthesis, not duplication** — Section 10 references section 01–09 content but adds Q&A framing and interview-specific patterns (trade-off articulation, what interviewers really want).

5. **Labs reinforce learning** — Section 11 provides muscle memory that reading alone cannot. Each lab maps to one or more topic sections.

6. **Cloud is split AWS/Azure** — With dedicated subdirectories, not a monolithic file. Engineers working in one cloud don't wade through the other. Multi-cloud patterns are in their own file.

7. **Advanced topics are separated** — Section 09 contains staff/principal-level depth that would intimidate entry points into other sections. These are clearly marked as depth boosters.

8. **Mermaid diagrams throughout** — Per spec requirements: network flows, Kubernetes traffic, TLS handshake, debugging decision trees, cloud architectures.

---

## Content Creation Priority (Phase 2 Execution Order)

When approved, content should be generated in this order to maximize utility at each step:

**Batch 1 (Foundation):** 01-networking-fundamentals + 02-linux-networking
**Batch 2 (Cloud):** 03-cloud-networking/aws + 04-kubernetes-networking
**Batch 3 (Security):** 05-network-security + 06-cloud-security
**Batch 4 (Operations):** 07-debugging-playbooks + 08-system-design
**Batch 5 (Advanced):** 09-advanced-topics + 10-interview-prep + 11-hands-on-labs

Each batch can use parallel agents (max 3 per spec instructions) to generate content concurrently.

---

## File Count Summary

| Section | Files | Status |
|---------|-------|--------|
| 01 Networking Fundamentals | 8 | NEW (8) |
| 02 Linux Networking | 8 | EXTEND (8 from interview-prep) |
| 03 Cloud Networking | 9 | NEW (7) + EXTEND (2) |
| 04 Kubernetes Networking | 8 | EXTEND (4) + NEW (4) |
| 05 Network Security | 6 | EXTEND (1) + NEW (5) |
| 06 Cloud Security | 8 | EXTEND (5) + NEW (3) |
| 07 Debugging Playbooks | 9 | EXTEND (1) + NEW (8) |
| 08 System Design | 6 | NEW (6) |
| 09 Advanced Topics | 6 | EXTEND (3) + NEW (3) |
| 10 Interview Prep | 10 | EXTEND (4) + NEW (6) |
| 11 Hands-on Labs | 5 | NEW (5) |
| **TOTAL** | **83 files** | **47 new, 36 extended** |

---

## ⛔ PHASE 1 COMPLETE — AWAITING APPROVAL

This document proposes structure only. No content has been written.

**Next:** Upon approval, Phase 2 begins with parallel agent content creation in the batch order defined above.
