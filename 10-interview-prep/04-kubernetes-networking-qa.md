# Kubernetes Networking — Interview Q&A

## Quick Reference

Kubernetes networking interviews test whether you understand that K8s networking is entirely built on Linux primitives — veth pairs, iptables/eBPF, network namespaces, and routing tables. Every "K8s networking issue" is ultimately a Linux networking issue. The key questions interviewers ask: can you trace a packet from pod A to pod B step by step? Can you debug a NetworkPolicy without guessing? Do you understand why kube-proxy iptables mode breaks at scale? Source material: `06_Cross_Domain_Integration.md` and `02-linux-networking/`. See `04-kubernetes-networking/` for deeper reading.

---

## Q: Trace the complete packet path from a pod on Node A to a pod on Node B. Include all interfaces, tables, and decisions.

**What the interviewer is testing:** Whether you can mentally execute the kernel's packet processing for cross-node pod communication.

**Model Answer:**

Example: Pod A (10.244.1.5) on Node A → Pod B (10.244.2.8) on Node B. Using Calico BGP mode (native routing, no overlay).

**On Node A (source side):**
```
Pod A network namespace
  eth0 (pod interface): 10.244.1.5
  ↓ packet leaves pod via veth pair
  ↓ exits pod namespace into Node A's network namespace

Node A host namespace
  cali<hash> (veth host end)
  ↓ ip route lookup: "10.244.2.8 via 192.168.1.11 dev eth0 proto bird"
    (this route was installed by Calico's BGP daemon from Node B's advertisement)
  ↓ NF_INET_PRE_ROUTING (conntrack lookup)
  ↓ NF_INET_FORWARD (Calico iptables NetworkPolicy check)
  ↓ eth0: Node A's physical NIC
  ↓ packet on wire with src=10.244.1.5, dst=10.244.2.8
```

**Physical network:**
```
  Node A eth0 → switch → Node B eth0
  (switch must allow pod CIDR traffic — not just node subnet)
  (on AWS: source/dest check must be disabled on EC2 instances)
```

**On Node B (destination side):**
```
  eth0: receives packet with dst=10.244.2.8
  ↓ ip route lookup: "10.244.2.0/24 dev cali<hash2> proto bird"
    (Calico added this route when Pod B was scheduled on Node B)
  ↓ NF_INET_LOCAL_IN (actually FORWARD since it's being forwarded to pod)
  ↓ NF_INET_FORWARD (Calico NetworkPolicy check for Pod B)
  ↓ cali<hash2> (veth host end)
  ↓ enters Pod B's network namespace

Pod B network namespace
  eth0: 10.244.2.8 — packet delivered
  ↓ tcp_v4_rcv() → socket receive queue → application recv()
```

**Key decision points where failures occur:**
1. BGP route missing on Node A: `ip route show | grep 10.244.2.0` — no route found
2. Anti-spoofing on underlay: switch or cloud drops packets from unexpected src IP
3. IP forwarding disabled: `sysctl net.ipv4.ip_forward` must be 1 on ALL nodes
4. NetworkPolicy iptables rule: `iptables -L FORWARD -n -v | grep DROP`
5. Route missing on Node B: pod CIDR not advertised back to Node A

**With overlay (VXLAN mode):**
- Node A encapsulates the pod packet in UDP/VXLAN: outer src=NodeA_IP, outer dst=NodeB_IP
- Physical network only sees node IPs (no anti-spoofing concern)
- VXLAN adds 50 bytes overhead — inner MTU must be reduced

**Follow-up Questions:**
- How does the path differ with VXLAN encapsulation vs native routing?
- What is the role of Felix in Calico and what does it program?
- How does BIRD BGP daemon maintain the cross-node routing table?

**Connecting Points:**
- `10-interview-prep/07-cross-domain-scenarios.md` for Calico BGP debugging scenario
- `04-kubernetes-networking/` for Calico architecture deep dive

---

## Q: How does kube-proxy implement ClusterIP? Walk through the iptables rules for a service with 3 endpoints.

