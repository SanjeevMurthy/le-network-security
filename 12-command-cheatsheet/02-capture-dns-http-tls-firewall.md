# Network & Security Command Cheatsheet

> Part 2 of 3 — **Packet Capture, DNS, HTTP, TLS & Firewall**
> | [Part 1: Linux Network Core](./01-linux-network-core.md) | [Part 3: eBPF / Kubernetes / Cloud / Container](./03-ebpf-kubernetes-cloud-container.md) |

---

## Table of Contents

| Tool | Purpose |
|------|---------|
| [`tcpdump`](#tcpdump--packet-capture-and-real-time-traffic-analysis) | BPF filter capture, pcap write/read |
| [`tshark`](#tshark--cli-wireshark-for-pcap-analysis-and-field-extraction) | Protocol dissection, field extraction, statistics |
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

---

# Part 2 — Packet Capture, DNS, HTTP, TLS & Firewall

## `tcpdump` — Packet capture and real-time traffic analysis

**When to use:** Capture live traffic or write pcap files for offline analysis. Essential for diagnosing packet drops, retransmissions, SYN floods, malformed packets, and verifying that traffic actually reaches or leaves an interface.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-i <iface>` | Interface to capture on (`any` for all interfaces) |
| `-n` | Do not resolve hostnames |
| `-nn` | Do not resolve hostnames or port names |
| `-v` / `-vv` / `-vvv` | Increase verbosity (IP TTL, checksum, protocol details) |
| `-e` | Print Ethernet (link-layer) headers |
| `-w <file>` | Write raw packets to a pcap file |
| `-r <file>` | Read packets from a pcap file instead of live interface |
| `-s <snaplen>` | Capture only first N bytes per packet (0 = full packet) |
| `-c <count>` | Stop after capturing N packets |
| `-A` | Print packet payload as ASCII |
| `-X` | Print payload in hex and ASCII |
| `-q` | Quiet mode: less protocol information |
| `-tttt` | Print absolute human-readable timestamps |
| `'<filter>'` | BPF filter expression (see below) |

### Common Usage

```bash
# Capture all traffic on eth0, no name resolution
tcpdump -i eth0 -nn

# Capture traffic to/from a host on a specific port, write to file
tcpdump -i eth0 -nn -w /tmp/capture.pcap host 10.0.1.100 and port 5432

# Read a previously saved pcap file
tcpdump -r /tmp/capture.pcap -nn
```

**Output:**
```
14:22:03.112451 IP 10.0.1.50.54231 > 10.0.1.100.5432: Flags [S], seq 1483920211, win 64240, options [mss 1460,sackOK,TS val 3789 ecr 0,nop,wscale 7], length 0
14:22:03.112612 IP 10.0.1.100.5432 > 10.0.1.50.54231: Flags [S.], seq 3012744890, ack 1483920212, win 65160, options [mss 1460,sackOK,TS val 4421 ecr 3789,nop,wscale 7], length 0
14:22:03.112690 IP 10.0.1.50.54231 > 10.0.1.100.5432: Flags [.], ack 1, win 502, length 0
```

### BPF Filter Syntax

```bash
# Filter by host
tcpdump -i eth0 -nn host 192.168.1.1

# Filter by source or destination
tcpdump -i eth0 -nn src 10.0.0.5
tcpdump -i eth0 -nn dst 10.0.0.5

# Filter by port
tcpdump -i eth0 -nn port 443
tcpdump -i eth0 -nn portrange 8000-8080

# Filter by protocol
tcpdump -i eth0 -nn icmp
tcpdump -i eth0 -nn udp port 53

# Combine filters with and / or / not
tcpdump -i eth0 -nn 'host 10.0.1.100 and port 5432'
tcpdump -i eth0 -nn 'not port 22 and not arp'

# TCP flag filters (BPF byte offset syntax)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack|tcp-fin) != 0'
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack = 0'

# ICMP Fragmentation Needed (type 3, code 4) for PMTUD debugging
tcpdump -ni eth0 'icmp and icmp[icmptype] == 3 and icmp[icmpcode] == 4'

# IPv6 NDP Neighbor Solicitation (135) and Advertisement (136)
tcpdump -i eth0 -n 'icmp6 and (ip6[40] == 135 or ip6[40] == 136)'

# IPv6 Router Advertisement (type 134)
tcpdump -i eth0 -n 'icmp6 and ip6[40] == 134' -v

# ARP packets with link-layer headers
tcpdump -e -i eth0 arp -n

# Capture ARP replies only (arp-reply type = 2)
tcpdump -i eth0 'arp and arp[6:2] = 2'

# VXLAN encapsulated traffic
tcpdump -i eth0 -w /tmp/vxlan.pcap udp port 4789

# Flannel VXLAN (some clusters use 8472)
tcpdump -i eth0 -nn udp port 8472
```

### Important Variants

```bash
# Capture SYN packets only, write to pcap (SYN flood evidence)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack = 0' \
  -w /tmp/syn_flood.pcap -c 10000

# Capture with 96-byte snap length (headers only, smaller files)
tcpdump -i eth0 -w /tmp/headers.pcap -s 96 host 10.0.1.100 and port 5432

# Capture inside a container network namespace via nsenter
nsenter -t $CONTAINER_PID -n tcpdump -i eth0 -w /tmp/container.pcap

# DNS query capture (spot ndots search domain amplification)
tcpdump -i any -nn port 53

# Capture on all interfaces, find retransmissions (duplicate seqs)
tcpdump -i eth0 -nn -w /tmp/cap.pcap host 10.0.2.50 and port 443
tcpdump -r /tmp/cap.pcap -nn | grep -E "\[P\.\]|seq" | head -50

# Capture with full timestamps
tcpdump -i eth0 -tttt -nn port 443
```

---

## `tshark` — CLI Wireshark for pcap analysis and field extraction

**When to use:** Parse pcap files with Wireshark dissectors from the command line. Extract specific protocol fields, apply display filters, or convert pcap to JSON/CSV for downstream analysis.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-r <file>` | Read from pcap file |
| `-i <iface>` | Live capture interface |
| `-Y '<filter>'` | Display filter (Wireshark syntax, applied after capture) |
| `-f '<filter>'` | Capture filter (BPF syntax, applied during capture) |
| `-T fields` | Output mode: print specific fields |
| `-T json` | Output as JSON |
| `-T pdml` | Output as XML |
| `-e <field>` | Field to extract (used with `-T fields`) |
| `-E header=y` | Print field header row |
| `-E separator=,` | Set field separator (default tab) |
| `-R '<filter>'` | Read filter (legacy; prefer `-Y`) |
| `-n` | Disable name resolution |
| `-q` | Quiet — suppress packet summary lines |
| `-z <stat>` | Statistics: `io,stat,1`, `conv,tcp`, `endpoints,ip`, etc. |
| `-2` | Two-pass analysis (enables some advanced filters) |

### Common Usage

```bash
# Read pcap and apply a display filter
tshark -r /tmp/capture.pcap -Y 'tcp.flags.syn == 1 and tcp.flags.ack == 0'

# Extract specific fields from SYN packets
tshark -r /tmp/syn.pcap -Y 'tcp.flags.syn == 1' \
  -T fields -e ip.src -e ip.dst -e tcp.srcport -e tcp.dstport \
  -e tcp.options.wscale -E header=y
```

**Output:**
```
ip.src          ip.dst          tcp.srcport  tcp.dstport  tcp.options.wscale
10.0.1.50       10.0.1.100      54231        5432         7
10.0.1.51       10.0.1.100      54232        5432         7
```

### Important Variants

```bash
# Live capture with display filter, output JSON
tshark -i eth0 -Y 'http.response.code >= 500' -T json

# Extract TLS SNI from ClientHello packets
tshark -r /tmp/tls.pcap -Y 'tls.handshake.type == 1' \
  -T fields -e ip.src -e tls.handshake.extensions_server_name

# HTTP request/response summary
tshark -r /tmp/http.pcap -Y 'http' \
  -T fields -e http.request.method -e http.request.uri -e http.response.code

# TCP conversation statistics
tshark -r /tmp/capture.pcap -q -z conv,tcp

# Endpoint statistics
tshark -r /tmp/capture.pcap -q -z endpoints,ip

# IO statistics in 1-second buckets
tshark -r /tmp/capture.pcap -q -z io,stat,1

# Extract DNS queries and responses
tshark -r /tmp/dns.pcap -Y 'dns' \
  -T fields -e dns.qry.name -e dns.resp.addr -e dns.flags.response
```

---

## `dig` — DNS lookup and DNSSEC debugging tool

**When to use:** The primary tool for DNS interrogation. Query any record type, trace the full delegation path, test DNSSEC validation, compare resolvers, and measure lookup times.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `@<server>` | Use this resolver instead of system default |
| `<name>` | DNS name to query |
| `<type>` | Record type: `A`, `AAAA`, `MX`, `NS`, `TXT`, `SOA`, `PTR`, `SRV`, `CAA`, `DNSKEY`, `DS`, `RRSIG` |
| `+short` | One-line answer only |
| `+noall +answer` | Print only the answer section |
| `+trace` | Trace full delegation from root servers |
| `+dnssec` | Request DNSSEC records (sets DO bit) |
| `+cd` | Checking disabled — bypass DNSSEC validation |
| `+tcp` | Force query over TCP |
| `+stats` | Show query time and server stats |
| `+time=<n>` | Set query timeout in seconds |
| `+retry=<n>` | Number of retries |
| `-x <ip>` | Reverse (PTR) lookup |
| `-4` / `-6` | Force IPv4 / IPv6 transport |
| `+multiline` | Pretty-print SOA and DNSKEY records |

### Common Usage

```bash
# Basic A record query
dig example.com A

# Query using a specific resolver
dig @8.8.8.8 example.com A +stats
```

**Output:**
```
; <<>> DiG 9.18.1 <<>> @8.8.8.8 example.com A +stats
;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            300     IN      A       93.184.216.34

;; Query time: 12 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Mar 27 14:00:00 UTC 2026
;; MSG SIZE  rcvd: 56
```

### Important Variants

```bash
# Trace full delegation chain from root
dig example.com A +trace

# Short answer only — great for scripting
dig @8.8.8.8 api.company.com A +short

# Compare public vs private resolver (split-horizon debugging)
dig @8.8.8.8 api.company.com A +short       # public view
dig @10.0.0.2 api.company.com A +short      # internal view

# Query AWS VPC resolver directly
dig @169.254.169.253 api.example.com

# Query CoreDNS directly (Kubernetes)
dig @10.96.0.10 service-name.namespace.svc.cluster.local

# Kubernetes service DNS FQDN
dig +short service-name.namespace.svc.cluster.local

# Reverse lookup
dig -x 93.184.216.34

# SOA record — shows serial, refresh, and negative cache TTL (minimum)
dig example.com SOA +multiline

# Authoritative nameservers
dig NS example.com +short

# Query authoritative NS directly (bypass recursive cache)
dig @ns1.example.com service.example.com +short

# DNSSEC validation check — look for 'ad' (authenticated data) flag
dig +dnssec example.com A

# DNSSEC with checking disabled — diagnose DNSSEC failures
dig +cd example.com A

# Trace DNSSEC chain — find broken or expired RRSIG
dig +dnssec +trace service.example.com

# Check DNSKEY records
dig +dnssec example.com DNSKEY

# Check DS (delegation signer) record in parent zone
dig DS example.com

# Check CAA — restricts which CAs can issue certs
dig example.com CAA

# Force TCP transport (test TCP fallback, large responses)
dig +tcp api.example.com @10.0.0.53

# Measure DNS latency
time dig +short service.namespace.svc.cluster.local

# Check negative TTL in SOA MINIMUM field
dig service.example.com | grep -A5 "AUTHORITY SECTION"

# DNS-over-TLS test via TCP (port 853)
dig @1.1.1.1 -p 853 example.com +tcp
```

---

## `nslookup` — Interactive and non-interactive DNS queries

**When to use:** Quick DNS lookups when `dig` is unavailable (common on Windows and older Linux). Useful for interactive DNS debugging sessions and verifying search domain expansion inside containers.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `<name>` | Name to resolve (positional argument) |
| `<server>` | Optional second argument: resolver to use |
| `-type=<type>` | Record type: `A`, `AAAA`, `MX`, `NS`, `TXT`, `SOA`, `PTR` |
| `-debug` | Show full DNS message including query/response details |
| `-timeout=<n>` | Seconds before retrying |
| `-port=<n>` | Query on non-standard port |

### Common Usage

```bash
# Simple forward lookup
nslookup example.com

# Query using a specific server
nslookup example.com 8.8.8.8
```

**Output:**
```
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name:   example.com
Address: 93.184.216.34
```

### Important Variants

```bash
# Check a specific record type
nslookup -type=MX example.com
nslookup -type=NS example.com
nslookup -type=TXT example.com

# Reverse lookup
nslookup 93.184.216.34

# Debug mode — shows full query/response packets
nslookup -debug example.com

# Verify RDS endpoint resolves after failover
nslookup mydb.cluster-abcdefg.us-east-1.rds.amazonaws.com

# Test from inside a pod (check search domain expansion)
nslookup service-name           # should expand to service-name.namespace.svc.cluster.local
nslookup service-name.namespace.svc.cluster.local  # FQDN, no search expansion
```

---

## `host` — Simple, human-readable DNS lookups

**When to use:** Fast one-liner DNS queries in scripts or when you want clean output without `dig`'s verbosity. Supports all record types and reverse lookups.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-t <type>` | Record type: `A`, `AAAA`, `MX`, `NS`, `TXT`, `SOA`, `PTR`, `ANY` |
| `-v` | Verbose — show full query and response |
| `-a` | Equivalent to `-v -t ANY` |
| `-4` / `-6` | Force IPv4 / IPv6 transport |
| `-W <secs>` | Timeout in seconds |
| `-r` | Disable recursive queries (non-recursive lookup) |

### Common Usage

```bash
# Forward lookup
host example.com
```

**Output:**
```
example.com has address 93.184.216.34
example.com has IPv6 address 2606:2800:220:1:248:1893:25c8:1946
example.com mail is handled by 0 .
```

### Important Variants

```bash
# Reverse lookup
host 93.184.216.34

# MX records
host -t MX example.com

# NS records
host -t NS example.com

# TXT records (SPF, DKIM, DMARC)
host -t TXT example.com
host -t TXT _dmarc.example.com

# SOA record
host -t SOA example.com

# Query a specific resolver
host example.com 8.8.8.8

# Verbose output showing full request/response
host -v example.com
```

---

## `resolvectl` — Query and manage systemd-resolved

**When to use:** On systemd-based Linux hosts, `resolvectl` is the interface to `systemd-resolved`. Use it to check per-interface DNS settings, test resolution, flush the cache, and inspect DNSSEC/LLMNR/mDNS configuration.

### Key Options

| Subcommand | Description |
|------------|-------------|
| `status` | Show DNS servers, search domains, protocols per interface |
| `query <name>` | Perform a DNS lookup through systemd-resolved |
| `query -t <type> <name>` | Query a specific record type |
| `flush-caches` | Flush the DNS cache |
| `statistics` | Show resolver cache hit/miss statistics |
| `reset-statistics` | Reset resolver statistics counters |
| `dns [iface] [servers]` | Show or set DNS servers for an interface |
| `domain [iface] [domains]` | Show or set search domains for an interface |
| `log-level <level>` | Set logging verbosity (`debug`, `info`, `warning`) |
| `monitor` | Live stream of all DNS queries and responses |

### Common Usage

```bash
# Show full resolver status: DNS servers, protocols, search domains
resolvectl status
```

**Output:**
```
Global
       LLMNR setting: no
MulticastDNS setting: no
  DNSOverTLS setting: no
      DNSSEC setting: allow-downgrade
    DNSSEC supported: yes
          DNS Servers: 10.0.0.53
           DNS Domain: ~.

Link 2 (eth0)
      Current Scopes: DNS
       LLMNR setting: yes
          DNS Servers: 10.0.0.53
           DNS Domain: us-east-1.compute.internal
```

### Important Variants

```bash
# Perform a DNS query through systemd-resolved
resolvectl query example.com
resolvectl query -t MX example.com
resolvectl query -t AAAA example.com

# Flush DNS cache after record change or propagation wait
resolvectl flush-caches
# (legacy alias: systemd-resolve --flush-caches)

# Show resolver cache statistics
resolvectl statistics

# Monitor all DNS queries in real time (debugging ndots, search domain amplification)
resolvectl monitor

# Check DNSSEC status for a domain
resolvectl query --legend=yes example.com

# Set a specific DNS server for eth0 (temporary, survives until NetworkManager resets)
resolvectl dns eth0 1.1.1.1 8.8.8.8

# Set search domain for eth0
resolvectl domain eth0 example.internal corp.local
```

---

## `curl` — HTTP/HTTPS client for testing, debugging, and automation

**When to use:** The go-to tool for HTTP/HTTPS connectivity testing, TLS debugging, header inspection, API calls, authentication flows, and measuring per-phase request latency. Covers HTTP/1.1, HTTP/2, and HTTP/3.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-v` | Verbose: show TLS handshake, request headers, response headers |
| `-s` | Silent: suppress progress meter and error messages |
| `-S` | With `-s`, still show errors |
| `-o <file>` | Write response body to file (`/dev/null` to discard) |
| `-O` | Write to file named after remote file |
| `-I` | HEAD request — show response headers only |
| `-i` | Include response headers in stdout output |
| `-L` | Follow HTTP redirects |
| `-X <method>` | HTTP method: `GET`, `POST`, `PUT`, `DELETE`, `PATCH` |
| `-d '<data>'` | POST body (string) |
| `--data-binary @file` | POST body from file (preserves newlines) |
| `-H '<header>'` | Add or override a request header |
| `-u user:pass` | HTTP Basic authentication |
| `-b '<cookies>'` | Send cookies |
| `-c <file>` | Save response cookies to file |
| `--cookie-jar <file>` | Alias for `-c` |
| `-k` | Skip TLS certificate verification (insecure) |
| `--cacert <file>` | Use custom CA bundle for TLS verification |
| `--cert <file>` | Client certificate for mTLS |
| `--key <file>` | Client private key for mTLS |
| `--tls-max <ver>` | Maximum TLS version: `1.0`, `1.1`, `1.2`, `1.3` |
| `--tlsv1.2` | Require at least TLS 1.2 |
| `--tlsv1.3` | Require TLS 1.3 only |
| `--http2` | Force HTTP/2 (ALPN negotiation) |
| `--http3` | Force HTTP/3 over QUIC |
| `--resolve <host:port:ip>` | Override DNS for a specific host:port |
| `--connect-timeout <n>` | TCP connection timeout in seconds |
| `--max-time <n>` | Total request timeout in seconds |
| `-w '<format>'` | Write-out: print timing/status variables after response |
| `-A '<agent>'` | Set User-Agent header |
| `--compressed` | Request gzip/deflate compression |
| `-4` / `-6` | Force IPv4 / IPv6 |
| `--noproxy '*'` | Bypass all proxies |
| `--proxy <url>` | Use an HTTP/SOCKS proxy |

### Common Usage

```bash
# Verbose HTTPS request — shows TLS handshake, all headers
curl -v https://example.com

# Check HTTP status code only (silent, discard body)
curl -s -o /dev/null -w "%{http_code}\n" https://example.com
```

**Output:**
```
* Trying 93.184.216.34:443...
* Connected to example.com (93.184.216.34) port 443
* ALPN: offering h2,http/1.1
* TLS 1.3, TLS handshake, Client hello (1):
* TLS 1.3, TLS handshake, Server hello (2):
* TLS 1.3, TLS handshake, Encrypted Extensions (8):
* TLS 1.3, TLS handshake, Certificate (11):
* TLS 1.3, TLS handshake, CERT Verify (15):
* TLS 1.3, TLS handshake, Finished (20):
> GET / HTTP/2
> Host: example.com
...
< HTTP/2 200
```

### Important Variants

```bash
# HEAD request — check status and headers without downloading body
curl -I https://example.com

# Check public IP of current host
curl -s ifconfig.me

# Per-phase latency breakdown (DNS / TCP / TLS / TTFB / total)
curl -s -o /dev/null \
  -w "dns_lookup:     %{time_namelookup}s\ntcp_connect:    %{time_connect}s\ntls_handshake:  %{time_appconnect}s\nttfb:           %{time_starttransfer}s\ntotal:          %{time_total}s\n" \
  https://service:443/health

# Compact one-liner timing (run 10x to get distribution)
curl -w "dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" \
  -o /dev/null -s https://service:443/health

# Timed connectivity test with connection and total timeouts
curl -v --connect-timeout 5 --max-time 10 http://service:8080/health

# Force HTTP/2 and show ALPN negotiation
curl --http2 -v https://example.com

# Test with custom Host header (K8s ingress routing)
curl -H "Host: hostname.example.com" http://10.0.1.5/path

# Override DNS — test a specific IP without changing /etc/hosts
curl -v https://api.example.com/health --resolve api.example.com:443:54.1.2.3

# POST with JSON body
curl -X POST https://api.example.com/v1/resource \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"key": "value"}'

# POST with file body
curl -X POST https://api.example.com/upload \
  -H "Content-Type: application/octet-stream" \
  --data-binary @/path/to/file

# mTLS client certificate authentication
curl --cert client.crt --key client.key --cacert ca.crt https://service:443/

# Follow redirects (up to 30 by default)
curl -L https://example.com

# Check HTTP security headers on API endpoint
curl -I https://api.example.com/v1/health | \
  grep -E "Strict-Transport|X-Content-Type|X-Frame|Content-Security|Access-Control"

# Extract TLS error details from verbose output
curl -v https://service:443/health 2>&1 | grep -E "SSL|error|certificate|expired|verify"

# Bypass TLS verification (testing self-signed certs; never in production)
curl -sk https://internal-service:8443/health

# Test with custom CA bundle
curl --cacert /etc/ssl/custom/ca-bundle.crt https://internal-service/

# DNS-over-HTTPS (DoH) query via Cloudflare JSON API
curl -H "accept: application/dns-json" \
  "https://cloudflare-dns.com/dns-query?name=example.com&type=A"

# Check certificate transparency logs for a domain
curl -s "https://crt.sh/?q=api.example.com&output=json" | \
  jq '.[] | {cn: .common_name, issuer: .issuer_name, expiry: .not_after}'

# Test internet connectivity through NAT Gateway
curl -s --connect-timeout 5 http://checkip.amazonaws.com

# Test EC2 IMDS (should be blocked from workload subnets)
curl -s --connect-timeout 3 http://169.254.169.254/latest/meta-data/

# Envoy admin API — dump full config
curl -s localhost:15000/config_dump | jq .

# Envoy cluster live stats
curl -s localhost:15000/clusters | grep "backend::" | \
  grep -E "(healthy|cx_active|rq_active|rq_timeout)"
```

---

## `wget` — Non-interactive file downloader and HTTP client

**When to use:** Download files from HTTP/FTP servers in scripts. Supports recursive downloads, resuming interrupted transfers, mirroring, and authenticated downloads. Useful when `curl` is unavailable or when you need built-in retry logic.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-O <file>` | Save output to a named file (`-` for stdout) |
| `-o <logfile>` | Log messages to a file instead of stderr |
| `-q` | Quiet: suppress output except errors |
| `-v` | Verbose output |
| `-c` | Continue/resume a partially downloaded file |
| `--tries=<n>` | Number of retries (0 = unlimited) |
| `--timeout=<n>` | Timeout for connect, DNS, and read in seconds |
| `--connect-timeout=<n>` | Timeout for connection establishment only |
| `--read-timeout=<n>` | Timeout for data read stalls |
| `-r` | Recursive download |
| `-l <n>` | Maximum recursion depth |
| `-np` | Do not ascend to parent directory during recursion |
| `-p` | Download all page dependencies (images, CSS, JS) |
| `--no-check-certificate` | Skip TLS verification |
| `--ca-certificate=<file>` | Custom CA bundle |
| `--certificate=<file>` | Client certificate for mTLS |
| `--private-key=<file>` | Client private key for mTLS |
| `--header='<H>'` | Add a request header |
| `--post-data='<data>'` | Send POST request with data |
| `--post-file=<file>` | Send POST request with file body |
| `--user=<u>` | HTTP Basic auth username |
| `--password=<p>` | HTTP Basic auth password |
| `--spider` | Do not download — just check URL exists (HTTP 200 vs error) |
| `-S` | Print server response headers |
| `--limit-rate=<rate>` | Throttle download speed (e.g., `1m` = 1 MB/s) |
| `-b` | Run in background |
| `-i <file>` | Read URLs from a file |
| `--mirror` | Enable mirroring (`-r -N -l inf --no-remove-listing`) |

### Common Usage

```bash
# Download a file to current directory
wget https://example.com/file.tar.gz

# Download to stdout (pipe to another command)
wget -qO- https://example.com/script.sh | bash
```

**Output:**
```
--2026-03-27 14:00:00--  https://example.com/file.tar.gz
Resolving example.com... 93.184.216.34
Connecting to example.com|93.184.216.34|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5242880 (5.0M) [application/gzip]
Saving to: 'file.tar.gz'
file.tar.gz     100%[===================>]   5.00M  2.35MB/s    in 2.1s
```

### Important Variants

```bash
# Resume a partial download
wget -c https://example.com/large-file.iso

# Check if a URL is reachable without downloading (spider mode)
wget --spider https://service:8080/health

# Show response headers
wget -S -q -O /dev/null https://example.com

# Download with 3 retries and 10 second timeout
wget --tries=3 --timeout=10 https://example.com/file.tar.gz

# Download with Authorization header
wget --header="Authorization: Bearer $TOKEN" https://api.example.com/export

# POST request
wget --post-data='{"key":"value"}' --header="Content-Type: application/json" \
  -O - https://api.example.com/v1/resource

# Download and extract in one pipeline
wget -qO- https://releases.example.com/app-v1.2.tar.gz | tar xz

# Recursive download with depth limit (mirror a doc site)
wget -r -l 2 -np -p https://docs.example.com/

# Throttle download to avoid saturating links
wget --limit-rate=5m https://example.com/bigfile.bin

# Skip TLS verification (insecure — testing only)
wget --no-check-certificate https://internal-dev.example.com/
```

---

## `openssl` — TLS debugging, certificate inspection, and PKI toolkit

**When to use:** The essential Swiss Army knife for anything TLS-related. Use `s_client` to test TLS connections, debug handshake failures, inspect certificates, and verify mTLS. Use `x509`, `req`, `verify`, and `genrsa` for PKI operations and certificate lifecycle management.

### Key Options — `s_client`

| Flag/Option | Description |
|-------------|-------------|
| `-connect <host:port>` | Target server |
| `-servername <name>` | SNI hostname (required if IP ≠ hostname) |
| `-tls1_2` / `-tls1_3` | Force a specific TLS version |
| `-tls1_1` | Force TLS 1.1 (use to confirm it is rejected) |
| `-showcerts` | Show full certificate chain, not just leaf |
| `-cert <file>` | Client certificate for mTLS |
| `-key <file>` | Client private key for mTLS |
| `-CAfile <file>` | CA bundle for server certificate verification |
| `-verify <depth>` | Enable certificate verification with chain depth |
| `-alpn <proto>` | Offer ALPN protocol (e.g., `h2`, `http/1.1`) |
| `-starttls <proto>` | STARTTLS upgrade: `smtp`, `pop3`, `imap`, `ftp` |
| `-msg` | Show all protocol messages |
| `-debug` | Full hex dump of protocol messages |
| `-state` | Print SSL state machine transitions |
| `-status` | Request OCSP stapling response |

### Common Usage

```bash
# Connect to HTTPS server and inspect TLS handshake
openssl s_client -connect example.com:443 -servername example.com </dev/null
```

**Output:**
```
CONNECTED(00000003)
depth=2 C=US, O=DigiCert Inc, CN=DigiCert Global Root CA
verify return:1
depth=1 C=US, O=DigiCert Inc, CN=DigiCert TLS RSA SHA256 2020 CA1
verify return:1
depth=0 CN=www.example.com
verify return:1
---
Certificate chain
 0 s:CN=www.example.com
   i:C=US, O=DigiCert Inc, CN=DigiCert TLS RSA SHA256 2020 CA1
...
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: ...
---
```

### Important Variants

```bash
# Test TLS 1.3 specifically
openssl s_client -connect example.com:443 -tls1_3 -servername example.com </dev/null

# Test with SNI on an ingress IP
openssl s_client -connect 10.0.1.5:443 -servername hostname.example.com </dev/null

# Get full cert chain (check for missing intermediates)
openssl s_client -connect host:443 -servername host -showcerts </dev/null 2>/dev/null

# Extract cert from live server and inspect it
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null \
  | openssl x509 -noout -text

# Check expiry dates from live server
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null \
  | openssl x509 -noout -dates

# Script-friendly expiry check (exits 0 = valid, 1 = expired)
echo | openssl s_client -connect host:443 -servername host 2>/dev/null \
  | openssl x509 -noout -checkend 0 && echo "cert valid" || echo "cert EXPIRED"

# Check if cert expires within 24 hours
openssl x509 -noout -checkend 86400 -in cert.pem && echo "OK" || echo "expires within 24h"

# List SANs from live server (hostname mismatch debugging)
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName

# mTLS — present client cert
openssl s_client -connect host:443 -cert client.crt -key client.key \
  -servername host </dev/null

# PCI/TLS version compliance: confirm TLS 1.1 is rejected
openssl s_client -connect payment-api.example.com:443 -tls1_1 2>&1 | \
  grep -E "handshake failure|alert|no protocols"

# DNS-over-TLS (DoT) — port 853
openssl s_client -connect 1.1.1.1:853 -servername 1.1.1.1

# SMTP with STARTTLS
openssl s_client -connect smtp.example.com:587 -starttls smtp

# x509: inspect a certificate file
openssl x509 -noout -text -in cert.pem

# x509: quick summary — subject, issuer, dates, SANs
openssl x509 -noout -subject -issuer -dates -ext subjectAltName -in cert.pem

# x509: verify expiry of a local cert file
openssl x509 -noout -dates -in cert.pem

# x509: get fingerprint (SHA-256)
openssl x509 -noout -fingerprint -sha256 -in cert.pem

# verify: check cert chain against system CA bundle
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt cert-chain.pem

# verify: check cert chain with explicit CA and intermediate
openssl verify -CAfile root-ca.pem -untrusted intermediate.pem leaf.pem

# verify: verify client cert is signed by server's trusted CA
openssl verify -CAfile server-trusted-ca.pem client.crt

# genrsa: generate a 4096-bit RSA private key
openssl genrsa -out server.key 4096

# req: generate a CSR with SANs inline
openssl req -new -key server.key -out server.csr \
  -subj "/CN=api.example.com/O=Example Inc/C=US" \
  -addext "subjectAltName=DNS:api.example.com,DNS:api-v2.example.com"

# req: generate a self-signed cert for testing
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 \
  -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1" -nodes

# speed: benchmark TLS performance
openssl speed rsa2048 ecdh

# Check if private key matches certificate
openssl x509 -noout -modulus -in cert.pem | md5sum
openssl rsa  -noout -modulus -in key.pem  | md5sum
# (outputs must match)
```

---

## `iptables` — IPv4 packet filtering, NAT, and packet mangling

**When to use:** Inspect, modify, and debug the Linux kernel's IPv4 netfilter rules. Use it to allow/deny traffic, implement NAT, mark packets for policy routing, clamp TCP MSS, trace packet paths, and review kube-proxy rules in Kubernetes clusters.

### Key Options

| Flag/Option | Description |
|-------------|-------------|
| `-L [chain]` | List rules in chain (default: all chains in filter table) |
| `-n` | No hostname/service resolution (faster listing) |
| `-v` | Verbose: show interface, packet/byte counters |
| `--line-numbers` | Show rule line numbers |
| `-t <table>` | Target table: `filter` (default), `nat`, `mangle`, `raw` |
| `-A <chain>` | Append rule to chain |
| `-I <chain> [n]` | Insert rule at position N (default: 1 = top) |
| `-D <chain> [n]` | Delete rule by line number or by specification |
| `-R <chain> <n>` | Replace rule at line number N |
| `-F [chain]` | Flush (delete all rules in) chain |
| `-Z [chain]` | Zero packet/byte counters |
| `-P <chain> <target>` | Set chain default policy: `ACCEPT`, `DROP` |
| `-N <chain>` | Create a new user-defined chain |
| `-p <proto>` | Protocol: `tcp`, `udp`, `icmp`, `all` |
| `-s <cidr>` | Source address/network |
| `-d <cidr>` | Destination address/network |
| `--dport <port>` | Destination port (requires `-p tcp` or `-p udp`) |
| `--sport <port>` | Source port |
| `-i <iface>` | Input interface (PREROUTING, INPUT, FORWARD) |
| `-o <iface>` | Output interface (FORWARD, OUTPUT, POSTROUTING) |
| `-j <target>` | Jump to target: `ACCEPT`, `DROP`, `REJECT`, `LOG`, `DNAT`, `SNAT`, `MASQUERADE`, `MARK`, `RETURN`, `TRACE`, `NOTRACK` |
| `-m <module>` | Load match extension: `conntrack`, `state`, `set`, `limit`, `multiport`, `comment` |

### Common Usage

```bash
# List INPUT chain rules with packet counts and line numbers
iptables -L INPUT -n -v --line-numbers
```

**Output:**
```
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1     1823  145K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:443
2        0     0 DROP       tcp  --  *      *       10.10.10.0/24        0.0.0.0/0
```

### Important Variants

```bash
# --- Listing rules ---

# All filter table chains with counters
iptables -L -n -v

# Specific table listing
iptables -t nat -L -n -v
iptables -t mangle -L -n -v
iptables -t raw -L -n -v

# Show only active DROP/REJECT rules (non-zero counters)
iptables -L -n -v | grep -E "(DROP|REJECT)"
iptables -L -n -v | grep -E "DROP|REJECT" | awk '$1 != "0" || $2 != "0"'

# Dump all rules in restorable format
iptables-save

# Count total rules (high counts cause O(n) lookup latency)
iptables-save | wc -l

# --- Accepting and blocking ---

# Allow established and related connections (stateful)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Accept traffic on specific port
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Drop traffic from a CIDR
iptables -I INPUT -s 198.51.100.0/24 -j DROP

# Reject with ICMP port-unreachable (client gets immediate error)
iptables -A INPUT -p tcp --dport 8080 -j REJECT --reject-with icmp-port-unreachable

# Insert ACCEPT rule before an existing DROP (use line numbers)
iptables -I INPUT 2 -p tcp --dport 8443 -s 10.0.0.0/8 -j ACCEPT

# Delete a rule by line number
iptables -D INPUT 3

# Zero counters, then monitor which rules fire
iptables -Z INPUT
iptables -L INPUT -n -v --line-numbers | awk '$1 > 0 {print}'

# --- NAT ---

# SNAT: translate private source to a specific public IP
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j SNAT --to-source 1.2.3.4

# SNAT across a pool of IPs (expands port capacity)
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 10.0.1.1-10.0.1.10

# MASQUERADE: dynamic SNAT using interface IP (for DHCP/dynamic public IPs)
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# DNAT: redirect inbound traffic to internal host:port
iptables -t nat -A PREROUTING -d 1.2.3.4 -p tcp --dport 80 \
  -j DNAT --to-destination 10.0.0.10:8080

# List Docker NAT rules
iptables -t nat -L DOCKER -n -v

# List all POSTROUTING NAT rules
iptables -t nat -L POSTROUTING -n -v

# --- Packet marking for policy routing ---

# Mark packets destined for a network
iptables -t mangle -A OUTPUT -d 10.8.0.0/8 -j MARK --set-mark 0x1

# Mark monitoring traffic by destination port
iptables -t mangle -A OUTPUT -p tcp --dport 9090 -j MARK --set-mark 0x10

# --- MSS clamping (VXLAN/VPN overlay networks) ---

# Clamp MSS to PMTU (dynamic, recommended)
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# Set fixed MSS for VXLAN overlay (1450 MTU - 54B headers = 1396B MSS)
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN SYN \
  -j TCPMSS --set-mss 1396

# --- ICMP / PMTUD ---

# Allow ICMP Fragmentation Needed (required for PMTUD to work)
iptables -I INPUT  -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -I FORWARD -p icmp --icmp-type fragmentation-needed -j ACCEPT

# Remove a rule blocking ICMP (fix MTU black holes)
iptables -D INPUT -p icmp -j DROP

# --- Debugging and tracing ---

# Add a LOG rule at top of INPUT chain
iptables -I INPUT 1 -j LOG --log-prefix "IPT-DEBUG: " --log-level 4 --log-uid

# View logged packets from the kernel journal
# journalctl -k | grep "IPT-DEBUG"

# Enable TRACE for a specific flow (follow through all chains)
iptables -t raw -I PREROUTING -s 10.0.0.5 -p tcp --dport 80 -j TRACE
iptables -t raw -I PREROUTING -p tcp -d 10.96.100.50 --dport 443 -j TRACE

# Exempt high-frequency traffic from conntrack (health checks)
iptables -t raw -A PREROUTING -p tcp --dport 8080 -j NOTRACK
iptables -t raw -A OUTPUT     -p tcp --sport 8080 -j NOTRACK

# --- SYN flood mitigation ---

# Rate-limit SYN packets (100/s, burst 200)
iptables -I INPUT -p tcp --syn -m limit --limit 100/s --limit-burst 200 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# --- Kubernetes kube-proxy rules ---

# List KUBE-SERVICES NAT chain
iptables -t nat -L KUBE-SERVICES -n

# Find rule for a specific ClusterIP
iptables -t nat -L KUBE-SERVICES -n | grep 10.96.50.100

# Follow kube-proxy chain to endpoints
iptables -t nat -L KUBE-SVC-XXXXXXXXXXXXXXXX -n -v
iptables -t nat -L KUBE-SEP-XXXXXXXXXXXXXXXX -n -v

# --- Migration to nftables ---

# Translate a single iptables rule to nftables syntax
iptables-translate -A INPUT -p tcp --dport 80 -j ACCEPT

# Migrate full ruleset to nftables
iptables-save | iptables-restore-translate -f - | nft -f -
```

---

## `ip6tables` — IPv6 packet filtering (iptables API for IPv6)

**When to use:** Manage IPv6 firewall rules. Syntax is identical to `iptables` but operates on IPv6 (inet6) traffic. On modern systems, prefer `nft` which handles both IPv4 and IPv6 in a single dual-stack table.

### Key Options

Same flags as `iptables`. Key IPv6-specific differences:

| Difference | Notes |
|------------|-------|
| Protocol `-p ipv6-icmp` | ICMPv6 protocol name (not `icmp`) |
| `--icmpv6-type` | ICMPv6 type names (vs `--icmp-type`) |
| No MASQUERADE + dynamic address | Use SNAT with explicit IPv6 address |
| No fragmentation | IPv6 fragments handled by end-hosts only |

### Common Usage

```bash
# List IPv6 INPUT rules
ip6tables -L INPUT -n -v --line-numbers

# Allow established/related IPv6 traffic
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Important Variants

```bash
# Allow ICMPv6 Packet Too Big (mandatory for IPv6 PMTUD)
ip6tables -A INPUT  -p ipv6-icmp --icmpv6-type packet-too-big -j ACCEPT
ip6tables -A FORWARD -p ipv6-icmp --icmpv6-type packet-too-big -j ACCEPT

# Allow NDP (Neighbor Solicitation and Advertisement — equivalent to ARP in IPv4)
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type neighbor-solicitation -j ACCEPT
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type neighbor-advertisement -j ACCEPT

# Allow Router Advertisement (required for SLAAC)
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type router-advertisement -j ACCEPT

# Block all IPv6 traffic from a prefix
ip6tables -A INPUT -s 2001:db8:bad::/48 -j DROP

# Allow SSH over IPv6
ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT

# Flush all IPv6 rules in filter table
ip6tables -F

# Save and restore IPv6 rules
ip6tables-save > /etc/iptables/rules.v6
ip6tables-restore < /etc/iptables/rules.v6

# Translate to nftables
ip6tables-translate -A INPUT -p tcp --dport 443 -j ACCEPT
```

---

## `nft` / `nftables` — Modern dual-stack netfilter framework

**When to use:** The successor to `iptables`/`ip6tables`/`arptables`/`ebtables`. Handles IPv4 and IPv6 in a single table. Supports sets, maps, and verdict maps natively for O(1) lookups. Atomic ruleset loading eliminates rule-update races. Required on modern Linux distributions where `iptables` is a compatibility shim over nft.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **table** | Top-level container; family: `ip`, `ip6`, `inet` (dual-stack), `arp`, `bridge` |
| **chain** | Contains rules; types: `filter`, `nat`, `route`; hooks: `prerouting`, `input`, `forward`, `output`, `postrouting` |
| **rule** | Match + verdict: `accept`, `drop`, `reject`, `return`, `jump <chain>` |
| **set** | Named collection of IPs, ports, or other values; used in match expressions |
| **map** | Key-to-value lookup (e.g., IP to verdict) |

### Common Usage

```bash
# List full ruleset
nft list ruleset

# Load ruleset atomically from file
nft -f /etc/nftables/ruleset.nft
```

**Output (`nft list ruleset`):**
```
table inet myfilter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        tcp dport { 22, 443, 8443 } accept
        drop
    }
}
```

### Important Variants

```bash
# --- Table and chain management ---

