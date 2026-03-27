# Lab 02: Kubernetes Networking

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Prerequisites](#prerequisites)
- [Debugging Methodology Alignment](#debugging-methodology-alignment)
- [Lab 2A: Trace CNI Pod-to-Pod Packet Path](#lab-2a-trace-cni-pod-to-pod-packet-path)
  - [Setup: Deploy Two Pods on Different Nodes](#setup-deploy-two-pods-on-different-nodes)
  - [Trace: Map the Network Path](#trace-map-the-network-path)
  - [Verify with tcpdump at Each Hop](#verify-with-tcpdump-at-each-hop)
- [Lab 2B: Reproduce and Fix ndots:5 DNS Latency](#lab-2b-reproduce-and-fix-ndots5-dns-latency)
  - [Background](#background)
  - [Setup and Baseline Measurement](#setup-and-baseline-measurement)
  - [The Break: Observe the Actual DNS Queries](#the-break-observe-the-actual-dns-queries)
  - [Symptoms](#symptoms)
  - [Diagnosis](#diagnosis)
  - [Fix Option 1: Set ndots:1 in Pod Spec](#fix-option-1-set-ndots1-in-pod-spec)
  - [Fix Option 2: Use FQDN with Trailing Dot](#fix-option-2-use-fqdn-with-trailing-dot)
  - [Before/After Comparison](#beforeafter-comparison)
- [Lab 2C: NetworkPolicy Break/Fix — Missing DNS Egress](#lab-2c-networkpolicy-breakfix-missing-dns-egress)
  - [Setup: Deploy Frontend and Backend](#setup-deploy-frontend-and-backend)
  - [The Break: Apply Broken NetworkPolicy](#the-break-apply-broken-networkpolicy)
  - [Symptoms](#symptoms)
  - [Diagnosis](#diagnosis)
  - [Root Cause](#root-cause)
  - [Fix](#fix)
  - [Verify](#verify)
  - [Cleanup](#cleanup)
- [Lab 2D: kube-proxy iptables Rules Inspection](#lab-2d-kube-proxy-iptables-rules-inspection)
  - [Setup: Deploy a ClusterIP Service](#setup-deploy-a-clusterip-service)
  - [Inspect kube-proxy iptables Rules](#inspect-kube-proxy-iptables-rules)
  - [IPVS Mode (Alternative)](#ipvs-mode-alternative)
  - [Verify Load Distribution](#verify-load-distribution)
  - [Cleanup](#cleanup)
- [Summary: Kubernetes Networking Lab Checklist](#summary-kubernetes-networking-lab-checklist)
- [Interview Discussion Points](#interview-discussion-points)

---

## Learning Objectives

- Trace the exact packet path for pod-to-pod traffic across nodes — every hop, every interface
- Reproduce and measure the ndots:5 DNS latency tax that affects every K8s workload
- Build and debug NetworkPolicy rules including the common "forgot DNS egress" mistake
- Inspect the iptables/IPVS rules kube-proxy creates for ClusterIP services

## Prerequisites

- Kubernetes cluster: 2+ nodes (use `kind`, `kubeadm`, GKE, or EKS)
- Tools: `kubectl`, `tcpdump`, `curl`, `dig`/`nslookup`, `ip`, `bridge-utils`
- For kind multi-node: `kind create cluster --config kind-multinode.yaml`
- CNI: Flannel or Calico (Lab 2A assumes Flannel/VXLAN; note differences for Calico)
- For Lab 2D IPVS mode: `sudo apt-get install ipvsadm`

```yaml
# kind-multinode.yaml — paste this to create a 3-node cluster
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-multinode.yaml --name net-lab
export KUBECONFIG=$(kind get kubeconfig-path --name net-lab 2>/dev/null || echo ~/.kube/config)
```

## Debugging Methodology Alignment

These labs reinforce the OSI model traversal from `07-debugging-playbooks/00-debugging-methodology.md`. In Kubernetes:
- L2: veth pair inside node, cni0/flannel.1 bridge
- L3: pod CIDR routing, VXLAN encapsulation between nodes
- L4: kube-proxy iptables/IPVS rules implementing ClusterIP load balancing
- L5: DNS search domain expansion causing application-level latency

---

## Lab 2A: Trace CNI Pod-to-Pod Packet Path

**Objective:** Follow a single packet from pod-A on node-1 to pod-B on node-2, identifying every interface it crosses, so you can place tcpdump captures correctly during incidents.

---

### Setup: Deploy Two Pods on Different Nodes

```bash
# Get node names
kubectl get nodes -o wide
# Expected:
# NAME                    STATUS   ROLES    AGE   VERSION   INTERNAL-IP
# net-lab-control-plane   Ready    master   5m    v1.28.0   172.18.0.2
# net-lab-worker          Ready    <none>   4m    v1.28.0   172.18.0.3
# net-lab-worker2         Ready    <none>   4m    v1.28.0   172.18.0.4

# Store node names
NODE1=$(kubectl get nodes --no-headers | grep worker | head -1 | awk '{print $1}')
NODE2=$(kubectl get nodes --no-headers | grep worker | tail -1 | awk '{print $1}')
echo "Node1: $NODE1, Node2: $NODE2"

# Deploy pod-a pinned to node1
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  labels:
    app: pod-a
spec:
  nodeName: $NODE1
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

# Deploy pod-b pinned to node2
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
  labels:
    app: pod-b
spec:
  nodeName: $NODE2
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

kubectl wait --for=condition=Ready pod/pod-a pod/pod-b --timeout=60s
```

### Trace: Map the Network Path

```bash
# Step 1: Get pod IPs
POD_A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')
POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')
echo "pod-a IP: $POD_A_IP (on $NODE1)"
echo "pod-b IP: $POD_B_IP (on $NODE2)"
# Expected:
# pod-a IP: 10.244.1.5 (on net-lab-worker)
# pod-b IP: 10.244.2.7 (on net-lab-worker2)

# Step 2: Inside pod-a — what interface and route does it use?
kubectl exec pod-a -- ip addr show eth0
# Expected:
# 2: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet 10.244.1.5/24 brd 10.244.1.255 scope global eth0

kubectl exec pod-a -- ip route show
# Expected:
# default via 10.244.1.1 dev eth0
# 10.244.0.0/16 via 10.244.1.1 dev eth0   (Flannel route to all pods)

# Step 3: On node1 — find the veth that connects to pod-a
# The pod's eth0 is one end of a veth pair; the other end is on the host
docker exec $NODE1 ip link show type veth
# Expected:
# 7: veth3a2b1c4@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> master cni0
#     (note: if2 = interface index 2 inside the pod = eth0 inside pod-a)

# Step 4: On node1 — show the bridge (cni0 for Flannel)
docker exec $NODE1 ip addr show cni0
# Expected:
# cni0: inet 10.244.1.1/24 — this is the gateway pod-a uses

docker exec $NODE1 bridge fdb show dev cni0
# Shows which MACs are known to the bridge

# Step 5: On node1 — how does traffic destined for node2's pod CIDR get routed?
docker exec $NODE1 ip route show
# Expected (Flannel VXLAN):
# 10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
# This means: traffic for pod-b's subnet exits via flannel.1 (VXLAN interface)

# Step 6: What is flannel.1?
docker exec $NODE1 ip -d link show flannel.1
# Expected:
# flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> ... vxlan id 1 local 172.18.0.3 dev eth0

# VXLAN: original packet is encapsulated in UDP (port 8472) and sent to node2
# Outer packet: SRC=172.18.0.3 (node1) DST=172.18.0.4 (node2) UDP dport 8472

# Step 7: On node2 — VXLAN arrives on flannel.1, decapsulated, routed to pod-b
docker exec $NODE2 ip route show
# Expected:
# 10.244.2.0/24 dev cni0 proto kernel   <-- local pod CIDR, go to bridge
# Traffic arrives on flannel.1, decapsulated to original pod IP, routed via cni0 to pod-b
```

### Verify with tcpdump at Each Hop

```bash
# Window 1: Capture on pod-a's veth on node1
docker exec $NODE1 bash -c "tcpdump -i veth3a2b1c4 -n 'icmp' -c 5" &

# Window 2: Capture VXLAN encapsulated traffic on node1's eth0
docker exec $NODE1 bash -c "tcpdump -i eth0 -n 'udp port 8472' -c 5" &

# Window 3: Capture on pod-b's veth on node2
VETH_B=$(docker exec $NODE2 ip link show type veth | grep -o 'veth[^ :@]*' | head -1)
docker exec $NODE2 bash -c "tcpdump -i $VETH_B -n 'icmp' -c 5" &

# Trigger: ping from pod-a to pod-b
kubectl exec pod-a -- ping -c 3 $POD_B_IP

# Expected path summary:
# pod-a eth0 -> veth(node1) -> cni0(bridge) -> flannel.1(VXLAN) -> eth0(node1)
#   [UDP 8472 across node network]
# eth0(node2) -> flannel.1(decap) -> cni0(bridge) -> veth(node2) -> pod-b eth0
```

**Key Takeaway:** Pod-to-pod cross-node traffic is: `veth -> bridge -> VXLAN encap -> UDP across node network -> VXLAN decap -> bridge -> veth`. Place tcpdump on `flannel.1` (or `tunl0` for Calico IPIP) to see cross-node pod traffic. Place it on the specific veth to see a single pod's traffic.

---

## Lab 2B: Reproduce and Fix ndots:5 DNS Latency

**Objective:** Measure and demonstrate the performance impact of the default `ndots:5` DNS search domain expansion, then fix it.

---

### Background

Every pod has `/etc/resolv.conf` injected by kubelet with:
```
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```
`ndots:5` means: if a hostname has fewer than 5 dots, try each search domain first before treating it as absolute. For `google.com` (1 dot), the resolver tries 4 queries (all NXDOMAIN) before the real query succeeds.

### Setup and Baseline Measurement

```bash
# Exec into a running pod to inspect DNS config
kubectl exec pod-a -- cat /etc/resolv.conf
# Expected:
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# Measure time for external DNS resolution (ndots:5 forces search expansion)
kubectl exec pod-a -- time nslookup google.com
# Expected output (pay attention to time):
# Server:   10.96.0.10
# Address:  10.96.0.10#53
# Non-authoritative answer:
# Name: google.com
# Address: 142.250.x.x
# real    0m0.087s   <-- seems fast but...
```

### The Break: Observe the Actual DNS Queries

```bash
# Capture DNS traffic from inside the pod (in another terminal):
# Find which node pod-a is on and capture on that node
NODE1=$(kubectl get pod pod-a -o jsonpath='{.spec.nodeName}')

# On node1, capture DNS (port 53 UDP) traffic
docker exec $NODE1 tcpdump -i any -n 'udp port 53' -c 20 2>/dev/null &
TCPDUMP_PID=$!

# Trigger DNS resolution from pod-a
kubectl exec pod-a -- nslookup google.com

# Stop capture
sleep 2; kill $TCPDUMP_PID 2>/dev/null

# Expected tcpdump output (6 queries for 1 hostname!):
# Query:  google.com.default.svc.cluster.local  -> NXDOMAIN
# Query:  google.com.svc.cluster.local          -> NXDOMAIN
# Query:  google.com.cluster.local              -> NXDOMAIN
# Query:  google.com.                           -> SUCCESS (A record)
# (Plus AAAA queries for IPv6, doubling the count to 8 total round trips)
```

### Symptoms

```bash
# Quantify the latency penalty by running many resolutions
kubectl exec pod-a -- bash -c '
for i in $(seq 1 10); do
  start=$(date +%s%N)
  nslookup google.com > /dev/null 2>&1
  end=$(date +%s%N)
  echo "Resolution $i: $(( (end - start) / 1000000 ))ms"
done'
# Expected: each resolution takes 30-100ms due to 3 NXDOMAIN round trips
```

### Diagnosis

```bash
# Confirm ndots setting
kubectl exec pod-a -- cat /etc/resolv.conf | grep ndots
# Expected: options ndots:5

# Trace actual search domain expansion
kubectl exec pod-a -- dig google.com +search +stats 2>/dev/null | grep -E "Query time|status"
# Expected: status: NOERROR, but Query time includes prior NXDOMAIN overhead
```

### Fix Option 1: Set ndots:1 in Pod Spec

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-a-fixed
spec:
  nodeName: $NODE1
  dnsConfig:
    options:
      - name: ndots
        value: "1"
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

kubectl wait --for=condition=Ready pod/pod-a-fixed --timeout=60s

# Verify new resolv.conf
kubectl exec pod-a-fixed -- cat /etc/resolv.conf
# Expected:
# options ndots:1  (reduced from 5)

# Now measure DNS resolution — much faster
kubectl exec pod-a-fixed -- bash -c '
for i in $(seq 1 10); do
  start=$(date +%s%N)
  nslookup google.com > /dev/null 2>&1
  end=$(date +%s%N)
  echo "Resolution $i: $(( (end - start) / 1000000 ))ms"
done'
# Expected: 5-15ms (direct query, no search expansion)
```

### Fix Option 2: Use FQDN with Trailing Dot

```bash
# Adding a trailing dot forces absolute lookup, bypassing ndots entirely
kubectl exec pod-a -- time nslookup google.com.
# Expected: direct query, no search domain expansion, faster response
```

### Before/After Comparison

```bash
# Run in pod-a (ndots:5) vs pod-a-fixed (ndots:1)
echo "=== ndots:5 (default) ==="
kubectl exec pod-a -- bash -c 'time for i in $(seq 1 20); do nslookup google.com > /dev/null 2>&1; done'

echo "=== ndots:1 (fixed) ==="
kubectl exec pod-a-fixed -- bash -c 'time for i in $(seq 1 20); do nslookup google.com > /dev/null 2>&1; done'
# Expected: ndots:1 is 3-5x faster for external names
```

**Key Takeaway:** The `ndots:5` default causes 3-4 wasted DNS round trips for every external hostname lookup. For latency-sensitive services, set `ndots: 1` in dnsConfig. For internal K8s service names, use FQDNs (`svc.namespace.svc.cluster.local`) to avoid ambiguity.

---

## Lab 2C: NetworkPolicy Break/Fix — Missing DNS Egress

**Objective:** Reproduce the most common NetworkPolicy mistake: blocking DNS egress while trying to restrict pod-to-pod traffic, causing mysterious application failures that look like network issues but are actually DNS failures.

---

### Setup: Deploy Frontend and Backend

```bash
# Create test namespace
kubectl create namespace netpol-lab

# Deploy backend
kubectl apply -n netpol-lab -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo
        args: ["-text=backend-response"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5678
EOF

# Deploy frontend
kubectl apply -n netpol-lab -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

kubectl wait -n netpol-lab --for=condition=Ready pod/frontend --timeout=60s
kubectl wait -n netpol-lab --for=condition=Available deployment/backend --timeout=60s

# Confirm connectivity works BEFORE applying any policy
kubectl exec -n netpol-lab frontend -- curl -s http://backend-svc/
# Expected: backend-response
```

### The Break: Apply Broken NetworkPolicy

```bash
# This policy intends to allow frontend->backend HTTP only
# BUG: it also restricts egress from frontend, but omits DNS (port 53)
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5678
EOF
```

### Symptoms

```bash
# This will hang or fail — but WHY is non-obvious
kubectl exec -n netpol-lab frontend -- curl -v --max-time 5 http://backend-svc/
# Expected failure:
# * Could not resolve host: backend-svc
# curl: (6) Could not resolve host: backend-svc

# Or it might say "Connection timed out" if curl retries
# The symptom looks like a network connectivity failure but it's DNS

# DNS is blocked too — confirm:
kubectl exec -n netpol-lab frontend -- nslookup backend-svc.netpol-lab.svc.cluster.local
# Expected:
# ;; connection timed out; no servers could be reached

# Using the ClusterIP directly (bypass DNS) — but this also fails
# because the NetworkPolicy only allows traffic to pods with app=backend
# and kube-dns pods don't have that label
BACKEND_IP=$(kubectl get svc -n netpol-lab backend-svc -o jsonpath='{.spec.clusterIP}')
kubectl exec -n netpol-lab frontend -- curl --max-time 5 http://$BACKEND_IP/
# May work (depends on whether IPVS/iptables routes ClusterIP correctly)
# But DNS will still not resolve
```

### Diagnosis

```bash
# Step 1: Confirm NetworkPolicy exists and is applied
kubectl get networkpolicy -n netpol-lab
# Expected: frontend-policy

# Step 2: Read the policy carefully
kubectl describe networkpolicy frontend-policy -n netpol-lab
# Expected output — look at Egress rules:
# Egress:
#   To: PodSelector: app=backend
#   Ports: TCP 5678
# Notice: NO rule for port 53 (DNS). Any egress not matching this rule is BLOCKED.

# Step 3: Confirm kube-dns namespace and pod labels
kubectl get pods -n kube-system -l k8s-app=kube-dns
# Expected: coredns pods with label k8s-app=kube-dns

kubectl get svc -n kube-system kube-dns
# Expected:
# NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
# kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   10m

# Step 4: Verify the frontend has NO other network policies
kubectl get networkpolicy -n netpol-lab -o yaml
# Only frontend-policy exists. No default-deny ingress, but egress IS restricted.
```

### Root Cause

Once `policyTypes: [Egress]` is set in any NetworkPolicy selecting frontend pods, ALL egress is denied by default except what's explicitly allowed. The policy allows TCP to backend on port 5678, but DNS (UDP+TCP port 53 to kube-dns) is not included. Without DNS, hostname resolution fails, making the service unreachable even though backend connectivity would otherwise work.

### Fix

```bash
# Replace the broken policy with one that includes a DNS egress rule
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    # Rule 1: Allow DNS to kube-dns in kube-system namespace
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Rule 2: Allow HTTP to backend pods in same namespace
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5678
EOF
```

### Verify

```bash
# DNS should work now
kubectl exec -n netpol-lab frontend -- nslookup backend-svc.netpol-lab.svc.cluster.local
# Expected:
# Server: 10.96.0.10
# Name: backend-svc.netpol-lab.svc.cluster.local
# Address: 10.96.x.x

# Service call should work
kubectl exec -n netpol-lab frontend -- curl -s http://backend-svc/
# Expected: backend-response

# Confirm that traffic to other destinations is still blocked
kubectl exec -n netpol-lab frontend -- curl --max-time 3 http://1.1.1.1 || echo "BLOCKED (expected)"
# Expected: connection timed out — internet egress is correctly blocked
```

### Cleanup

```bash
kubectl delete namespace netpol-lab
```

**Key Takeaway:** Any NetworkPolicy with `policyTypes: [Egress]` must include a DNS egress rule or ALL hostname resolution breaks. DNS uses UDP *and* TCP port 53. Always include both protocols. The symptom — "curl: could not resolve host" — looks like DNS is down, but the real cause is a NetworkPolicy silently blocking port 53.

---

## Lab 2D: kube-proxy iptables Rules Inspection

**Objective:** Find the exact iptables (or IPVS) rules kube-proxy creates for a ClusterIP service, understand how VIP load balancing works at the kernel level.

---

### Setup: Deploy a ClusterIP Service

```bash
kubectl create namespace kp-lab

kubectl apply -n kp-lab -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo
        args: ["-text=echo-$(hostname)"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: echo-svc
spec:
  selector:
    app: echo
  ports:
  - port: 80
    targetPort: 5678
  type: ClusterIP
EOF

kubectl wait -n kp-lab --for=condition=Available deployment/echo-server --timeout=60s

# Get the ClusterIP
CLUSTER_IP=$(kubectl get svc -n kp-lab echo-svc -o jsonpath='{.spec.clusterIP}')
echo "ClusterIP: $CLUSTER_IP"
# Expected: 10.96.x.x
```

### Inspect kube-proxy iptables Rules

```bash
# kube-proxy runs on each node. SSH/docker exec into a worker node:
docker exec net-lab-worker bash << 'NODEEOF'

# Step 1: Find the service entry in KUBE-SERVICES chain
CLUSTER_IP=$(kubectl get svc -n kp-lab echo-svc -o jsonpath='{.spec.clusterIP}' 2>/dev/null || echo "10.96.x.x")
iptables -t nat -L KUBE-SERVICES -n | grep "$CLUSTER_IP"
# Expected:
# KUBE-SVC-XXXXXXXXXXXXXXXX  tcp  --  0.0.0.0/0  10.96.3.45  tcp dpt:80
# This jumps to a service-specific chain (KUBE-SVC-...)

NODEEOF

# Step 2: Inspect the KUBE-SVC chain — contains load balancing rules
docker exec net-lab-worker bash -c "
  SVC_CHAIN=\$(iptables -t nat -L KUBE-SERVICES -n | grep '$CLUSTER_IP' | awk '{print \$3}')
  echo \"Service chain: \$SVC_CHAIN\"
  iptables -t nat -L \"\$SVC_CHAIN\" -n
"
# Expected:
# Chain KUBE-SVC-XXXXXXXXXXXXXXXX (1 references)
# KUBE-SEP-AAA  tcp  -- statistic mode random probability 0.33333333349
# KUBE-SEP-BBB  tcp  -- statistic mode random probability 0.50000000000
# KUBE-SEP-CCC  tcp  -- (no probability = catch-all, ~33% chance)
# Random probability creates equal load distribution across 3 endpoints

# Step 3: Inspect a KUBE-SEP (Service EndPoint) chain — actual DNAT rule
docker exec net-lab-worker bash -c "
  SVC_CHAIN=\$(iptables -t nat -L KUBE-SERVICES -n | grep '$CLUSTER_IP' | awk '{print \$3}')
  FIRST_SEP=\$(iptables -t nat -L \"\$SVC_CHAIN\" -n | grep KUBE-SEP | head -1 | awk '{print \$1}')
  echo \"Endpoint chain: \$FIRST_SEP\"
  iptables -t nat -L \"\$FIRST_SEP\" -n
"
# Expected:
# Chain KUBE-SEP-XXXXXXXXXXXXXXXX
# KUBE-MARK-MASQ  all  -- src=<pod-IP>  (marks hairpin traffic)
# DNAT            tcp  -- 0.0.0.0/0  0.0.0.0/0  tcp to:<pod-IP>:5678
# This is the actual DNAT: ClusterIP:80 -> PodIP:5678

# Step 4: Show the full chain for all services
docker exec net-lab-worker bash -c "iptables -t nat -L KUBE-SERVICES -n | head -30"
```

### IPVS Mode (Alternative)

```bash
# If kube-proxy is running in IPVS mode (check with):
kubectl get configmap -n kube-system kube-proxy -o jsonpath='{.data.config\.conf}' | grep mode
# If mode: ipvs, use ipvsadm instead:

docker exec net-lab-worker ipvsadm -Ln
# Expected:
# IP Virtual Server version 1.2.1
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port     Forward Weight ActiveConn InActConn
# TCP  10.96.3.45:80 rr
#   -> 10.244.1.5:5678        Masq    1      0          0
#   -> 10.244.2.6:5678        Masq    1      0          0
#   -> 10.244.2.7:5678        Masq    1      0          0
# rr = round robin, much cleaner than iptables probabilistic chains
```

### Verify Load Distribution

```bash
# Make 9 requests — should hit all 3 pods
for i in $(seq 1 9); do
  kubectl exec -n kp-lab $(kubectl get pods -n kp-lab -l app=echo -o jsonpath='{.items[0].metadata.name}') \
    -- curl -s http://echo-svc/
done
# Expected: mix of "echo-<hostname>" from different pods
```

### Cleanup

```bash
kubectl delete namespace kp-lab
```

**Key Takeaway:** A ClusterIP is a virtual IP that exists only in iptables/IPVS — no process listens on it. kube-proxy creates DNAT rules: `ClusterIP:port -> random(PodIP:port)`. The probability chains (`statistic mode random probability`) implement equal-cost load balancing. IPVS mode scales better (hash table vs linear iptables scan) for >1000 services.

---

## Summary: Kubernetes Networking Lab Checklist

| Lab  | Core Skill | Production Scenario |
|------|-----------|---------------------|
| 2A   | Packet path tracing across nodes | "Where do I put tcpdump to see this traffic?" |
| 2B   | DNS ndots:5 latency | Unexplained p99 latency spikes on external calls |
| 2C   | NetworkPolicy DNS egress | "DNS works fine, but only inside this pod fails" |
| 2D   | kube-proxy rule inspection | Understanding service mesh bypass and VIP behavior |

## Interview Discussion Points

1. "A pod can't reach an external service but other pods can — what do you check?" — NetworkPolicy egress + DNS egress rules.
2. "We see high DNS query volume to CoreDNS — how do you reduce it?" — ndots:1 + DNS caching at pod level (dnscache sidecar or NodeLocal DNSCache).
3. "How does a ClusterIP service actually work?" — iptables DNAT in KUBE-SVC/KUBE-SEP chains, or IPVS virtual server.
4. "We need to place tcpdump to capture traffic between two pods on different nodes — where?" — On `flannel.1`/`tunl0` for inter-node, or on the specific veth interface for a single pod.