**What the interviewer is testing:** Whether you understand that ClusterIP is a DNAT illusion implemented in conntrack.

**Model Answer:**

ClusterIP (e.g., 10.96.45.200:80) is not a real IP address assigned to any interface. It's a virtual IP that exists only in iptables/conntrack rules. When a pod sends a packet to 10.96.45.200:80, kube-proxy's DNAT rules intercept it and rewrite the destination to one of the real endpoint IPs before the packet leaves the pod's network namespace.

**Rule structure (3 endpoints: 10.244.1.5:8080, 10.244.2.8:8080, 10.244.3.10:8080):**
```bash
# Chain 1: Entry point in KUBE-SERVICES
-A KUBE-SERVICES -d 10.96.45.200/32 -p tcp --dport 80 -j KUBE-SVC-<HASH>

# Chain 2: Probabilistic load balancing
-A KUBE-SVC-<HASH> -m statistic --mode random --probability 0.33333 \
  -j KUBE-SEP-EP1-<HASH>
-A KUBE-SVC-<HASH> -m statistic --mode random --probability 0.50000 \
  -j KUBE-SEP-EP2-<HASH>
-A KUBE-SVC-<HASH> \
  -j KUBE-SEP-EP3-<HASH>
# Probability math: 1/3, then 1/2 of remaining (= 1/3), then all remaining (= 1/3)

# Chain 3: DNAT to actual endpoint
-A KUBE-SEP-EP1-<HASH> -p tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-EP2-<HASH> -p tcp -j DNAT --to-destination 10.244.2.8:8080
-A KUBE-SEP-EP3-<HASH> -p tcp -j DNAT --to-destination 10.244.3.10:8080
```

**What conntrack does:** After the DNAT rule fires, conntrack creates an entry recording the translation: `10.244.1.50:54321 → 10.96.45.200:80 becomes 10.244.1.50:54321 → 10.244.1.5:8080`. Return packets from the endpoint are automatically reverse-NATted back to 10.96.45.200:80 using the conntrack entry.

**Why this breaks at scale:**
- Each packet to a ClusterIP traverses KUBE-SERVICES (all services) + KUBE-SVC (all endpoints) — worst case O(services × endpoints) rules per packet
- At 10,000 services: KUBE-SERVICES chain has 10,000 rules; worst case 10,000 evaluations per packet
- kube-proxy iptables sync: updating rules for a single endpoint requires a full iptables-restore of ALL rules (atomic replacement)

```bash
# View actual kube-proxy rules
iptables -L -t nat -n | grep KUBE-
# Count rules
iptables -L KUBE-SERVICES -n | wc -l
```

**Follow-up Questions:**
- Why does kube-proxy need the MASQUERADE rule in the POSTROUTING chain?
- How does kube-proxy IPVS mode differ from iptables mode for the same service?
- What happens to existing connections when an endpoint is removed?

**Connecting Points:**
- `10-interview-prep/02-linux-networking-qa.md` for kube-proxy scaling breakdown
- `04-kubernetes-networking/` for kube-proxy implementation

---

## Q: Why is Cilium eBPF better than kube-proxy at scale? Be specific about the mechanism.

**What the interviewer is testing:** Whether you can explain eBPF advantages with data structures and numbers, not just "it's faster."

**Model Answer:**

Cilium replaces kube-proxy with eBPF programs attached to the kernel's cgroup and TC hooks. The improvement is architectural, not incremental.

**kube-proxy iptables:** O(n) sequential rule evaluation, O(n) full-replace updates
**Cilium eBPF:** O(1) BPF map lookups, O(1) map entry updates (no lock for read path)

**Mechanism — Service lookup:**

kube-proxy DNAT flow:
```
Packet → traverse KUBE-SERVICES (n rules) → match service → traverse KUBE-SVC (m rules) → DNAT
Total: O(n × m) rule evaluations
```

