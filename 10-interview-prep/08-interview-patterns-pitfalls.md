# Interview Patterns and Pitfalls — What Interviewers Actually Evaluate

## Quick Reference

This section addresses the meta-layer of technical interviews: how to structure answers, what interviewers are actually measuring, how to avoid common failure patterns, and how to demonstrate different levels of seniority. Understanding these patterns is worth as much as technical knowledge — a candidate who knows every iptables command but can't structure a coherent answer under pressure will fail, while a candidate with solid fundamentals and good structure will often pass. Source material: `08_Interview_Patterns_and_Pitfalls.md` (extended). See `10-interview-prep/00-study-guide.md` for the evaluation rubric by level.

---

## Q: An interviewer asks you to explain the OSI model. Most candidates recite all 7 layers and stop. What do Staff engineers say instead?

**What the interviewer is testing:** Whether you use theoretical models as practical debugging frameworks, not memorization exhibits.

**Model Answer:**

The OSI trap is reciting "Physical, Data Link, Network, Transport, Session, Presentation, Application" and stopping. This answers nothing about your ability to use the model in production.

**What Staff engineers say:**

"The OSI model is a conceptual framework that I use as a debugging ladder. In practice, the TCP/IP 4-layer model is what the kernel implements. When I'm troubleshooting a connectivity issue, I start at the bottom because a failure at a lower layer manifests as a failure at every layer above it."

Then immediately ground it in tools:
```text
Layer 1/2 (Physical/Link):
  ethtool -S eth0 | grep -i error    # NIC errors, CRC failures
  ip link show                        # is the link UP?
  "Is the cable plugged in? What speed is negotiated?"

Layer 3 (Network):
  ip route get <destination>          # which route does the kernel use?
  ping <gateway>                      # can I reach the default gateway?
  traceroute <destination>            # where does the path diverge?

Layer 4 (Transport):
  ss -tn state established            # is the TCP handshake completing?
  nstat -a | grep Retrans             # are packets being lost?
  tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'  # are SYNs arriving?

Layer 7 (Application):
  curl -v https://host/health         # what does the app return?
  openssl s_client -connect host:443  # is TLS negotiating?
  dig +trace hostname                 # is DNS resolving correctly?
```

**Where the model breaks down (staff-level awareness):**
- Layers 5 and 6 (Session, Presentation) don't map cleanly to real protocols. TLS spans L4-L6 depending on who you ask.
- NAT operates at L3 and L4 simultaneously (rewrites both IP headers and port numbers)
- MPLS is "between L2 and L3" — the so-called "Layer 2.5"
- eBPF hook points (XDP, TC, cgroup-bpf) don't map to OSI layers at all

**The real mental model for 2025 kernel networking:**
```text
NIC driver (XDP hook) → sk_buff (TC hook) → IP routing (netfilter hooks) → socket → application
```

This is more accurate than OSI for reasoning about performance and eBPF hook placement.

**Follow-up Questions:**
- At which OSI layer does ARP operate and why does it span two layers?
- How does VXLAN encapsulation create a "stack within a stack" in the OSI model?
- If TLS spans layers 4-6, what does tcpdump show vs Wireshark with session keys?

**Connecting Points:**
- `01-networking-fundamentals/` for TCP/IP model implementation
- `10-interview-prep/09-exotic-deep-dive-qa.md` for kernel hook points

---

## Q: An interviewer gives you a live scenario: "A service on port 8080 is not responding. Debug it." Many candidates jump to application logs immediately. What is the systematic 60-second triage?

**What the interviewer is testing:** Efficiency under pressure and bottom-up diagnostic thinking, not application-layer assumptions.

**Model Answer:**

The single biggest failure pattern: spending 10 minutes reading application logs when `ss -tlnp` would show in 5 seconds that the service isn't running.

**The 60-second triage sequence:**

```bash
# 1. Is the process running and listening? (5 seconds)
ss -tlnp | grep 8080
# Nothing → service is not running. Check: systemctl status <service>
# "127.0.0.1:8080" → only accessible locally, not from network

# 2. Can I connect locally? (5 seconds)
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health
# "Connection refused" → process not bound to port
# Timeout → process stuck (deadlock, resource exhaustion, accept queue overflow)
# 4xx/5xx → application-layer issue (proceed to app logs NOW)

# 3. Is a local firewall blocking? (5 seconds)
nft list ruleset | grep 8080        # modern kernels
iptables -L INPUT -n | grep 8080    # older kernels

# 4. Can the client reach the server? (5 seconds, from client side)
nc -zv <server-ip> 8080
# Timeout → network path issue (security group, routing, firewall)
# Connection refused → server-side issue (process or local firewall)

# 5. Is DNS correct? (5 seconds, if connecting by hostname)
dig <hostname> +short
# Verify the IP matches the expected server
```

