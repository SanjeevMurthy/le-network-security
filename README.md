# Networking + Security Knowledge Base

Production-grade networking and security guide for **10+ years Senior SRE** interview preparation and day-to-day reference.

Built by synthesizing and extending:
- `le-study-notes/networking` — Application-layer networking (DNS, TLS, HTTP/2-3, LB, Service Mesh)
- `le-study-notes/security` — Cloud-native security (OWASP, ZTA, Container, Supply Chain)
- `le-interview-prep/networking-linux` — Linux kernel internals, debugging, interview Q&A

---

## Learning Paths

### Path 1: Fundamentals-to-Production (8–10 weeks)
Follow sections in order: 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08

### Path 2: Interview Fast-Track (3–4 weeks)
Start with `10-interview-prep/00-study-guide.md`, then drill weak areas in sections 01–09.

### Path 3: On-Call Reference
Jump directly to `07-debugging-playbooks/` for any active incident.

### Path 4: Security Focus
Follow: 05 → 06 → 01 (fundamentals) → 07 (playbooks) → 08 (design)

---

## Section Index

| Section | Topic | Files | Focus |
|---------|-------|-------|-------|
| [01](./01-networking-fundamentals/) | Networking Fundamentals | 8 | OSI, TCP, ARP, BGP, NAT, VXLAN, DNS |
| [02](./02-linux-networking/) | Linux Networking | 8 | Kernel stack, veth, iptables, eBPF, namespaces |
| [03](./03-cloud-networking/) | Cloud Networking | 9 | AWS VPC/EKS, Azure VNet/AKS, Multi-cloud |
| [04](./04-kubernetes-networking/) | Kubernetes Networking | 8 | CNI, services, ingress, policies, Cilium, mesh |
| [05](./05-network-security/) | Network Security | 6 | Firewalls, TLS, DDoS, WAF, VPN, segmentation |
| [06](./06-cloud-security/) | Cloud Security | 8 | ZTA, AWS/Azure sec, containers, supply chain |
| [07](./07-debugging-playbooks/) | Debugging Playbooks | 9 | Hypothesis-driven incident response |
| [08](./08-system-design/) | System Design | 6 | Multi-tier, zero-trust, multi-region, CDN, HA |
| [09](./09-advanced-topics/) | Advanced Topics | 6 | eBPF, mesh internals, observability, BGP, IPv6 |
| [10](./10-interview-prep/) | Interview Prep | 10 | Domain Q&A, cross-domain, patterns, deep-dives |
| [11](./11-hands-on-labs/) | Hands-on Labs | 5 | Break/Fix labs for Linux, K8s, AWS, TLS, eBPF |

**Total: 83 files**

---

## Core Themes

Every section integrates:
1. **Real-world production scenarios** — not toy examples
2. **Failure modes** — how things break and why
3. **Debugging workflows** — step-by-step with commands
4. **Security implications** — attack vectors and mitigations
5. **Trade-off analysis** — design decisions with consequences

---

## Critical Commands Quick Reference

```bash
# TCP connection state
ss -tnp state established
ss -s  # summary

# Packet capture
tcpdump -i eth0 -nn -s0 'port 443' -w /tmp/cap.pcap
tcpdump -r /tmp/cap.pcap -nn 'tcp[tcpflags] & tcp-rst != 0'

# DNS debugging
dig +trace @8.8.8.8 example.com
dig +dnssec example.com
resolvectl query example.com  # systemd-resolved

# TLS/certificate debugging
openssl s_client -connect host:443 -servername host -showcerts
openssl s_client -connect host:443 -tls1_3
openssl x509 -in cert.pem -text -noout | grep -A2 "Validity"

# Linux network state
ip route show table all
ip rule show
conntrack -L | wc -l
nstat -asz | grep -i drop
ethtool -S eth0 | grep -i miss

# Kubernetes networking
kubectl exec -it pod -- curl -v http://service.namespace.svc.cluster.local
kubectl get endpoints
kubectl describe networkpolicy
kubectl exec -n kube-system -it coredns-xxx -- cat /etc/coredns/Corefile

# eBPF/BPF
bpftool prog list
bpftool map list
bpftrace -e 'kprobe:tcp_retransmit_skb { @[comm] = count(); }'
```

---

## Format Conventions

- Mermaid diagrams for all flows (no ASCII art)
- Code blocks for all commands
- Tables for comparisons and trade-offs
- Security callout boxes: `> **Security:** ...`
- Production gotcha callout: `> **Gotcha:** ...`