# Create a dual-stack filter table
nft add table inet myfilter

# Create an input chain with default-drop policy
nft add chain inet myfilter input \
  '{ type filter hook input priority 0; policy drop; }'

# Create a forward chain
nft add chain inet myfilter forward \
  '{ type filter hook forward priority 0; policy accept; }'

# Create a NAT table and postrouting chain
nft add table ip nat
nft add chain ip nat postrouting \
  '{ type nat hook postrouting priority srcnat; }'

# --- Rules ---

# Allow established and related
nft add rule inet myfilter input ct state established,related accept

# Accept ports using inline set (no O(n) rule scan)
nft add rule inet myfilter input tcp dport { 22, 443, 8443 } accept

# Drop traffic from a source IP
nft add rule inet myfilter input ip saddr 198.51.100.0/24 drop

# Reject with TCP reset
nft add rule inet myfilter input tcp dport 8080 reject with tcp reset

# Log dropped packets (with log prefix)
nft add rule inet myfilter input log prefix "nft-drop: " drop

# MASQUERADE for NAT gateway
nft add rule ip nat postrouting oifname "eth0" masquerade

# DNAT — redirect port 80 to internal host
nft add rule ip nat prerouting ip daddr 1.2.3.4 tcp dport 80 \
  dnat to 10.0.0.10:8080

