# System Design — Architecture Review

**Reviewer Profile:** Solutions Architect (15+ years), enterprise-scale distributed systems
**Review Date:** April 2026
**Scope:** All 6 design documents under `08-system-design/`

---

## Executive Summary

Overall, this is a **strong collection of system design documents** that demonstrates depth across networking, cloud infrastructure, security, and distributed systems. The designs are production-grade, well-reasoned, and include critical operational details (failure modes, runbooks, cost models) that most reference materials skip entirely.

That said, there are architectural blind spots, missing edge cases, and a few design choices that would be challenged in a Staff+/Principal design review. This document catalogs them all.

**Rating by document:**

| Document | Completeness | Architecture Soundness | Interview Readiness |
|----------|-------------|----------------------|-------------------|
| 01 — Secure Multi-Tier | ★★★★☆ | ★★★★☆ | ★★★★★ |
| 02 — Zero Trust K8s | ★★★★★ | ★★★★☆ | ★★★★★ |
| 03 — Multi-Region Failover | ★★★★☆ | ★★★☆☆ | ★★★★☆ |
| 04 — CDN Edge | ★★★★★ | ★★★★★ | ★★★★★ |
| 05 — HA Networking | ★★★★☆ | ★★★★☆ | ★★★★★ |
| 06 — API Gateway | ★★★★★ | ★★★★☆ | ★★★★★ |

---

## 01 — Secure Multi-Tier Architecture

### Issues

> [!WARNING]
> **IRSA on EC2 is incorrect terminology.** Line 178 references "IRSA pattern on EC2." IRSA (IAM Roles for Service Accounts) is an EKS-specific feature binding Kubernetes ServiceAccounts to IAM roles via OIDC federation. EC2 instances use **Instance Profiles** (not IRSA). The concept is correct (credential-free IAM roles), but using IRSA terminology for EC2 would be flagged in a technical interview by someone familiar with the distinction.

- **Subnet sizing too small.** `/24` subnets (254 usable IPs) for the app tier may be insufficient for auto-scaling to 30 instances, especially if EKS is adopted later (pods consume IPs). A `/22` (1022 IPs) or `/20` per subnet is safer for growth. Kubernetes with the AWS VPC CNI uses one IP per pod — 30 EC2 instances running 50 pods each would exhaust a `/24` instantly.

- **No WAF tuning discussion.** The design deploys WAF with the OWASP managed rule set but doesn't address the common operational reality: managed rules produce **false positives** at scale. Missing: a strategy for running rules in `Count` mode initially, analyzing blocked requests via WAF logs, and selectively overriding specific rule IDs. In production, untuned WAF rules block legitimate traffic within the first week.

- **Missing: application-layer encryption for PCI.** The design covers TLS in transit and KMS at rest, but PCI-DSS Requirement 3.4 requires rendering the Primary Account Number (PAN) unreadable **anywhere it is stored**. The design should mention application-level field encryption (e.g., AES-256-GCM for PAN field) in addition to RDS disk encryption. Disk-level encryption alone does not satisfy Req 3.4 if the data is readable by anyone with DB query access.

- **No connection pooling.** The app tier connects directly to RDS with `sslmode=verify-full`. For a 3-30 instance ASG, each opening 10-50 DB connections, the total connections could reach 1,500. RDS PostgreSQL `max_connections` on `db.r6g.xlarge` defaults to ~500. **RDS Proxy** or **PgBouncer** should be mentioned as a day-1 design choice, not just a scaling optimization.

### Recommendations

1. Correct IRSA → Instance Profile for EC2 workloads; mention IRSA specifically for potential EKS migration.
2. Increase subnet CIDR to `/20` for future-proofing.
3. Add a WAF tuning runbook or at least mention the `Count → Block` migration path.
4. Add application-layer field encryption for sensitive PCI data fields (PAN, CVV).
5. Include RDS Proxy or PgBouncer from day 1 to handle connection pooling.

---

## 02 — Zero Trust Architecture for Kubernetes

### Issues

> [!IMPORTANT]
> **SPIRE failure window is understated.** The document says "existing SVIDs still valid for remaining TTL" and "SPIRE Server HA (3-node Raft cluster)." However, if the SPIRE Server is down for longer than the maximum SVID TTL (24h), **all workload identities expire simultaneously** — a catastrophic cliff. The design should mention a SVID TTL staggering strategy or a fallback mechanism, and the monitoring section should alert well before the oldest SVID expires (e.g., alert at 75% of max TTL while SPIRE is unreachable).