Cilium BPF flow:
```
Socket connect() syscall → cgroup-bpf hook → BPF hash map lookup: key=(VIP, port) → value=(endpoint list)
→ select endpoint → rewrite destination in socket → NO DNAT, NO conntrack
Total: O(1) map lookup
```

The critical insight: Cilium does socket-level load balancing, not packet-level DNAT. The destination is rewritten BEFORE the TCP handshake. The kernel never creates a conntrack DNAT entry. The socket directly connects to the endpoint's real IP.

**BPF map types used:**
```c
// Service table: ClusterIP:port → service ID
BPF_MAP_TYPE_HASH: key=(svc_ip, svc_port, proto), value=service_entry

// Endpoint table: service_id → endpoint list
BPF_MAP_TYPE_ARRAY or BPF_MAP_TYPE_LRU_HASH: key=service_id, value=[endpoints]

// Policy table: pod identity → allowed connections
BPF_MAP_TYPE_LPM_TRIE: for CIDR-based policy
BPF_MAP_TYPE_HASH: for identity-based policy
```

**Update performance:**
- kube-proxy: endpoint added → full iptables-restore (10,000 rules = ~11 seconds)
- Cilium: endpoint added → single BPF map entry update (< 1 millisecond, lock-free on read path)

**Additional advantages:**
- Eliminates conntrack for pod-to-service traffic (massive reduction in conntrack table pressure)
- NetworkPolicy enforcement in BPF (O(1) identity lookup vs O(n) iptables chain)
- XDP-based load balancing for north-south traffic (processes before sk_buff allocation)
- Hubble: eBPF-based observability with zero application changes

**Concrete scale numbers:**
- 10,000 services: kube-proxy sync takes 11s; Cilium update takes < 100ms
- 100,000 pods: conntrack table has 100K+ DNAT entries with kube-proxy; Cilium has near-zero

**Follow-up Questions:**
- What kernel version does Cilium require for full eBPF mode?
- What is Cilium's eBPF host routing mode and how does it bypass kube-proxy entirely?
- How does Cilium's Maglev consistent hashing differ from kube-proxy's random selection?

**Connecting Points:**
- `10-interview-prep/09-exotic-deep-dive-qa.md` for eBPF internals
- `09-advanced-topics/` for Cilium architecture

---

## Q: A NetworkPolicy is blocking traffic. How do you debug it without guessing?

**What the interviewer is testing:** Whether you have a systematic approach to NetworkPolicy debugging, not just "delete the policy and see."

**Model Answer:**

NetworkPolicy debugging requires understanding that the CNI plugin (not Kubernetes itself) implements the policy. The same NetworkPolicy YAML results in different iptables/eBPF rules depending on whether you use Calico, Cilium, or Flannel.

**Step 1 — Verify the policy exists and matches the pods:**
```bash
# List all NetworkPolicies affecting a specific pod
kubectl get networkpolicies -n <namespace> -o yaml

# Verify labels match — policies select by label
kubectl get pod <pod> -n <namespace> --show-labels
# podSelector in the NetworkPolicy must match these labels

# Check effective policies for a pod
kubectl describe pod <pod> -n <namespace>
# Look for "annotations" showing applied policies
```

**Step 2 — Test connectivity to isolate the direction:**
```bash
# From blocked pod, test outbound
kubectl exec -n <ns> <blocked-pod> -- curl -v --connect-timeout 5 http://<target-pod-ip>:8080

# From allowed pod, test to blocked pod
kubectl exec -n <ns> <allowed-pod> -- curl -v --connect-timeout 5 http://<blocked-pod-ip>:8080

# Key: timeout (policy drop) vs "connection refused" (app not listening) vs success
```

**Step 3 — Examine the actual firewall rules (CNI-specific):**

For Calico:
```bash
# Check Calico policy programming on the node
kubectl exec -n calico-system calico-node-<hash> -- calico-typha-client show policy

# Check iptables rules Felix programmed
iptables -L cali-fw-<pod-interface> -n -v
# cali-fw = forward, cali-to = to pod, cali-from = from pod
iptables -L cali-from-wl-dispatch -n -v
```

