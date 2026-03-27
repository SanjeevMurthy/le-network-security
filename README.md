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
| [00](./00-networking-core/) | Networking Core | 4 | Internet, HTTP, TCP/IP, IP addressing (foundational) |
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
| [12](./12-command-cheatsheet/) | Command Cheatsheet | 3 | 51 tools — Linux, K8s, cloud CLI quick reference |

**Total: 91 files**

---

## Command Cheatsheet

51 production-grade commands extracted from the full knowledge base. Each entry includes purpose, key flags, examples with output, and common variants.

| Part | File | Coverage |
|------|------|----------|
| Linux Network Core | [01-linux-network-core.md](./12-command-cheatsheet/01-linux-network-core.md) | `ip`, `ss`, `tc`, `ethtool`, `bridge`, `conntrack`, `nstat`, `sysctl`, `ping`, `mtr`, `traceroute`, `arp`, `nc`, `socat`, `nmap`, `iperf3`, `nsenter` |
| Capture / DNS / HTTP / TLS / Firewall | [02-capture-dns-http-tls-firewall.md](./12-command-cheatsheet/02-capture-dns-http-tls-firewall.md) | `tcpdump`, `tshark`, `dig`, `nslookup`, `host`, `resolvectl`, `curl`, `wget`, `openssl`, `iptables`, `ip6tables`, `nft`, `ipset` |
| eBPF / Kubernetes / Cloud / Container | [03-ebpf-kubernetes-cloud-container.md](./12-command-cheatsheet/03-ebpf-kubernetes-cloud-container.md) | `bpftool`, `bpftrace`, `perf`, `strace`, `lsof`, `kubectl`, `istioctl`, `cilium`, `hubble`, `aws`, `az`, `docker`, `crictl`, `dmesg`, `journalctl`, `systemctl`, `vtysh`, `trivy`, `vault` |

---

## Core Themes

Every section integrates:
1. **Real-world production scenarios** — not toy examples
2. **Failure modes** — how things break and why
3. **Debugging workflows** — step-by-step with commands
4. **Security implications** — attack vectors and mitigations
5. **Trade-off analysis** — design decisions with consequences

---

## Format Conventions

- Mermaid diagrams for all flows (no ASCII art)
- Code blocks for all commands
- Tables for comparisons and trade-offs
- Security callout boxes: `> **Security:** ...`
- Production gotcha callout: `> **Gotcha:** ...`