# Notrack — exempt health checks from conntrack
nft add rule ip raw prerouting tcp dport 8080 notrack

# --- Sets ---

# Create a named set of IP addresses
nft add set inet myfilter blocked-ips { type ipv4_addr; }
nft add element inet myfilter blocked-ips { 198.51.100.1, 198.51.100.2 }

# Create a set with CIDR ranges
nft add set inet myfilter trusted-nets { type ipv4_addr; flags interval; }
nft add element inet myfilter trusted-nets { 10.0.0.0/8, 172.16.0.0/12 }

# Use a set in a rule
nft add rule inet myfilter input ip saddr @blocked-ips drop
nft add rule inet myfilter input ip saddr @trusted-nets tcp dport 8443 accept

# --- Inspection and debugging ---

# List only a specific table
nft list table inet myfilter

# Check rules matching a port
nft list ruleset | grep -B5 -A2 "8080"

# Find drop/reject rules during incident
nft list ruleset | grep -B5 "drop\|reject"

# Enable nftables tracing for a specific flow
# Step 1: add trace rule
nft add rule inet myfilter input ip saddr 10.0.0.5 tcp dport 80 meta nftrace set 1
# Step 2: monitor trace events
nft monitor trace

# --- Atomic ruleset management (safest approach) ---

# Verify syntax without applying
nft -c -f /etc/nftables/ruleset.nft

