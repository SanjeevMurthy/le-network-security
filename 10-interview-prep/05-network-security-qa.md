# Network Security — Interview Q&A

## Table of Contents

- [Quick Reference](#quick-reference)
- [Q: Walk me through a TLS 1.3 handshake with sequence numbers. Where is the key improvement over TLS 1.2?](#q-walk-me-through-a-tls-13-handshake-with-sequence-numbers-where-is-the-key-improvement-over-tls-12)
- [Q: mTLS vs TLS — what's different, and when is mTLS required?](#q-mtls-vs-tls-whats-different-and-when-is-mtls-required)
- [Q: How does a SYN flood work and how do SYN cookies mitigate it? What does SYN cookies cost you?](#q-how-does-a-syn-flood-work-and-how-do-syn-cookies-mitigate-it-what-does-syn-cookies-cost-you)
- [Q: PKI hierarchy — why is the root CA offline? What is the attack if it's compromised?](#q-pki-hierarchy-why-is-the-root-ca-offline-what-is-the-attack-if-its-compromised)
- [Q: DDoS mitigation layers — what stops a 1 Tbps volumetric attack?](#q-ddos-mitigation-layers-what-stops-a-1-tbps-volumetric-attack)
- [Q: Certificate expiry in production — how do you prevent it and detect it before it causes an outage?](#q-certificate-expiry-in-production-how-do-you-prevent-it-and-detect-it-before-it-causes-an-outage)
- [Q: Network segmentation for PCI-DSS compliance — what does the auditor check?](#q-network-segmentation-for-pci-dss-compliance-what-does-the-auditor-check)
- [Key Takeaways](#key-takeaways)

---

## Quick Reference

Network security interviews at Senior/Staff SRE level test whether you understand security mechanisms as systems, not checkbox compliance. Every answer should include: how the mechanism works at the protocol or kernel level, what breaks it, how you detect a failure, and the threat model it's addressing. The most common failure pattern is describing security controls without knowing their failure modes — "we use TLS" without knowing what TLS 1.2 vs 1.3 means operationally, or "we have a WAF" without knowing what it misses. See `05-network-security/` and `06-cloud-security/` for deeper reading.

---

## Q: Walk me through a TLS 1.3 handshake with sequence numbers. Where is the key improvement over TLS 1.2?

**What the interviewer is testing:** Whether you understand TLS 1.3 as a security and performance improvement, and can cite specifics.

**Model Answer:**

**TLS 1.3 Handshake (1-RTT):**

```
Client                                    Server
  |                                          |
  |-- ClientHello (seq 1) ----------------> |
  |   Contains: supported_versions=[TLS1.3] |
  |   key_share (ECDH public key)           |
  |   supported_ciphers                     |
  |   random (32 bytes)                     |
  |                                          |
  |<-- ServerHello (seq 2) --------------- |
  |    key_share (server ECDH public key)   |
  |    selected_cipher (TLS_AES_256_GCM_SHA384)
  |                                          |
  |<-- {EncryptedExtensions} (seq 3) ----- | ← Encrypted from here!
  |<-- {Certificate} (seq 4) ------------ |
  |<-- {CertificateVerify} (seq 5) ------- |
  |<-- {Finished} (seq 6) ---------------- |
  |                                          |
  |-- {Finished} (seq 7) ----------------> |
  |                                          |
  |-- Application Data ----------------->  | ← 1-RTT: TLS complete
  |<-- Application Data ----------------  |
```

**Key improvements over TLS 1.2:**

1. **Fewer round trips: 1-RTT vs 2-RTT for TLS 1.2**
   - TLS 1.2: ClientHello → ServerHello → Certificate → ServerHelloDone → ClientKeyExchange → ChangeCipherSpec → Finished (2 round trips before data)
   - TLS 1.3: ClientHello → ServerHello+Keys → Finished (1 round trip)
   - At 100ms RTT: TLS 1.2 adds 200ms latency; TLS 1.3 adds 100ms

2. **0-RTT resumption (for returning connections):**
   ```
   Client: ClientHello + early_data (PSK) + Application Data (immediately!)
   Server: ... processes early data after handshake validation
   ```
   0-RTT has **replay attack risk** — safe only for idempotent requests.

3. **Forward secrecy mandatory:**
   - TLS 1.3 removes RSA key exchange (static private key can decrypt all past sessions if compromised)
   - Only ECDHE/DHE supported — each session has a unique ephemeral key pair
   - Past sessions cannot be decrypted even if long-term cert is compromised

4. **Removed weak cipher suites:**
   - TLS 1.2 allowed RC4, 3DES, CBC mode
   - TLS 1.3: only AES-GCM, ChaCha20-Poly1305 (AEAD ciphers) — no CBC mode, no compression

5. **Certificate and extensions encrypted:**
   - In TLS 1.2: Certificate is sent in plaintext — passive observer sees which certificate you're using
   - In TLS 1.3: Certificate is encrypted — reduces metadata leakage

**Diagnosing TLS issues:**
```bash
# Check TLS version and cipher in use
openssl s_client -connect api.example.com:443 -tls1_3
# Look for: Protocol: TLSv1.3, Cipher: TLS_AES_256_GCM_SHA384

# Test TLS 1.2 fallback
openssl s_client -connect api.example.com:443 -tls1_2
# If this fails but 1.3 works: server has disabled 1.2 (good)

# Check certificate chain
openssl s_client -connect api.example.com:443 -showcerts 2>/dev/null | openssl x509 -noout -dates
```

**Follow-up Questions:**
- What is ESNI/ECH (Encrypted Client Hello) and what privacy problem does it solve?
- When is 0-RTT safe to use and when is it dangerous?
- How does certificate transparency (CT) logging work and why is it required?

**Connecting Points:**
- `05-network-security/` for PKI and certificate management
- `10-interview-prep/07-cross-domain-scenarios.md` for mTLS certificate rotation scenario

---

## Q: mTLS vs TLS — what's different, and when is mTLS required?

**What the interviewer is testing:** Whether you understand that most TLS is one-way authentication (you verify the server), and mTLS adds client authentication.

**Model Answer:**

**Standard TLS (one-way):**
- Server presents a certificate; client verifies it against a CA
- Client identity is NOT established at the TLS layer (application handles auth via cookies, tokens, API keys)
- Attacker scenario: if an attacker intercepts traffic and presents a valid certificate for the domain (via a compromised CA), they can MITM the connection

**mTLS (mutual TLS):**
- BOTH sides present certificates and verify each other
- Client certificate identifies the specific service/workload making the request
- Connection is refused if either certificate is invalid, expired, or revoked

**When mTLS is required:**

1. **Service-to-service authentication (zero-trust):**
   - Each microservice has a certificate with its identity (e.g., `spiffe://cluster.local/ns/payments/sa/payment-service`)
   - SPIFFE (Secure Production Identity Framework For Everyone) defines the identity format
   - No need for network-level isolation — identity is cryptographic

2. **Zero-trust network architecture:**
   - Replace VPN with mTLS: "never trust the network, always verify the identity"
   - An attacker inside the network cannot impersonate a service without its private key

3. **Regulatory requirements:**
   - PCI-DSS 4.0 requires strong authentication for all connections between system components
   - mTLS satisfies this for service-to-service connections

**Implementation in Kubernetes (Istio):**
```yaml
# Enforce STRICT mTLS for all services in production namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT   # reject any non-mTLS connection
```

```bash
# Verify mTLS is working between services
istioctl authn tls-check <pod> <service>
# Output: "OK" means mTLS is active, "CONFLICT" means policy mismatch

# Check certificate validity
kubectl exec -n production <pod> -c istio-proxy -- \
  openssl x509 -in /etc/certs/cert-chain.pem -noout -dates -subject
```

**Operational challenges:**
- Certificate rotation must be seamless (brief interruption if certs expire before rotation)
- Bootstrap problem: first connection to istiod needs a cert to get a cert (solved by SPIFFE/SPIRE bootstrap tokens)
- Debugging: TLS handshake failures are logged at `WARN` level in Envoy; plaintext debugging is harder than with non-TLS

**Follow-up Questions:**
- What is SPIFFE SVID and how does it identify a workload?
- How do you handle mTLS between Kubernetes clusters (cross-cluster)?
- What is certificate transparency and does it apply to internal mTLS certificates?

**Connecting Points:**
- `10-interview-prep/04-kubernetes-networking-qa.md` for Istio mTLS implementation
- `06-cloud-security/` for zero-trust architecture patterns

---

## Q: How does a SYN flood work and how do SYN cookies mitigate it? What does SYN cookies cost you?

**What the interviewer is testing:** Whether you understand the kernel's defense mechanism and its trade-offs.

**Model Answer:**

**The attack:**
The TCP handshake requires the server to maintain state for half-open connections (SYN received, SYN-ACK sent, waiting for ACK). This half-open state lives in the SYN backlog queue:
```
sysctl net.ipv4.tcp_max_syn_backlog    # default 512 on many systems
```

The attacker sends SYN packets with spoofed source IPs at high rate. The server:
1. Receives SYN
2. Allocates a backlog entry (500 bytes of kernel memory per entry)
3. Sends SYN-ACK to the (spoofed) source — packet goes nowhere
4. Waits for ACK (never arrives)

At 10,000 SYNs/second: the backlog fills in 50ms. Legitimate SYNs are dropped.

**SYN Cookies — how they work:**
Instead of allocating a backlog entry, the server encodes the connection state into the SYN-ACK's ISN (Initial Sequence Number):

```
Normal ISN: random 32-bit number
SYN cookie ISN: t=timestamp(5 bits) + m=MSS(3 bits) + hash(IP, port, secret, t)(24 bits)
```

If the client is real, it sends ACK with `ISN + 1`. The server decodes:
- Extract timestamp, MSS, and verify hash
- If hash matches: reconstruct the TCP state, complete the handshake
- If hash doesn't match (spoofed): discard silently — no backlog entry was ever allocated

```bash
sysctl net.ipv4.tcp_syncookies=1    # default: enabled
# Linux only enables cookies when backlog is full (not always)
sysctl net.ipv4.tcp_syncookies=2    # always use cookies (kernel 4.4+)
```

**The cost — what SYN cookies can't do:**

1. **No TCP options:** The SYN cookie encodes only the MSS (3 bits, 8 possible values). Cannot encode: SACK, window scaling, timestamps. If the client negotiated window scaling, the server cannot honor it after a cookie-based handshake. This limits the connection to a maximum window size of 65,535 bytes (no scaling), reducing throughput on high-BDP links.

2. **Limited MSS values:** Only 8 MSS values can be encoded. The actual negotiated MSS is rounded down to the nearest encoded value.

3. **Cannot resume half-open connections:** Traditional half-open state allows retransmitting the SYN-ACK if the client's ACK is lost. SYN cookies have no state to retransmit from.

**In production:**
SYN cookies are a backstop, not a complete DDoS defense. For production-grade defense:
```bash
# Layer 1: Increase backlog (delay when cookies kick in)
sysctl -w net.ipv4.tcp_max_syn_backlog=65536
sysctl -w net.core.somaxconn=65536

# Layer 2: Reduce SYN-ACK retransmissions (reduce state retention time)
sysctl -w net.ipv4.tcp_syn_retries=2    # default 5 (reduces time per spoofed SYN from 67s to 7s)

# Layer 3: XDP-based rate limiting before the kernel stack
# Layer 4: BGP blackhole or scrubbing center
```

**Follow-up Questions:**
- What is the difference between a SYN flood and a connection flood?
- How do ACK floods bypass SYN cookies?
- What kernel metric shows when SYN cookies are being used?

**Connecting Points:**
- `02-linux-networking/` for SYN flood defense tools (XDP, iptables rate limiting)
- `05-network-security/` for DDoS mitigation architecture

---

## Q: PKI hierarchy — why is the root CA offline? What is the attack if it's compromised?

**What the interviewer is testing:** Whether you understand certificate trust chains and the blast radius of CA compromise.

**Model Answer:**

**PKI hierarchy:**
```
Root CA (offline, air-gapped)
  └── Intermediate CA 1 (online, issues leaf certs for web servers)
  └── Intermediate CA 2 (online, issues leaf certs for internal services)
       └── Leaf Certificate (api.example.com, valid 90 days)
```

**Why the root CA is offline:**

The root CA's private key is the trust anchor for ALL certificates in the hierarchy. If compromised:
- The attacker can issue certificates for ANY domain
- All browsers and systems would trust these fraudulent certificates
- The attacker can MITM any TLS connection to any site

By keeping the root CA offline (hardware security module in a physical vault, or air-gapped system used only annually to sign intermediate CA certificates), you:
1. Reduce the attack surface to near-zero (can't compromise a disconnected system remotely)
2. Limit the "blast radius" of an online system compromise — compromising an intermediate CA is bad, but the root CA (and therefore all other intermediates) remains intact

**What happens if the root CA is compromised:**

```
1. Attacker obtains root CA private key
2. Attacker can sign ANY certificate: "*.google.com", "*.github.com", "*.yourbank.com"
3. These certificates are trusted by ALL systems that trust the root CA
4. MITM any HTTPS connection — user sees green padlock, valid certificate
5. No TLS error, no warning — attack is invisible to end users

Recovery:
- Revoke the root CA certificate from all trust stores (Windows, macOS, Firefox, Chrome)
- This requires:
  - Microsoft Windows Update
  - Apple OS update
  - Mozilla NSS update
  - Google Chrome Root Store update
- Timeline: weeks to months for full revocation across all clients
```

**Certificate Transparency (CT) as a detection mechanism:**
```bash
# All publicly trusted certificates must be logged to CT logs
# Monitor your domain's CT logs for unauthorized certificates
# Tools: crt.sh, certspotter

curl "https://crt.sh/?q=%.yourdomain.com&output=json" | jq '.[].name_value'
# Alert on any certificate you didn't issue
```

**Intermediate CA compromise (more common):**
- Compromised intermediate can issue certs for any domain
- Recovery: revoke intermediate CA certificate via CRL/OCSP
- CRL/OCSP revocation is faster (hours) but client enforcement is inconsistent
- Certificate pinning (HPKP, now deprecated) could detect this but caused outages

**Follow-up Questions:**
- What is OCSP stapling and why is it better than OCSP?
- How does Let's Encrypt's domain validation work and what can it be bypassed by?
- What is CAA (Certification Authority Authorization) DNS record?

**Connecting Points:**
- `05-network-security/` for PKI operations
- `10-interview-prep/05-network-security-qa.md` for certificate expiry prevention

---

## Q: DDoS mitigation layers — what stops a 1 Tbps volumetric attack?

**What the interviewer is testing:** Whether you understand that no single server can absorb 1 Tbps and that DDoS defense is an architectural problem.

**Model Answer:**

A 1 Tbps volumetric attack saturates any single datacenter's uplink. Defense must happen UPSTREAM — at the transit provider, CDN edge, or scrubbing center. Layers:

**Layer 1 — Anycast and distributed absorption (CDN/Cloud):**
The most effective defense is being everywhere. If your traffic is spread across 200+ PoPs globally, a 1 Tbps attack directed at one PoP is absorbed across the global infrastructure.
- Cloudflare Magic Transit: anycast routing of your IP space through Cloudflare's 200+ PoP network (250+ Tbps capacity)
- AWS Shield Advanced: absorbs at AWS edge before traffic enters your VPC
- Akamai Prolexic: scrubbing centers with 20+ Tbps capacity

**Layer 2 — BGP Blackhole (RTBH):**
```bash
# Emergency: announce your target IP with blackhole community to upstreams
# community 65535:666 (RFC 7999) signals "blackhole this prefix"
ip route add <target-ip>/32 nexthop blackhole
# Propagate with BGP community to upstream providers:
# route-map BGPANNOUNCE set community 65535:666
# Cost: ALL traffic to that IP is dropped, including legitimate traffic
# Use only when attack volume exceeds your capacity to serve legitimate users
```

**Layer 3 — XDP line-rate filtering (your edge):**
```c
// XDP program running at 100M pps — drops attack traffic before sk_buff allocation
// Uses BPF LRU map to track per-source-IP packet rate
// Rate limit: >10K pps from single source → XDP_DROP
SEC("xdp")
int ddos_filter(struct xdp_md *ctx) {
    // parse IP header
    // lookup src_ip in rate_limit_map
    // if rate > threshold: return XDP_DROP
    // else: return XDP_PASS and increment counter
}
```

**Layer 4 — SYN cookies + backlog tuning:**
```bash
sysctl -w net.ipv4.tcp_syncookies=2           # always use SYN cookies
sysctl -w net.ipv4.tcp_max_syn_backlog=65536  # larger backlog buffer
```

**Layer 5 — Application-layer defense (L7 DDoS):**
- Rate limiting by IP, ASN, user agent in your WAF or API gateway
- Challenge-response (CAPTCHA) for suspected bot traffic
- Bot fingerprinting (TLS JA3 hash, browser behavior analysis)

**For a 1 Tbps attack specifically:**
Single server: can absorb ~100 Gbps at most (NIC limit), and processing overhead means practical limit is 10-20 Gbps.
Scrubbing center: 20-100 Tbps capacity, but adds 20-50ms latency for all traffic.
CDN anycast: absorbs at global edge with minimal latency impact, but requires routing all traffic through CDN.

**Trade-off:** Scrubbing centers absorb attacks but add latency. CDN anycast provides better performance but requires giving a CDN provider access to your traffic.

**Follow-up Questions:**
- What is amplification DDoS (DNS amplification, NTP amplification) and how does it achieve 1 Tbps from small sources?
- What is BCP38 (ingress filtering) and why does it reduce DDoS risk?
- How do you distinguish DDoS traffic from legitimate traffic surge (flash crowd)?

**Connecting Points:**
- `05-network-security/` for DDoS mitigation tools
- `09-advanced-topics/` for XDP-based rate limiting

---

## Q: Certificate expiry in production — how do you prevent it and detect it before it causes an outage?

**What the interviewer is testing:** Whether you have operational experience with certificate lifecycle management, not just TLS theory.

**Model Answer:**

Certificate expiry outages are among the most embarrassing and preventable production incidents. The infamous 2021 Let's Encrypt cross-signing expiry affected millions of sites; the 2023 Teams/Outlook Microsoft outage was certificate-related.

**Prevention — automated renewal:**
```bash
# Let's Encrypt with cert-manager (Kubernetes):
# cert-manager renews certificates 30 days before expiry
kubectl get certificate -A    # check status
kubectl describe certificate <name>    # check renewal status

# cert-manager auto-renewal:
apiVersion: cert-manager.io/v1
kind: Certificate
spec:
  renewBefore: 720h   # 30 days before expiry
  secretName: my-tls-secret

# Let's Encrypt certbot (bare metal):
# Certificates are 90 days; certbot renews when <30 days remain
certbot renew --dry-run    # test renewal
# Cron: 0 0,12 * * * certbot renew --quiet
```

**Detection — multi-layer monitoring:**

1. **External probe (most reliable):**
```bash
# Script: check certificate expiry from outside
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null \
  | openssl x509 -noout -enddate
# Output: notAfter=Jan 15 00:00:00 2027 GMT

# Prometheus blackbox_exporter SSL probe:
# ssl_cert_expiry metric: seconds until expiry
# Alert: ssl_cert_expiry < 86400 * 14  (14 days warning)
#        ssl_cert_expiry < 86400 * 7   (7 days critical)
```

2. **Internal certificate store scanning:**
```bash
# Find all certificates in Kubernetes secrets
kubectl get secrets -A -o json | jq -r '
  .items[] |
  select(.data["tls.crt"] != null) |
  "\(.metadata.namespace)/\(.metadata.name)"
' | while read ns_name; do
  # decode and check expiry
  kubectl get secret -n ${ns_name%/*} ${ns_name#*/} \
    -o jsonpath='{.data.tls\.crt}' | base64 -d | \
    openssl x509 -noout -enddate -subject
done
```

3. **Datadog/CloudWatch certificate expiry monitors:**
   - Most monitoring platforms have built-in certificate expiry checks
   - Alert when expiry < 30 days (warning), < 7 days (critical), expired (page)

**Common failure modes beyond expiry:**

- **Chain incomplete:** Server sends leaf cert but not intermediate. Client without the intermediate cached gets "unable to verify" error.
```bash
openssl s_client -connect host:443 2>/dev/null | openssl x509 -text | grep "Issuer\|Subject"
# Verify the chain with: openssl verify -CAfile chain.pem leaf.pem
```

- **Wrong hostname:** Certificate issued for `www.example.com` but accessed as `api.example.com`. Check SAN (Subject Alternative Name) field.

- **Certificate/key mismatch after rotation:** New cert deployed but old key still in place (or vice versa).
```bash
# Verify cert and key match (modulus must be identical)
openssl x509 -noout -modulus -in cert.pem | openssl md5
openssl rsa  -noout -modulus -in key.pem  | openssl md5
```

**Follow-up Questions:**
- How does cert-manager handle Let's Encrypt rate limits for large clusters?
- What is certificate pinning and why was HPKP deprecated?
- How do you handle certificate rotation for mTLS in a service mesh without downtime?

**Connecting Points:**
- `10-interview-prep/07-cross-domain-scenarios.md` for mTLS blocking after rotation
- `06-cloud-security/` for AWS ACM certificate lifecycle

---

## Q: Network segmentation for PCI-DSS compliance — what does the auditor check?

**What the interviewer is testing:** Whether you understand PCI-DSS network requirements operationally, not just compliance checkboxes.

**Model Answer:**

PCI-DSS v4.0 (effective 2024) requires network segmentation to isolate the Cardholder Data Environment (CDE) from other networks. The auditor's goal: verify that a compromise outside the CDE cannot reach cardholder data.

**What the auditor actually checks (Requirements 1.2, 1.3, 1.4):**

1. **Network diagrams showing all traffic flows to/from CDE:**
```bash
# Auditor requests: current-as-of-date network diagram
# Must show: all systems with CHD, all connections to CDE, all external connections
# Tool: AWS VPC Reachability Analyzer, draw.io updated weekly
aws ec2 start-network-insights-analysis \
  --network-insights-path-id nip-XXXXXXXX
```

2. **Firewall rules allowing only necessary traffic:**
```bash
# Auditor reviews Security Group rules for CDE instances
# Must be able to justify EVERY inbound and outbound rule
# Common findings:
# - Port 22 (SSH) open from 0.0.0.0/0  → FAIL
# - Port 3389 (RDP) open from 0.0.0.0/0 → FAIL
# - Outbound 0.0.0.0/0 → should be restricted to known destinations
aws ec2 describe-security-groups \
  --filters "Name=tag:Environment,Values=PCI" \
  --query 'SecurityGroups[].IpPermissions'
```

3. **Segmentation between CDE and non-CDE proven:**
```bash
# Auditor may request penetration test results showing CDE is unreachable from DMZ
# Or network test: from a system in "payment-processing" VPC, can I reach "internal" VPC?
# Reachability test:
aws ec2 start-network-insights-analysis \
  --network-insights-path-id <path-id>
# Path: Non-CDE instance → CDE instance
# Expected result: "Not reachable"
```

4. **All inbound connections to CDE go through firewall (Requirement 1.2.3):**
- No direct internet-to-CDE paths
- All traffic flows: internet → WAF/ALB (DMZ) → application tier (CDE)
- Jump box / bastion host required for administrative access

5. **Change management for firewall rules:**
- Every rule change requires documented approval
- Firewall rules reviewed and approved at least every 6 months
- AWS Config Rule: check for SG changes without approval ticket

**AWS architecture for PCI-DSS CDE:**
```
Internet
  ↓
Route 53 + Shield Advanced (DDoS)
  ↓
WAF + ALB (public subnet, DMZ tier)
  ↓ [Security Group: allow 443 from ALB only]
Application servers (private subnet, CDE)
  ↓ [Security Group: allow 5432 from app tier only]
Database (isolated subnet, CDE core)
  ↓
No outbound internet (all external calls via VPC Endpoints or proxy)
```

**Follow-up Questions:**
- What is the "scope reduction" strategy for PCI-DSS and how does tokenization help?
- How do NACLs serve as an additional segmentation control alongside Security Groups?
- What does PCI-DSS say about encryption of cardholder data in transit?

**Connecting Points:**
- `06-cloud-security/` for PCI-DSS compliant AWS architecture
- `10-interview-prep/06-system-design-qa.md` for PCI-DSS system design question

---

## Key Takeaways

- TLS 1.3 reduces handshake to 1-RTT (vs 2-RTT for TLS 1.2), enforces forward secrecy for every session, and removes all weak cipher suites — know these specifics
- mTLS adds client certificate authentication, enabling cryptographic identity for service-to-service connections — required for zero-trust and many compliance frameworks
- SYN cookies save memory by encoding state in the ISN, but cost you TCP option support (window scaling, SACK) — a meaningful throughput trade-off
- Root CA compromise means the attacker can impersonate any site — recovery takes months through OS/browser trust store updates, which is why the root is kept offline
- PCI-DSS auditors check: network diagrams, justified firewall rules, proven CDE isolation (reachability tests), and change management for every rule
