# Network & Security Command Cheatsheet

Production-grade command reference for Senior SREs. Extracted and synthesized from the full knowledge base (sections 00–11), covering 51 tools across Linux networking, packet analysis, DNS, TLS, firewalls, eBPF, Kubernetes, and cloud CLIs.

---

## How to Use This Cheatsheet

- **Finding a command**: Use your editor's search (`/tool-name`) or jump via the TOC below
- **Quick copy**: All code blocks are self-contained and runnable
- **Flag tables**: Each tool lists the most important options — not exhaustive, but production-relevant
- **Expected output**: Abbreviated realistic output shown under each example

---

## Table of Contents

### Part 1 — Linux Network Core

| Tool | Purpose |
|------|---------|
| [`ip`](#ip--network-interface-routing-and-policy-management) | Interface, address, route, neigh, rule, netns, tunnel, VRF |
| [`ss`](#ss--socket-statistics) | Active connections, listening ports, TCP internals |
| [`tc`](#tc--traffic-control) | Queueing disciplines, rate limiting, netem, BPF filters |
| [`ethtool`](#ethtool--nic-configuration-and-statistics) | NIC settings, ring buffers, offloads, RSS |
| [`bridge`](#bridge--bridge-management) | FDB, VLAN, STP state, VXLAN VTEPs |
| [`conntrack`](#conntrack--connection-tracking) | NAT table, flow stats, insert failures |
| [`nstat` / `netstat`](#nstat--netstat--network-statistics) | Kernel counters, retransmits, listen overflow |
| [`sysctl`](#sysctl--kernel-network-parameters) | Kernel tuning: buffers, conntrack, routing |
| [`ping` / `ping6`](#ping--ping6--icmp-connectivity-test) | Reachability, PMTU probing |
| [`mtr`](#mtr--combined-traceroute--ping) | Per-hop loss and latency |
| [`traceroute` / `tracepath`](#traceroute--tracepath--path-tracing) | UDP/TCP/ICMP path trace, MTU discovery |
| [`arp` / `arping`](#arp--arping--arp-table-management) | ARP table, gratuitous ARP, duplicate detection |
| [`nc`](#nc--netcat---network-swiss-army-knife) | Port scan, listener, banner grab, file transfer |
| [`socat`](#socat--advanced-socket-relay) | Port forward, TLS probe, UNIX socket relay |
| [`nmap`](#nmap--network-scanner) | Host discovery, port scan, service/TLS enumeration |
| [`iperf3`](#iperf3--bandwidth-testing) | TCP/UDP throughput, parallel streams |
| [`nsenter`](#nsenter--enter-process-namespaces) | Enter container network namespace |
| [`ip netns exec`](#ip-netns-exec--network-namespace-execution) | Execute commands in named namespaces |

### Part 2 — Packet Capture, DNS, HTTP, TLS & Firewall

| Tool | Purpose |
|------|---------|
| [`tcpdump`](#tcpdump--packet-capture-and-analysis) | BPF filter capture, pcap write/read |
| [`tshark`](#tshark--cli-wireshark) | Protocol dissection, field extraction, statistics |
| [`dig`](#dig--dns-lookup) | DNS queries, trace, DNSSEC, reverse lookup |
| [`nslookup`](#nslookup--dns-query-tool) | Interactive DNS queries, debug mode |
| [`host`](#host--simple-dns-lookup) | Quick DNS resolution, all record types |
| [`resolvectl`](#resolvectl--systemd-resolved-query-tool) | systemd-resolved status, flush, monitor |
| [`curl`](#curl--http--https-client) | HTTP testing, TLS debug, timing, mTLS |
| [`wget`](#wget--file-download-tool) | Download, spider, resume, rate-limit |
| [`openssl`](#openssl--tls--certificate-toolkit) | s_client, x509, req, verify, genrsa |
| [`iptables`](#iptables--ipv4-packet-filtering) | Filter, NAT, conntrack states, DNAT/SNAT |
| [`ip6tables`](#ip6tables--ipv6-packet-filtering) | IPv6 filtering, ICMPv6, NDP |
| [`nft`](#nft--nftables---modern-netfilter) | Tables, sets, NAT, atomic ruleset load |
| [`ipset`](#ipset--ip-set-management) | IP/net sets, timed entries, blocklist swap |

### Part 3 — eBPF, Kubernetes, Cloud & Container

| Tool | Purpose |
|------|---------|
| [`bpftool`](#bpftool--bpf-program--map-inspection) | Prog/map list, BTF, CO-RE, net attach |
| [`bpftrace`](#bpftrace--dynamic-tracing) | TCP retransmit, DNS latency, drop reasons |
| [`perf`](#perf--linux-performance-profiling) | CPU perf, network events, flamegraphs |
| [`strace`](#strace--system-call-tracer) | Syscall trace, port exhaustion, BPF debug |
| [`lsof`](#lsof--list-open-filessockets) | Listening sockets, per-process connections |
| [`kubectl`](#kubectl--kubernetes-cli-networking) | Pods, services, endpoints, NetworkPolicy, exec |
| [`istioctl`](#istioctl--istio-service-mesh-cli) | proxy-status, proxy-config, authz, analyze |
| [`cilium`](#cilium--cilium-cni-cli) | Status, monitor, BPF policy/lb/ct, connectivity |
| [`hubble`](#hubble--cilium-observability) | Flow observation, drops, DNS, HTTP flows |
| [`aws`](#aws--aws-cli-networking) | VPC, SG, ELB, Route53, TGW, EKS |
| [`az`](#az--azure-cli-networking) | VNet, NSG, Network Watcher, AKS |
| [`docker`](#docker--container-networking) | Network ls/inspect/create, exec, PID |
| [`crictl`](#crictl--container-runtime-cli) | ps, inspect, exec, logs for containerd/CRI-O |
| [`dmesg`](#dmesg--kernel-ring-buffer) | Conntrack overflow, SYN flood, NIC errors |
| [`journalctl`](#journalctl--systemd-log-query) | Kubelet CNI errors, kernel messages, time range |
| [`systemctl`](#systemctl--service-management) | Network service status, restart, failed units |
| [`vtysh`](#vtysh--frrouting-cli) | BGP summary/neighbors, OSPF, RPKI, BFD |
| [`trivy`](#trivy--container-vulnerability-scanner) | Image, fs, k8s, config scan, secret detection |
| [`vault`](#vault--hashicorp-vault-cli) | PKI, KV, Kubernetes auth, lease management |

---