- **Istiod annotation for SPIRE is incorrect.** Line 181 mentions `ca.istio.io/override` annotation for configuring Istio to use SPIRE as external CA. The actual integration mechanism is the `PILOT_CERT_PROVIDER` environment variable on Istiod set to `spiffe/spire`, plus configuring the Workload API socket mount. The annotation referenced does not exist as a standard Istio configuration. This would be caught in a technical deep-dive.

- **50MB sidecar overhead is outdated.** The document quotes ~50MB RAM per Envoy sidecar. Modern Istio 1.20+ with `proxyStatsMatcher` optimization and reduced access log verbosity runs at ~30-40MB. This isn't a bug, but the cost analysis and scaling math should use current numbers. At 500 services, the difference is 5GB of memory.

- **OPA Gatekeeper `failurePolicy: Fail` blast radius is under-discussed.** The document correctly identifies the trade-off but doesn't mention a critical operational scenario: if Gatekeeper is updated with a misconfigured policy that rejects all pods, combined with `failurePolicy: Fail`, **the cluster cannot self-heal** — kube-system pods that need to restart (CoreDNS, kube-proxy) will also be blocked. The recommendation should be to **always exempt kube-system namespace** from Gatekeeper webhooks.

- **No egress policy in the NetworkPolicy section.** The default deny policy blocks egress, but the document never shows an example egress allow policy for DNS (UDP/53 to kube-dns), HTTPS to external services, or API server connectivity. In practice, a default-deny egress policy that doesn't whitelist DNS immediately breaks all pod-to-service communication.

- **Missing: graceful rollback plan for STRICT mTLS.** Phase 2 mentions "roll back if >0.1% error increase" but doesn't specify how. Rolling back from STRICT to PERMISSIVE mid-deployment requires re-applying the PeerAuthentication manifest — but if the error spike is caused by certificate issues, the rollback itself may fail because Istiod config push requires mTLS. A pre-staged rollback path (e.g., keeping PERMISSIVE as a git branch ready to apply) should be documented.

### Recommendations

1. Add SVID TTL cliff alerting and stagger strategy.
2. Fix the Istiod–SPIRE integration reference.
3. Exempt `kube-system` and other critical namespaces from Gatekeeper `failurePolicy: Fail`.
4. Add example egress NetworkPolicy allowing DNS, API server, and required external endpoints.
5. Document a concrete rollback procedure for mTLS migration failures.

---

## 03 — Multi-Region Failover

### Issues

> [!CAUTION]
> **Claims "active-active" but writes are single-region.** The design routes ALL writes to us-east-1, even from eu-west-1 app servers. This is architecturally **active-passive for writes** with active-active for reads. The document should be explicit about this — calling it "active-active" without qualification is misleading. In an interview, an experienced architect will probe this immediately: "If eu-west-1 can only read, is it truly active-active?" The correct characterization is **"active-active read / single-writer"** or **"read-anywhere, write-primary."**

> [!WARNING]
> **Aurora Global DB failover is NOT automatic — contradicts stated RTO.** The document states RTO < 1 minute, but also acknowledges that Aurora Global Database failover "requires an API call" and proposes a Lambda automation. The Lambda approach adds: CloudWatch alarm delay (1-3 data points, ~2 minutes) + Lambda cold start (~1s) + Aurora failover (~1 minute) + connection pool retry (~30s) = **3-5 minutes realistic RTO**. The stated 1-minute RTO is best-case theater. The design should present the honest range (1-5 minutes) and discuss how to reduce it.

- **GDPR compliance approach has legal gaps.** The design proposes "transfer under necessity" (GDPR Article 49(1)(f)) during failover when EU data goes to us-east-1. Article 49(1)(f) applies to vital interests of data subjects — not service availability. A more defensible position is **Article 49(1)(d)** (transfer for important public interest or legal obligations) or, more practically, using **Standard Contractual Clauses (SCCs)** as the cross-border transfer mechanism. The application-layer encryption approach is good but the legal citation should be corrected.