For Cilium:
```bash
# Check Cilium's view of policy for a specific pod endpoint
kubectl exec -n kube-system cilium-<hash> -- cilium endpoint list
kubectl exec -n kube-system cilium-<hash> -- cilium endpoint get <endpoint-id>

# Monitor live traffic decisions
kubectl exec -n kube-system cilium-<hash> -- \
  cilium monitor --from <endpoint-id> --type drop

# Check policy verdicts
kubectl exec -n kube-system cilium-<hash> -- cilium policy get
```

**Step 4 — Common NetworkPolicy gotchas:**
```yaml
# Gotcha 1: Empty podSelector = matches ALL pods in namespace
podSelector: {}  # This is intentional but often surprising

# Gotcha 2: Missing namespaceSelector means the policy applies to OWN namespace only
from:
  - podSelector:
      matchLabels:
        app: frontend  # Only allows pods WITH THIS LABEL in THIS NAMESPACE

# Gotcha 3: Default deny with no allow rules
# A NetworkPolicy with no ingress/egress rules but a podSelector
# DENIES ALL traffic to those pods — no rules means no allows

# Gotcha 4: AND vs OR in selectors
from:
  - podSelector:          # This is OR (two separate list items)
      matchLabels:
        app: frontend
  - namespaceSelector:
      matchLabels:
        tier: prod
# vs:
from:
  - podSelector:          # This is AND (combined in one list item)
      matchLabels:
        app: frontend
    namespaceSelector:
      matchLabels:
        tier: prod
```

**Step 5 — Verify DNS is also allowed:**
```bash
# Common mistake: block all egress but forget port 53
# All pods need egress to kube-dns IP (typically 10.96.0.10) port 53 UDP/TCP
kubectl exec <pod> -- nslookup kubernetes.default    # fails if DNS egress blocked
```

**Follow-up Questions:**
- How do you implement a default-deny-all policy for a namespace?
- How does Cilium's identity-based policy differ from IP-based NetworkPolicy?
- What is the difference between NetworkPolicy and CiliumNetworkPolicy?

**Connecting Points:**
- `04-kubernetes-networking/` for NetworkPolicy specification
- `10-interview-prep/07-cross-domain-scenarios.md` for Cilium policy debugging scenario

---

## Q: What causes ndots:5 DNS latency in Kubernetes and what are the trade-offs of fixing it?

**What the interviewer is testing:** Whether you understand DNS query amplification and can articulate the trade-offs of each fix.

**Model Answer:**

Kubernetes pods by default have `ndots:5` in `/etc/resolv.conf` with several search domains:

```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

`ndots:5` means: if a name has fewer than 5 dots, append each search domain and try before trying the absolute name.

For an external query like `api.stripe.com` (2 dots, less than 5), the resolver generates:
```
1. api.stripe.com.default.svc.cluster.local  → NXDOMAIN
2. api.stripe.com.svc.cluster.local          → NXDOMAIN
3. api.stripe.com.cluster.local              → NXDOMAIN
4. api.stripe.com.                           → SUCCESS (absolute FQDN)
```

Instead of 1 query, you make 4. Latency: ~4× the baseline DNS latency per resolution.

**The actual latency source:** The first 3 queries go to CoreDNS, which forwards them to the upstream resolver. Each query is a UDP round trip. At 10ms per query: 3 × 10ms = 30ms added latency before the real query.

**Fix options with trade-offs:**

**Option 1: Append trailing dot (absolute FQDN):**
```
curl https://api.stripe.com./endpoint    # trailing dot = no search domain expansion
```
Pros: No infra change. Cons: requires application code change; breaks many libraries.

**Option 2: Reduce ndots:**
```yaml
# In pod spec
dnsConfig:
  options:
    - name: ndots
      value: "2"