# Apply atomically (entire ruleset swapped as one transaction)
nft -f /etc/nftables/ruleset.nft

# Complete ruleset file example saved for reference:
# table inet filter {
#     chain input {
#         type filter hook input priority 0; policy drop;
#         ct state vmap { established : accept, related : accept, invalid : drop }
#         tcp dport { 22, 443 } accept
#         icmpv6 type { nd-neighbor-solicit, nd-neighbor-advert, mld-listener-query } accept
#         log prefix "INPUT-DROP: "
#     }
# }
```

---

## `ipset` — High-performance IP set management for iptables/nftables

**When to use:** When you have more than a handful of IP addresses, CIDRs, or port ranges to match in iptables rules. `ipset` uses hash tables or bitmaps for O(1) lookups instead of the O(n) linear scan of individual iptables rules. Essential for blocklists, allowlists, and rate-limiting at scale.

### Key Options

| Subcommand | Description |
|------------|-------------|
| `create <name> <type>` | Create a new set |
| `add <name> <entry>` | Add an entry to a set |
| `del <name> <entry>` | Remove an entry from a set |
| `test <name> <entry>` | Test if entry is in set (exit 0 = found) |
| `list [name]` | List set contents |
| `flush [name]` | Remove all entries from set (or all sets) |
| `destroy [name]` | Delete set and free memory |
| `save [name]` | Print set in restorable format |
| `restore` | Restore sets from `save` output |
| `swap <s1> <s2>` | Atomically swap two sets of the same type |
| `-exist` | Ignore errors if entry already exists |
| `--timeout <secs>` | Entry lifetime (0 = permanent) |

### Set Types

| Type | Use case |
|------|----------|
| `hash:ip` | Single IPv4/IPv6 addresses |
| `hash:net` | CIDR prefixes |
| `hash:ip,port` | IP + port combinations |
| `hash:net,port` | CIDR + port combinations |
| `hash:ip,port,ip` | IP + port + IP (NAT mapping) |
| `bitmap:port` | Fast port range bitmaps (0-65535) |
| `list:set` | Set of sets (meta-set) |

### Common Usage

```bash
# Create a set and add entries
ipset create blocked-ips hash:ip
ipset add blocked-ips 198.51.100.1
ipset add blocked-ips 203.0.113.50