- **No data conflict resolution strategy.** The design routes EU writes cross-region to us-east-1. What happens if us-east-1 is unreachable but eu-west-1 is healthy? The eu-west-1 app servers cannot write. The document mentions promoting the eu-west-1 replica, but during the 3-5 minute failover window, **all writes globally fail.** An alternative like a local write queue (SQS) for critical writes during failover is not discussed.

- **Cross-region write latency understated.** The document says "~80ms cross-region latency." This is only the network RTT. A single Aurora write involves: cross-region RTT (~80ms) + Aurora quorum write latency (~5ms) + TLS overhead (~2ms) = ~90ms per write. For transactions involving multiple writes, this compounds. A 5-write transaction from eu-west-1 takes ~450ms — potentially a UX problem for synchronous API calls.

- **ElastiCache Redis in both regions but no clear purpose.** The architecture diagram shows Redis in both regions, but the session management section concludes that JWTs are preferred (no Redis needed for sessions). Redis appears in the diagram but its actual use case is undefined. If it's for application caching, the cache invalidation strategy across regions is missing.

- **No discussion of database schema differences across regions.** Schema migrations in active-active must be deployed to all regions simultaneously and must be backward-compatible. The expand-contract pattern from Document 01 is relevant here but not referenced.

### Recommendations

1. Rename design to "Active-Active Read / Single-Writer" and be explicit about the write path limitation.
2. Present honest RTO range (1-5 min) and outline steps to reduce it (pre-warmed Lambda, shorter CloudWatch evaluation period).
3. Fix the GDPR legal citation — use SCCs as the primary mechanism, not Art. 49.
4. Add a write buffering strategy (SQS-based write queue) for the failover window.
5. Remove Redis from the architecture if JWTs are the session strategy, or clearly define its purpose.
6. Add a section on cross-region schema migration strategy.

---

## 04 — CDN and Edge Architecture

### Issues

- **Signed URL validation at Lambda@Edge is redundant.** Line 208 says "Lambda@Edge validates the signature before CloudFront serves from cache." This is incorrect — CloudFront's **built-in signed URL validation** happens before the cache layer. If using CloudFront's native signing, Lambda@Edge is not needed for validation. If using custom signing (e.g., application JWT), then Lambda@Edge is needed. The document should clarify which approach is being used and why.

- **Content Security Policy is too restrictive for a media platform.** The security headers include `default-src 'self'`. For a video platform using HLS.js (which uses `blob:` URLs and Web Workers), a CSP of `default-src 'self'` will block video playback. The `media-src blob: https:` is correct, but the `default-src 'self'` will block `worker-src`, `script-src` for third-party analytics, and `connect-src` for API calls to different domains. This CSP needs media platform-specific tuning.

- **No multi-CDN failover mechanism described.** At 10x scale, the document recommends multi-CDN but doesn't explain the failover mechanism. The mention of "Cedexis/Conviva" for CDN orchestration is good but insufficient. Missing: how does the system detect that CloudFront is degraded and fail traffic to Fastly? DNS-based switching? Client-side quality-of-experience metrics? Real User Monitoring (RUM)?

- **DRM is mentioned but not architected.** The security section lists "Widevine/FairPlay DRM" as a line item but doesn't discuss the key exchange architecture: where does the license server sit, how are content keys delivered, how does DRM interact with CloudFront signed URLs? For a "high-traffic media platform," DRM is a critical component that deserves more than one line.

- **Pre-warming strategy has practical limitations.** Making HEAD requests to trigger cache fill assumes CloudFront will serve the content from cache on subsequent requests from different PoPs. CloudFront PoPs are independent — warming from `us-east-1` doesn't warm PoPs in Asia. Pre-warming would need to originate from each target PoP region, which is non-trivial.

### Recommendations

1. Clarify whether native CloudFront signed URL validation or custom Lambda@Edge validation is used.
2. Tune the CSP for media platform requirements (add `worker-src blob:`, `connect-src` for API domains).
3. Add a multi-CDN failover mechanism section (RUM-based switching or DNS GSLB).
4. Expand DRM to include a basic license server architecture.
5. Acknowledge PoP-locality limitation of cache pre-warming; suggest using regional Lambda functions for targeted warming.

---

## 05 — High-Availability Networking

### Issues