**After triage — drill down based on finding:**

Finding: `ss` shows LISTEN but remote connection timeouts:
```bash
# Check if accept queue is overflowing
ss -ltn | grep 8080    # Recv-Q > 0 = accept queue overflow → app too slow to accept()
# Fix: increase backlog or fix the application
```

Finding: Connection times out from client but works locally:
```bash
# Check cloud security group / firewall
aws ec2 describe-security-groups --group-ids <sg-id>   # check inbound rules
# Or check nftables rules on the server
nft list ruleset
```

Finding: HTTP 503 (connects but service returns errors):
```bash
# Now read application logs — network is fine, it's an app issue
journalctl -u <service> -n 100
kubectl logs <pod> -n <ns> | tail -100
```

**What interviewers explicitly evaluate:**

1. **Start from the bottom:** Did you check if the service is listening before anything else?
2. **Communicate your reasoning:** Say "I'm checking Layer 4 connectivity first because a transport failure explains all application symptoms."
3. **Distinguish timeout vs refused vs error:** These have completely different causes.
4. **Don't waste time:** Each command in the triage takes < 5 seconds. 60 seconds total should eliminate 80% of possible causes.

**Common interview mistakes:**
- `docker logs` or `kubectl logs` before checking if the process is listening
- Not checking bind address (`127.0.0.1` vs `0.0.0.0`)
- Not testing from the client's perspective ("works on my machine")
- Forgetting to state what each command's output means

**Follow-up Questions:**
- How does this approach change in a Kubernetes environment where you can't SSH to nodes?
- What if the issue is intermittent — how do you debug without active reproduction?
- How do you debug high latency (responds, but slowly) vs not responding at all?

**Connecting Points:**
- `07-debugging-playbooks/` for full debugging playbooks
- `10-interview-prep/02-linux-networking-qa.md` for specific Linux debugging commands

---

## Q: How do you articulate trade-offs at staff level? Give me the framework.

**What the interviewer is testing:** Whether you make defensible decisions with explicit trade-offs, or hedge all answers.

**Model Answer:**

Staff-level trade-off articulation follows a specific structure. "It depends" is an incomplete answer — it must always be followed by: "specifically it depends on X, and here's how each value of X changes the decision."

**The trade-off framework (CART):**

**C — Cost (memory, CPU, money, complexity)**
- Quantify when possible: "nf_conntrack_max 2M costs 640MB kernel memory (non-swappable)"
- Per-unit cost: "each kube-proxy iptables sync at 10K services takes 11 seconds"

**A — Alternatives (what else could you do?)**
- Name at least two alternatives with their cost profiles
- Example: "Instead of increasing conntrack_max, use NOTRACK for stateless traffic (cost: 0 extra memory, but requires auditing all firewall rules for state-dependent matches)"

**R — Risks (what breaks at edge cases or scale)**
- Failure mode: "tcp_tw_recycle was removed from kernel 4.12 because it breaks NAT — packets from multiple clients sharing a NAT IP appear out-of-order timestamp-wise and are silently dropped"
- Scale: "this approach works for 100 nodes but at 1,000 nodes, the O(n^2) BGP mesh means 500,000 sessions — you need route reflectors"

**T — Time (when does this decision expire)**
- "This is the right answer for the next 6 months; at 10x traffic, we'd need to revisit"
- "This was correct in 2022; in 2025, the answer is Cilium because kube-proxy iptables mode is being deprecated"

**Example — applying CART to "Should we use BBR or CUBIC?":**

"CUBIC is the Linux default and has 20 years of production hardening. For our use case (internal LAN traffic between services in the same datacenter, ~1ms RTT, 0% random loss), CUBIC performs near-optimally and there's no gain from BBR. The C: switching to BBR adds negligible overhead. The A: we could also tune CUBIC's cubic_scale or use DCTCP for better behavior in our ECN-enabled data center. The R: BBR v1 is unfair to CUBIC flows in shared environments — if a single BBR flow competes with CUBIC flows, it can consume disproportionate bandwidth. The T: this decision changes if we add a WAN link (100ms RTT, 1% random loss) — BBR would be dramatically better there. For WAN, I'd switch per-service via setsockopt(TCP_CONGESTION)."

**What sounds wrong vs right:**

Wrong: "I'd use BBR, it's faster."
Wrong: "It depends." (no specifics)
Right: "For WAN links with random loss > 0.1%, BBR improves throughput dramatically. For LAN with <0.01% loss, CUBIC is equivalent and has better fairness in shared environments. I'd enable BBR system-wide only for services that primarily communicate over WAN, using the BBR per-socket option for precision."