```
With `ndots:2`, `api.stripe.com` (2 dots) is tried as absolute first.
Pros: Eliminates external DNS amplification. Cons: breaks short internal service names like `database` or `api` (1 dot) — they now skip search domains.

**Option 3: NodeLocal DNSCache (recommended for production):**
```bash
# Deploy NodeLocal DNSCache DaemonSet
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```
A DNS cache runs on each node (link-local 169.254.20.10). Pods query the local cache, which:
- Caches positive responses (reduces CoreDNS load by 90%+)
- Serves stale records for ndots:5 queries from cache after first resolution
- Avoids conntrack UDP race condition (uses TCP to CoreDNS)

Pros: Eliminates repeat latency (cache hit < 1ms), reduces CoreDNS pressure, no application changes.
Cons: Another DaemonSet to manage; cache staleness for short-TTL records; link-local address requires care in some environments.

**Option 4: Single-request-reopen:**
```yaml
dnsConfig:
  options:
    - name: single-request-reopen
# Sends A and AAAA queries sequentially instead of simultaneously
# Avoids conntrack UDP race condition but adds serial query latency
```

**The conntrack race condition (separate but related):**
```bash
conntrack -S | grep insert_failed    # if incrementing, race condition confirmed
# Fix: NodeLocal DNSCache (bypasses conntrack) or single-request-reopen
```

**Follow-up Questions:**
- How does NodeLocal DNSCache interact with the conntrack table?
- What is the DNS lookup flow with NodeLocal DNSCache deployed?
- How would you scale CoreDNS for a 5,000-node cluster?

**Connecting Points:**
- `04-kubernetes-networking/` for CoreDNS and ndots configuration
- `10-interview-prep/07-cross-domain-scenarios.md` for high DNS latency root cause scenario

---

## Q: Istio mTLS — how does it work without changing application code?

**What the interviewer is testing:** Whether you understand iptables traffic interception and the sidecar injection mechanism.

**Model Answer:**

Istio mTLS works through two mechanisms: sidecar injection and iptables traffic interception. The application never knows it's involved in TLS.

**Step 1 — Sidecar injection:**
When a pod is created in a namespace with `istio-injection: enabled`, the Istio mutating webhook automatically adds an Envoy sidecar container and an init container to the pod:
- `istio-init`: runs iptables commands during pod startup
- `istio-proxy` (Envoy): handles all inbound and outbound traffic

**Step 2 — iptables traffic interception (via istio-init):**
```bash
# istio-init runs these commands:
# Redirect all outbound traffic from the pod to Envoy (port 15001)
iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-port 15001
# Exception: Envoy's own traffic (UID 1337 = Envoy's UID) is not redirected
iptables -t nat -A OUTPUT -m owner --uid-owner 1337 -j RETURN

# Redirect all inbound traffic to Envoy (port 15006)
iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-port 15006
```

The application thinks it's connecting to `service-b:8080`. The kernel intercepts this at the `OUTPUT` hook and redirects it to Envoy on port 15001. The application's TCP handshake completes with Envoy, not with the remote service.

**Step 3 — mTLS between Envoy sidecars:**
- Source Envoy (port 15001) receives the application's plaintext connection
- Envoy establishes a TLS 1.3 connection to the destination Envoy (port 15006 on the target pod)
- Both Envoys present certificates issued by Istio's CA (mounted as secrets into the pod)
- The destination Envoy terminates TLS and forwards plaintext to the destination application

```
App A → [iptables REDIRECT] → Envoy A (outbound)
         [mTLS via Envoy to Envoy]
                               Envoy B (inbound) → [iptables REDIRECT] → App B
```

**Certificate rotation:**
```bash
# Certificates are rotated by istiod every 24 hours by default
# Envoy hot-reloads certificates without restart via SDS (Secret Discovery Service)
# istioctl check-inject -n <namespace>  # verify injection
# istioctl proxy-status                 # check Envoy sync status
```

**mTLS policy:**
```yaml
# Enforce STRICT mTLS (reject plaintext connections)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

**Follow-up Questions:**
- What happens during certificate rotation if Envoy can't reach istiod?
- How does Istio handle connections from non-mesh services (no sidecar)?
- What is the difference between PeerAuthentication and AuthorizationPolicy?