- **VRRP authentication uses MD5 — deprecated.** Line 221 recommends MD5 for VRRP authentication. VRRP v3 (RFC 5798) dropped the MD5 authentication mechanism entirely because it provided a false sense of security (MD5 is weak, and VRRP runs on a local subnet where the threat model doesn't justify complex auth). The document should note that VRRPv3 does not support authentication natively, and if authentication is needed, use MACsec or 802.1X at the link layer instead.

- **BGP MD5 session authentication is also weak.** MD5 is computationally broken. Modern BGP implementations support TCP-AO (TCP Authentication Option, RFC 5925) which replaces MD5 with HMAC-SHA-256. The document should recommend TCP-AO where supported (JunOS, recent IOS-XR) and mention MD5 as the legacy fallback.

- **No mention of IPv6.** The entire document discusses IPv4 networking only. For a 15+ year architect perspective: any new network design in 2026 should be **dual-stack** from day 1. IPv4 address exhaustion is real, AWS charges $0.005/hour for public IPv4 addresses (since February 2024), and IPv6 is free. The cost section should mention this.

- **VRRP section doesn't discuss virtual MAC behavior.** When VRRP failover occurs, the new master uses the VRRP virtual MAC address (`00:00:5E:00:01:XX`). Some network environments (especially cloud VPCs) do not support virtual MACs or gratuitous ARP. AWS, for example, does NOT support VRRP in VPCs. The document positions VRRP as a universal solution, but it is only applicable to on-premises/colo environments. This should be noted.

- **SR-IOV description overstates latency improvement.** Line 383 claims SR-IOV reduces latency from "~50 microseconds to ~5 microseconds." This is possible for raw packet I/O, but for TCP/IP workloads, the kernel network stack still adds latency. A more accurate claim: SR-IOV reduces **virtualization overhead** specifically (the vSwitch bypass), saving ~10-20 microseconds per packet. Application-level latency reduction depends on the workload.

- **Missing: segment routing and SRv6.** For a design document targeting Staff+ SRE roles, modern network designs increasingly use Segment Routing (SR-MPLS or SRv6) instead of or alongside traditional BGP ECMP for traffic engineering. This is a notable gap for advanced interview questions.

### Recommendations

1. Update VRRP and BGP authentication recommendations — deprecate MD5, recommend TCP-AO and MACsec.
2. Add a note that VRRP is not applicable in cloud VPCs (AWS, Azure, GCP all use different HA mechanisms).
3. Add IPv6 dual-stack discussion and the AWS IPv4 cost implications.
4. Correct SR-IOV latency claims to be more precise.
5. Add a brief section on Segment Routing (SRv6) as an advanced traffic engineering topic.

---

## 06 — API Gateway Architecture

### Issues

- **Rate limit bypass window on Redis failure.** The design recommends "fail open" when Redis is down. At 100K RPS with no rate limiting, a malicious client can send unlimited traffic for the duration of the Redis outage. Missing: a **local rate limiter fallback** — each gateway node maintains an in-memory token bucket as a rough per-IP limiter during Redis outage. This prevents a single client from consuming all 100K RPS capacity even when Redis is down.

- **JWT + API key dual-auth inconsistency.** The auth section describes JWT, OAuth2 introspection, and API key as separate flows, but doesn't clarify how the gateway decides **which to use** for a given request. Is it path-based (`/api/partner/*` uses mTLS + API key, `/api/v1/*` uses JWT)? Is it header-based (presence of `Authorization: Bearer` vs `X-API-Key`)? A routing rule for auth method selection is missing.

- **gRPC protocol translation not architected.** The requirements mention "REST ↔ gRPC protocol translation" but the component design doesn't explain how this works. gRPC uses HTTP/2 with Protocol Buffers, while REST uses HTTP/1.1 with JSON. Transcoding requires: proto definition files at the gateway, gRPC-JSON transcoding (Envoy's `grpc_json_transcoder` filter), and schema management. This is non-trivial and deserves its own subsection.

- **Canary routing with consistent hashing can cause hotspots.** Line 546 says "consistent hashing (hash on user ID)" for canary routing. If 5% of traffic goes to canary, and the hash distributes by user ID, some user IDs may land on the canary while being high-traffic power users, overwhelming the canary. Better: use deterministic bucketing with modulo (`user_id % 100 < 5 → canary`) which distributes more evenly across the population.