**Follow-up Questions:**
- How do you present trade-offs when you don't have the data to quantify them?
- What is your decision framework when two options are close and the trade-offs are unclear?
- How do you handle a situation where the technically correct choice conflicts with organizational constraints?

**Connecting Points:**
- `10-interview-prep/00-study-guide.md` for the full evaluation rubric by level
- All sections in `10-interview-prep/` for trade-off examples to memorize

---

## Q: What are the 5 most common failure patterns in networking interviews at the Staff level?

**What the interviewer is testing:** Not applicable — this is metacognitive prep.

**Model Answer:**

Based on interview debrief patterns at Google, Meta, Cloudflare, Stripe, and Datadog:

**Failure Pattern 1: Correct diagnosis, wrong scope**

The candidate correctly identifies the immediate cause but misses second-order effects. Example: "Increase nf_conntrack_max from 262K to 1M." Correct diagnosis, wrong solution — doesn't mention that 1M × 320 bytes = 320MB kernel memory, or that hash table buckets need proportional scaling, or that the real fix is NOTRACK for stateless traffic.

**Prevention:** Always explain WHY your solution works, including what changes in the kernel/system as a result.

**Failure Pattern 2: Tool fetishism without methodology**

The candidate knows many tools but applies them randomly. "I'd run strace, then tcpdump, then look at eBPF traces." Impressive-sounding but shows no systematic approach.

**Prevention:** State your methodology BEFORE using tools: "I start at L4 because a transport failure explains all application symptoms. The first tool is `ss -tlnp` because it answers whether the process is even listening."

**Failure Pattern 3: Memorized answers without understanding**

The candidate gives textbook-perfect answers to memorized questions but fails when the question is slightly reframed. "Walk me through the TCP handshake" → perfect answer. "What happens to the TCP handshake when the server's accept queue is full?" → silence.

**Prevention:** For every topic you study, prepare 3 variants: (1) the textbook question, (2) "what happens when X fails?", (3) "how does this interact with Y (Docker/Kubernetes/AWS)?"

**Failure Pattern 4: No awareness of current state of practice**

Candidates who haven't kept up with the field give outdated answers. "I'd use iptables for firewall management" — correct until 2022, but a Staff SRE in 2026 should also mention nftables (default on RHEL 9, Debian 12) and note that eBPF-based approaches are becoming standard in K8s.

**Prevention:** For each technology, know: (1) what was current 5 years ago, (2) what's current now, (3) what's the direction of travel. Cite specific versions and dates.

**Failure Pattern 5: Hesitation on questions you should know cold**

Hesitating on `ss` vs `netstat`, or not knowing the default ephemeral port range, signals that you've only read about these tools rather than used them. Interviewers expect sub-second recall for basic commands.

**Prevention:** Set up a lab environment and actually run every command in the study guide. Debugging experience that involves real pain (you broke networking in a namespace and had to fix it) creates durable memory that reading cannot.

**Follow-up Questions:**
- How do you recover when you've made a wrong diagnosis in a live troubleshooting interview?
- What do you say when you genuinely don't know the answer?
- How do you demonstrate depth on a topic you don't have direct production experience with?

**Connecting Points:**
- `10-interview-prep/00-study-guide.md` for the 4-week study plan
- `11-hands-on-labs/` for getting real debugging experience

---

## Q: How do you answer "debug this" questions without random commands?

**What the interviewer is testing:** Structured thinking under pressure, not command recall.

**Model Answer:**

The systematic approach: state your mental model, state your hypothesis, pick the fastest test of that hypothesis, interpret the result, update your model.

**The 5-step live debugging protocol:**

**Step 1: Narrate your mental model (10 seconds)**
"A service not responding can fail at: the process level (not running), L4 (TCP not establishing), L3 (routing), firewall (filtered), or application (HTTP error). I'll rule these out bottom-up."

This tells the interviewer your framework before you start typing. It demonstrates structured thinking.

**Step 2: State your hypothesis explicitly**
"My first hypothesis is that the service isn't running or isn't listening on the expected port."

Not: "Let me just run some commands."

**Step 3: Choose the fastest test of that hypothesis**
"The fastest test is `ss -tlnp | grep 8080` — takes 1 second, rules out the most common failure."