# Use the set in iptables
iptables -I INPUT -m set --match-set blocked-ips src -j DROP
```

**Output (`ipset list blocked-ips`):**
```
Name: blocked-ips
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 296
References: 1
Number of entries: 2
Members:
198.51.100.1
203.0.113.50
```

### Important Variants

```bash
# --- Creating sets ---

# Hash:net set for CIDR matching (O(1) for thousands of subnets)
ipset create trusted-sources hash:net
ipset add trusted-sources 10.0.0.0/8
ipset add trusted-sources 172.16.0.0/12
ipset add trusted-sources 192.168.0.0/16

# Hash:ip,port for combined IP+port matching
ipset create allowed-services hash:ip,port
ipset add allowed-services 10.0.1.5,tcp:443
ipset add allowed-services 10.0.1.5,tcp:8443

# Timed entries (automatically expire after N seconds)
ipset create temp-block hash:ip timeout 3600
ipset add temp-block 203.0.113.99 timeout 600   # expire in 10 min

# --- Using sets in iptables rules ---

# O(1) allowlist lookup for trusted CIDR accepting port 8443
iptables -I INPUT -m set --match-set trusted-sources src \
  -p tcp --dport 8443 -j ACCEPT

# Block all sources in a blocklist
iptables -I INPUT -m set --match-set blocked-ips src -j DROP

# Match both source IP and destination port using hash:ip,port
iptables -I INPUT -m set --match-set allowed-services src,dst -j ACCEPT

# --- Atomic blocklist update (avoid traffic gaps) ---

# Load new blocklist into a temporary set
ipset create new-blocklist hash:net
# populate new-blocklist with updated entries...
ipset add new-blocklist 198.51.100.0/24

# Atomically swap old and new (zero-downtime update)
ipset swap blocked-ips new-blocklist
ipset destroy new-blocklist

# --- Persistence ---

# Save all sets to file
ipset save > /etc/ipset.conf

# Restore sets from file
ipset restore < /etc/ipset.conf

# --- Inspection ---

# List all sets
ipset list

# Test if an IP is in a set (exit 0 = member, 1 = not member)
ipset test trusted-sources 10.0.1.50
echo $?   # 0 if member

# Count entries in a set
ipset list blocked-ips | grep "^Number of entries"

# Flush a set without destroying it (keep iptables rule intact)
ipset flush blocked-ips

# --- nftables equivalent ---
# nft add set inet filter blocked-ips { type ipv4_addr; flags interval; }
# nft add element inet filter blocked-ips { 198.51.100.0/24 }
# nft add rule inet filter input ip saddr @blocked-ips drop
```

---