- **Missing: API versioning strategy.** The routing config shows `/api/v1/` and `/api/v2/` but doesn't discuss the API versioning policy: how long do old versions live? How are clients migrated? What's the sunset process? For a platform with 50 microservices and external partners, API versioning is a critical architectural concern.

- **No discussion of WebSocket handling.** The requirements mention gRPC and REST but some microservices architectures require WebSocket support (real-time notifications, streaming). How does the gateway handle long-lived WebSocket connections for rate limiting (per-connection vs per-message), authentication (token refresh during connection), and scaling (connection affinity)?

- **Cost calculation for AWS API Gateway appears inflated.** Line 513 calculates 259 billion requests/month at 100K RPS × 86400s × 30d. Correct math: 100,000 × 86,400 × 30 = 259,200,000,000 (259.2B). At $3.50/M = **$907,200/month**. The math is correct but the document rounds to "$906,500" — close enough, but the point that self-hosted is 180x cheaper stands.

### Recommendations

1. Add a local in-memory rate limiter fallback for Redis outage scenarios.
2. Clarify auth method selection routing (which endpoints use which auth mechanism).
3. Add a gRPC-JSON transcoding section with Envoy filter configuration.
4. Use modulo-based bucketing for canary routing instead of consistent hashing.
5. Add an API versioning and deprecation strategy section.
6. Address WebSocket connection handling at the gateway.

---

## Cross-Cutting Concerns (All Documents)

### Gaps Present Across Multiple Designs

| Gap | Affected Documents | Recommendation |
|-----|-------------------|----------------|
| **No disaster recovery testing cadence** | 01, 02 | Only Doc 03 includes chaos engineering/game days. Add DR testing section to all designs. |
| **No capacity planning methodology** | 01, 02, 04 | The scaling sections describe what to do at 10x but don't describe how to **detect** that 10x is approaching before it hits. Add leading indicators and capacity alerts (e.g., alert at 70% of RDS max connections). |
| **Inconsistent cost estimation depth** | All | Doc 04 has excellent cost modeling with per-component math. Docs 02 and 05 have only qualitative cost discussions. Standardize. |
| **No incident response integration** | 01, 03, 04 | Failure modes are documented but how failures connect to incident management (PagerDuty escalation, runbook links, communication templates) is mostly absent. |
| **Missing: toil reduction and automation maturity** | All | None of the documents discuss IaC maturity (Terraform state management, drift detection), GitOps pipeline security, or self-service developer workflows. For Staff+ SRE interviews, operational efficiency is a key dimension. |
| **No accessibility or client-side considerations** | 03, 04, 06 | The multi-region and CDN designs don't discuss client-side resilience (retry libraries, fallback URLs, progressive degradation). The API gateway doesn't mention SDK generation or developer experience. |

### Structural Strengths

These are consistently well-done across all documents:

- **Defense-in-depth security layers** — every design includes layered security with clear justification
- **Trade-offs tables** — explicit alternatives-considered with reasoning
- **Failure mode analysis** — detection + mitigation for each component
- **Interview questions** — well-graduated difficulty with practical answers
- **Scaling sections** — realistic discussion of bottlenecks at 10x

---

## Summary of Critical Fixes (Priority Order)

| Priority | Document | Fix |
|----------|----------|-----|
| 🔴 High | 01 | Correct IRSA → Instance Profile for EC2 |
| 🔴 High | 03 | Rename to "Active-Active Read / Single-Writer" — misleading terminology |
| 🔴 High | 03 | Present honest RTO range (1-5 min), not aspirational 1 min |
| 🟡 Medium | 02 | Fix Istiod–SPIRE integration annotation reference |
| 🟡 Medium | 03 | Fix GDPR legal citation (Art 49 subsection) |
| 🟡 Medium | 05 | Note VRRP inapplicability in cloud VPCs |
| 🟡 Medium | 05 | Deprecate MD5 auth recommendations |
| 🟡 Medium | 06 | Add local rate limiter fallback for Redis outage |
| 🟢 Low | 01 | Increase subnet sizing from /24 to /20 |
| 🟢 Low | 02 | Add DNS egress NetworkPolicy example |
| 🟢 Low | 04 | Tune CSP for media platform |
| 🟢 Low | 05 | Add IPv6 dual-stack discussion |