Not: "Let me check the application logs." (30+ seconds and doesn't test the hypothesis)

**Step 4: Interpret every output**
"The output shows nothing on 8080. This eliminates all network-layer hypotheses — the problem is at the process level. I'd next check `systemctl status <service>` and `journalctl -u <service> -n 50`."

Not: running the next command without explaining what the current output means.

**Step 5: Pivot explicitly when a hypothesis is wrong**
"My initial hypothesis was wrong — the process is running and listening. I'm updating my hypothesis to: the firewall is blocking external access, since the local `curl localhost:8080` succeeds. Next: `nft list ruleset | grep 8080`."

**What the "simulation" looks like to an interviewer:**

Good candidate: "I see CLOSE_WAIT sockets accumulating. CLOSE_WAIT means the remote peer sent FIN, the kernel ACKed, but the application hasn't called close(). This is definitively an application bug — specifically, a file descriptor leak in the connection pool. The quick proof: `ls /proc/<pid>/fd | wc -l` growing over time. The fix requires application-level changes."

Bad candidate: "Let me try restarting the service... no, that didn't help. Maybe it's a network issue? Let me check iptables... "

**Follow-up Questions:**
- How do you handle a scenario where you've exhausted your hypotheses?
- What do you say when the interviewer gives you a hint?
- How do you balance thoroughness with time pressure in a 45-minute debugging session?

**Connecting Points:**
- `07-debugging-playbooks/` for scenario-specific debugging playbooks
- All files in `10-interview-prep/` for practice scenarios

---

## Q: Live debugging simulation — a pod in Kubernetes can't connect to a service. Walk me through exactly what you'd do at a terminal.

**What the interviewer is testing:** Whether your methodology holds up under realistic conditions with realistic constraints.

**Model Answer:**

```bash
# === STEP 1: Characterize the failure (30 seconds) ===
# What kind of failure is it? Timeout vs refused vs DNS?
kubectl exec -n production <pod> -- \
  curl -v --connect-timeout 5 http://payment-service:8080/health
# "Connection timed out" → routing/policy/firewall issue
# "Connection refused" → service exists but no pod is listening
# "Could not resolve" → DNS issue (start there)
# "502/503" → pods are unhealthy

# === STEP 2: If DNS failure ===
kubectl exec -n production <pod> -- nslookup payment-service
kubectl exec -n production <pod> -- nslookup payment-service.production.svc.cluster.local
# Does the service even exist?
kubectl get svc payment-service -n production
kubectl get endpoints payment-service -n production
# "Endpoints: <none>" = no healthy pods matching the service selector

# === STEP 3: If service exists, verify pod selector matches ===
kubectl get svc payment-service -n production -o yaml | grep selector
kubectl get pods -n production -l app=payment-service  # same labels?

# === STEP 4: If DNS works, test L4 connectivity ===
kubectl exec -n production <pod> -- nc -zv payment-service 8080
# Timeout → NetworkPolicy or kube-proxy issue
# Connection refused → pods exist but nothing listening on 8080

# === STEP 5: Check NetworkPolicy ===
kubectl get networkpolicies -n production
# Is there a policy that could block this connection?
# Check: does the caller pod's labels match the policy's podSelector?
# Check: does the callee pod's labels match the policy's ingress.from.podSelector?

# === STEP 6: Check kube-proxy / iptables ===
# From the source pod's node:
iptables -L -t nat | grep payment-service   # does ClusterIP DNAT rule exist?
kubectl get endpoints -n production payment-service
# If endpoints are empty: kube-proxy hasn't programmed the rule (or pods are unready)

# === STEP 7: Direct pod-to-pod test (bypass service) ===
kubectl get pods -n production -l app=payment-service -o wide  # get a pod IP
kubectl exec -n production <calling-pod> -- curl http://10.244.2.8:8080/health
# If this works: service/kube-proxy issue
# If this fails: pod-level networking or NetworkPolicy issue
```

**Narration during the simulation:**
"I'm starting with DNS because 'could not resolve' is the most common failure and takes 5 seconds to rule out. I see the service exists but has no endpoints — that means no pods are healthy or the selector doesn't match. Let me check the selector... the service selects `app=payment-service` but the pods are labeled `app=payment`. That's the bug — a label mismatch between the service selector and the pod labels."

**Follow-up Questions:**
- What would you check if the pod-to-pod direct test also fails?
- How does this debugging sequence change with Cilium vs kube-proxy?
- What Prometheus metrics would have caught this before a user noticed?

**Connecting Points:**
- `04-kubernetes-networking/` for Kubernetes service mechanics
- `10-interview-prep/04-kubernetes-networking-qa.md` for NetworkPolicy debugging

---

## Key Takeaways

- The OSI trap: reciting 7 layers impresses no one — map each layer to a specific diagnostic command and failure mode
- The 60-second triage is: `ss -tlnp` → `curl localhost` → `iptables -L` → `nc -zv` → `dig` — in that order, bottom-up
- Staff trade-off answers use CART: Cost, Alternatives, Risks, Time horizon — every answer needs all four
- The 5 failure patterns: wrong scope, no methodology, memorized answers without understanding, outdated knowledge, slow on basics — prevention is real lab experience
- "Debug this" questions evaluate methodology first, tool knowledge second — always state your hypothesis before running a command