**Connecting Points:**
- `05-network-security/` for mTLS certificate management
- `10-interview-prep/07-cross-domain-scenarios.md` for mTLS blocking traffic after rotation

---

## Q: StatefulSet pod DNS — why does it need a headless service and how does it work?

**What the interviewer is testing:** Whether you understand DNS-based StatefulSet identity and why it's needed for stateful workloads.

**Model Answer:**

A StatefulSet pod needs a stable network identity across restarts. If the pod is deleted and recreated, it gets a new IP — but with a headless service, its DNS name remains stable.

**Regular Service DNS:** `my-service.namespace.svc.cluster.local` → ClusterIP (e.g., 10.96.45.200) → any healthy pod. This is correct for stateless services. But for a database cluster, you need to reach pod-0 specifically (the primary), not just any pod.

**Headless Service (clusterIP: None):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None       # This makes it headless
  selector:
    app: postgres
  ports:
    - port: 5432
```

A headless service has NO ClusterIP. DNS returns the actual pod IPs instead of a virtual IP:

```
DNS query: postgres.default.svc.cluster.local
Regular service: → 10.96.45.200 (ClusterIP) — load balanced
Headless service: → 10.244.1.5, 10.244.2.8, 10.244.3.10 (ALL pod IPs)
```

**StatefulSet pod DNS (stable individual pod names):**

With a StatefulSet named `postgres` and headless service `postgres`:
```
postgres-0.postgres.default.svc.cluster.local → 10.244.1.5
postgres-1.postgres.default.svc.cluster.local → 10.244.2.8
postgres-2.postgres.default.svc.cluster.local → 10.244.3.10
```

If `postgres-0` pod is deleted and recreated: its IP changes, but `postgres-0.postgres.default.svc.cluster.local` still resolves to the new pod's IP. The primary identity (pod-0) is preserved.

**Implementation in CoreDNS:**
```
CoreDNS serves the cluster.local zone.
For headless services, CoreDNS creates an A record for each pod endpoint.
For StatefulSet pods, it creates A records for each pod: <pod-name>.<service-name>.<ns>.svc.cluster.local
```

```bash
# Verify StatefulSet DNS from another pod
kubectl exec -it some-pod -- nslookup postgres-0.postgres.default.svc.cluster.local
kubectl exec -it some-pod -- dig postgres-0.postgres.default.svc.cluster.local

# Check all endpoints
kubectl exec -it some-pod -- dig postgres.default.svc.cluster.local
# Should return all pod IPs
```

**Use case — database replication:**
```
Primary:  postgres-0 — clients connect to postgres-0.postgres.default.svc.cluster.local
Replicas: postgres-1, postgres-2 — replication connections use stable DNS names
# After postgres-0 restarts: DNS updates, clients reconnect to new IP via same name
```

**Follow-up Questions:**
- How does a StatefulSet pod find its own stable hostname?
- What is the DNS name format for an init container that runs before the main container?
- How does StatefulSet ordered pod management (pod-0 before pod-1) interact with readiness probes?

**Connecting Points:**
- `04-kubernetes-networking/` for CoreDNS and DNS service discovery
- `10-interview-prep/07-cross-domain-scenarios.md` for DNS latency scenarios

---

## Key Takeaways

- Pod-to-pod traffic crosses: veth pair → host routing table → physical network → host routing table → veth pair — BGP routes or VXLAN FDB entries connect the two nodes
- ClusterIP is a DNAT illusion: no real IP, just iptables DNAT rules backed by conntrack tracking the translation
- Cilium's eBPF advantage is architectural: socket-level load balancing at O(1) vs kube-proxy's O(n×m) iptables traversal, AND no conntrack entries for pod-to-service traffic
- ndots:5 causes 4-6 DNS queries for every external domain — NodeLocal DNSCache is the production fix
- StatefulSet headless service provides stable per-pod DNS names across IP changes — essential for databases, Kafka, and any stateful distributed system
