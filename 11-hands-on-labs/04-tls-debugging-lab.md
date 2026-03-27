# Lab 04: TLS Debugging

## Learning Objectives

- Build a complete PKI chain (root CA -> intermediate CA -> leaf cert) using only openssl
- Reproduce and diagnose the three most common TLS failures: expired cert, missing intermediate, wrong CA for mTLS
- Understand what `openssl s_client` output tells you at each stage of the handshake

## Prerequisites

- Ubuntu 22.04 or macOS with openssl 1.1.1+ or 3.x
- nginx installed: `sudo apt-get install -y nginx`
- openssl: `openssl version` should return 1.1.1 or 3.x
- All labs run on localhost (no DNS required)

```bash
# Verify prerequisites
openssl version
# Expected: OpenSSL 3.0.x or OpenSSL 1.1.1x

nginx -v
# Expected: nginx version: nginx/1.18.x or higher

# Create working directory for all lab artifacts
mkdir -p /tmp/tls-lab && cd /tmp/tls-lab
export LAB_DIR=/tmp/tls-lab
```

## Debugging Methodology Alignment

TLS failures appear as L5 (application) errors but root cause is L4/L5 certificate validation.
The 5-question checklist applied to TLS:
1. Is the service listening? (`ss -tnlp | grep 443`)
2. Can the TCP connection be established? (`openssl s_client -connect host:443`)
3. Does the cert chain validate? (`openssl verify -CAfile rootCA.pem cert.pem`)
4. Is the cert expired/not yet valid? (`openssl x509 -noout -dates`)
5. Does the SANs/CN match the hostname? (`openssl x509 -noout -text | grep -A1 "Subject Alternative"`)

---

## Lab 4A: Build Your Own PKI (CA -> Intermediate -> Leaf Cert)

**Objective:** Create a 3-level certificate chain from scratch. Understand what each layer does and why intermediate CAs exist.

---

### Setup: Create Root CA

```bash
cd $LAB_DIR

# Step 1: Generate Root CA private key (4096-bit RSA)
openssl genrsa -out rootCA.key 4096
# Expected: Generating RSA private key, 4096 bit...

# Step 2: Create self-signed Root CA certificate (valid 10 years)
openssl req -new -x509 \
  -key rootCA.key \
  -sha256 \
  -days 3650 \
  -out rootCA.pem \
  -subj "/C=US/ST=California/O=LabCA/CN=Lab Root CA"

# Verify Root CA
openssl x509 -noout -text -in rootCA.pem | grep -E "Subject:|Issuer:|Not (Before|After)|CA:"
# Expected:
# Issuer: C=US, ST=California, O=LabCA, CN=Lab Root CA
# Not Before: <today>
# Not After : <10 years from now>
# Subject: C=US, ST=California, O=LabCA, CN=Lab Root CA   (self-signed: issuer=subject)
# CA: TRUE
```

### Setup: Create Intermediate CA

```bash
# Step 3: Generate Intermediate CA private key
openssl genrsa -out intermediateCA.key 4096

# Step 4: Create CSR for Intermediate CA
openssl req -new \
  -key intermediateCA.key \
  -out intermediateCA.csr \
  -subj "/C=US/ST=California/O=LabCA/CN=Lab Intermediate CA"

# Step 5: Create extension file for intermediate CA
# Critical: must have CA:TRUE and pathlen:0 to allow signing leaf certs but not sub-CAs
cat > intermediate-ext.cnf << 'EOF'
[v3_intermediate_ca]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
EOF

# Step 6: Sign Intermediate CA with Root CA
openssl x509 -req \
  -in intermediateCA.csr \
  -CA rootCA.pem \
  -CAkey rootCA.key \
  -CAcreateserial \
  -out intermediateCA.pem \
  -days 1825 \
  -sha256 \
  -extensions v3_intermediate_ca \
  -extfile intermediate-ext.cnf

# Verify Intermediate CA
openssl x509 -noout -text -in intermediateCA.pem | grep -E "Issuer:|Subject:|CA:|pathlen"
# Expected:
# Issuer: CN=Lab Root CA         (signed by root)
# Subject: CN=Lab Intermediate CA (this cert)
# CA: TRUE
# pathlen: 0

# Verify chain: intermediate -> root
openssl verify -CAfile rootCA.pem intermediateCA.pem
# Expected: intermediateCA.pem: OK
```

### Setup: Create Server (Leaf) Certificate

```bash
# Step 7: Generate server private key
openssl genrsa -out server.key 2048

# Step 8: Create server CSR with SAN extension config
cat > server-ext.cnf << 'EOF'
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
subjectAltName = @alt_names
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[alt_names]
DNS.1 = localhost
DNS.2 = *.localhost
IP.1 = 127.0.0.1
EOF

openssl req -new \
  -key server.key \
  -out server.csr \
  -subj "/C=US/ST=California/O=LabApp/CN=localhost" \
  -config server-ext.cnf

# Step 9: Sign server cert with Intermediate CA
openssl x509 -req \
  -in server.csr \
  -CA intermediateCA.pem \
  -CAkey intermediateCA.key \
  -CAcreateserial \
  -out server.pem \
  -days 365 \
  -sha256 \
  -extensions v3_req \
  -extfile server-ext.cnf

# Step 10: Create full chain bundle (server + intermediate — DO NOT include root)
cat server.pem intermediateCA.pem > fullchain.pem

# Verify the full chain
openssl verify -CAfile rootCA.pem -untrusted intermediateCA.pem server.pem
# Expected: server.pem: OK

openssl verify -CAfile rootCA.pem fullchain.pem
# Expected: fullchain.pem: OK
```

### Setup: Configure nginx with TLS

```bash
# Create nginx config
sudo tee /etc/nginx/sites-available/tls-lab << 'EOF'
server {
    listen 8443 ssl;
    server_name localhost;

    ssl_certificate     /tmp/tls-lab/fullchain.pem;
    ssl_certificate_key /tmp/tls-lab/server.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        return 200 "TLS lab server OK\n";
        add_header Content-Type text/plain;
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/tls-lab /etc/nginx/sites-enabled/
sudo nginx -t && sudo nginx -s reload
```

### Verify: Test the Full PKI Chain

```bash
# Test TLS connection — provide our root CA as the trust anchor
openssl s_client -connect localhost:8443 -CAfile $LAB_DIR/rootCA.pem -quiet
# Expected:
# depth=2 CN=Lab Root CA
# verify return:1
# depth=1 CN=Lab Intermediate CA
# verify return:1
# depth=0 CN=localhost
# verify return:1
# (connection stays open)
# Type Ctrl+C to exit

# Or use curl with CA override
curl -s --cacert $LAB_DIR/rootCA.pem https://localhost:8443/
# Expected: TLS lab server OK

# Inspect the certificate chain presented by server
openssl s_client -connect localhost:8443 -CAfile $LAB_DIR/rootCA.pem \
  -showcerts 2>/dev/null | openssl x509 -noout -subject -issuer
# Expected:
# subject=CN=localhost
# issuer=CN=Lab Intermediate CA
```

**Key Takeaway:** A production PKI chain is Root CA -> Intermediate CA -> Leaf. The Root CA's private key is kept offline (or in HSM). Intermediate CAs are what actually sign leaf certs — this limits exposure if an intermediate is compromised. The server presents leaf + intermediate in `fullchain.pem`; the client validates up to a trusted root.

---

## Lab 4B: Reproduce and Debug Certificate Expiry

**Objective:** Create a cert that expires in the near past (or very soon), observe the error, and practice diagnosing expiry issues.

---

### Setup: Create a Short-Lived (Already Expired) Certificate

```bash
cd $LAB_DIR

# Create a cert that expires in 1 day
# To simulate an already-expired cert, we use -startdate/-enddate with a past date
# For OpenSSL 3.x:
openssl genrsa -out expired.key 2048

openssl req -new -key expired.key -out expired.csr \
  -subj "/CN=localhost"

# Sign with 1-day validity (will expire almost immediately)
openssl x509 -req \
  -in expired.csr \
  -CA intermediateCA.pem \
  -CAkey intermediateCA.key \
  -CAcreateserial \
  -out expired.pem \
  -days 1 \
  -sha256

# For truly already-expired: backdate the certificate using fake-time
# (requires libfaketime: apt-get install faketime)
faketime '2020-01-01 00:00:00' openssl x509 -req \
  -in expired.csr \
  -CA intermediateCA.pem \
  -CAkey intermediateCA.key \
  -CAcreateserial \
  -out expired.pem \
  -days 1 \
  -sha256

# Verify the expiry date
openssl x509 -noout -dates -in expired.pem
# Expected:
# notBefore=Jan  1 00:00:00 2020 GMT
# notAfter=Jan  2 00:00:00 2020 GMT    <-- expired in 2020
```

### The Break: Deploy Expired Cert to nginx

```bash
sudo tee /etc/nginx/sites-available/tls-lab << 'EOF'
server {
    listen 8443 ssl;
    server_name localhost;
    ssl_certificate     /tmp/tls-lab/expired.pem;
    ssl_certificate_key /tmp/tls-lab/expired.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    location / {
        return 200 "expired cert server\n";
        add_header Content-Type text/plain;
    }
}
EOF
sudo nginx -s reload
```

### Symptoms

```bash
# curl will reject the expired cert
curl -s --cacert $LAB_DIR/rootCA.pem https://localhost:8443/
# Expected:
# curl: (60) SSL certificate problem: certificate has expired
# More details: https://curl.se/docs/sslcerts.html

# With verbose output
curl -v --cacert $LAB_DIR/rootCA.pem https://localhost:8443/ 2>&1 | grep -E "expire|SSL|TLS|error"
# Expected:
# * SSL certificate verify result: certificate has expired (10), continuing anyway...
# OR
# * SSL certificate problem: certificate has expired
```

### Diagnosis

```bash
# Step 1: Check certificate dates directly
openssl x509 -noout -dates -in /tmp/tls-lab/expired.pem
# Expected:
# notBefore=Jan  1 00:00:00 2020 GMT
# notAfter=Jan  2 00:00:00 2020 GMT

# Compare to current time
date -u
# Expected: current date > notAfter = EXPIRED

# Step 2: Use s_client to see the TLS handshake error
openssl s_client -connect localhost:8443 -CAfile $LAB_DIR/rootCA.pem 2>&1 | \
  grep -E "Verify return:|error|expire|notAfter"
# Expected:
# verify error:num=10:certificate has expired
# notAfter=Jan  2 00:00:00 2020 GMT
# Verify return code: 10 (certificate has expired)

# Step 3: Check ALL certs in the chain for expiry
openssl s_client -connect localhost:8443 -showcerts 2>/dev/null | \
  while openssl x509 -noout -subject -dates 2>/dev/null; do true; done
# Shows notBefore/notAfter for each cert in the chain

# Step 4: How much time is left? (Negative = already expired)
# OpenSSL 3.x:
openssl x509 -noout -checkend 0 -in expired.pem
# Returns exit code 1 if expired:
# Certificate will expire
# Exit code: $? = 1

# Check if expiring within next 30 days (useful in monitoring):
openssl x509 -noout -checkend 2592000 -in server.pem
# Exit 0 = still valid for 30 days; exit 1 = expires within 30 days
```

### Fix: Renew the Certificate

```bash
# Generate a new cert with proper validity period
openssl x509 -req \
  -in expired.csr \
  -CA $LAB_DIR/intermediateCA.pem \
  -CAkey $LAB_DIR/intermediateCA.key \
  -CAcreateserial \
  -out renewed.pem \
  -days 365 \
  -sha256

# Update nginx to use renewed cert
sudo tee /etc/nginx/sites-available/tls-lab << 'EOF'
server {
    listen 8443 ssl;
    server_name localhost;
    ssl_certificate     /tmp/tls-lab/renewed.pem;
    ssl_certificate_key /tmp/tls-lab/expired.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    location / {
        return 200 "renewed cert\n";
        add_header Content-Type text/plain;
    }
}
EOF
sudo nginx -s reload
```

### Verify

```bash
curl -s --cacert $LAB_DIR/rootCA.pem https://localhost:8443/
# Expected: renewed cert

openssl x509 -noout -checkend 0 -in renewed.pem
# Expected: Certificate will not expire    (exit code 0)
```

**Key Takeaway:** `openssl x509 -noout -checkend 0` is the fastest way to check if a cert is currently expired (exit code 1 = expired). For monitoring, use `-checkend 2592000` (30 days) as an alerting threshold. In production, automate cert renewal with cert-manager or Let's Encrypt before manual emergency procedures are needed.

---

## Lab 4C: Debug Chain of Trust Failure (Missing Intermediate)

**Objective:** Reproduce the "unable to get local issuer certificate" error that happens when a server only sends the leaf cert without the intermediate.

---

### The Break: Deploy Leaf Cert Only (No Intermediate in Chain)

```bash
cd $LAB_DIR

# Configure nginx with ONLY the leaf cert — no intermediate
sudo tee /etc/nginx/sites-available/tls-lab << 'EOF'
server {
    listen 8443 ssl;
    server_name localhost;
    # BUG: using server.pem (leaf only) instead of fullchain.pem
    ssl_certificate     /tmp/tls-lab/server.pem;
    ssl_certificate_key /tmp/tls-lab/server.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    location / {
        return 200 "leaf-only cert\n";
        add_header Content-Type text/plain;
    }
}
EOF
sudo nginx -s reload
```

### Symptoms

```bash
# curl fails with a chain-of-trust error
curl -s --cacert $LAB_DIR/rootCA.pem https://localhost:8443/
# Expected:
# curl: (60) SSL certificate problem: unable to get local issuer certificate

# Note: NOT "certificate has expired" — the cert is valid
# The problem is: curl has the root CA, but cannot build a chain from:
#   leaf cert (signed by Intermediate CA) -> ??? -> Root CA
# because the server never sent the intermediate
```

### Diagnosis

```bash
# Step 1: Check what certs the server is actually sending
openssl s_client -connect localhost:8443 -CAfile $LAB_DIR/rootCA.pem 2>&1 | \
  grep -E "Certificate chain|depth|verify"
# Expected (incomplete chain):
# Certificate chain
#  0 s:CN = localhost
#    i:CN = Lab Intermediate CA
# verify error:num=20:unable to get local issuer certificate
# verify error:num=21:unable to verify the first certificate
# Verify return code: 21 (unable to verify the first certificate)

# Note: depth 0 is the leaf cert, issued by "Lab Intermediate CA"
# depth 1 should be the intermediate, but it's MISSING from the chain
# The client has Root CA but cannot find Intermediate CA to complete the chain

# Step 2: Count certs in chain (should be 2: leaf + intermediate; we have 1)
openssl s_client -connect localhost:8443 -showcerts 2>/dev/null | \
  grep -c "BEGIN CERTIFICATE"
# Expected: 1 (only leaf cert — missing intermediate)
# Correct value should be: 2

# Step 3: Verify the cert itself is fine (chain validation with explicit intermediate)
openssl verify \
  -CAfile $LAB_DIR/rootCA.pem \
  -untrusted $LAB_DIR/intermediateCA.pem \
  $LAB_DIR/server.pem
# Expected: server.pem: OK
# (Adding -untrusted provides the intermediate manually — this confirms the cert is valid,
#  the problem is the server not sending the intermediate)

# Step 4: Confirm issuer of leaf cert
openssl x509 -noout -issuer -in $LAB_DIR/server.pem
# Expected: issuer=CN=Lab Intermediate CA
# This tells us: we need intermediateCA.pem in the chain
```

### Root Cause

The server is only presenting the leaf certificate (server.pem). The client has the root CA in its trust store, but cannot validate the leaf cert because it was signed by the intermediate CA — and the intermediate CA certificate was not provided by the server. TLS requires the server to send the full chain up to (but not including) the root.

### Fix: Add Intermediate to nginx ssl_certificate

```bash
# Confirm fullchain.pem contains both certs
grep -c "BEGIN CERTIFICATE" $LAB_DIR/fullchain.pem
# Expected: 2

# Fix: use fullchain.pem (leaf + intermediate)
sudo tee /etc/nginx/sites-available/tls-lab << 'EOF'
server {
    listen 8443 ssl;
    server_name localhost;
    ssl_certificate     /tmp/tls-lab/fullchain.pem;
    ssl_certificate_key /tmp/tls-lab/server.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    location / {
        return 200 "full chain cert\n";
        add_header Content-Type text/plain;
    }
}
EOF
sudo nginx -s reload
```

### Verify

```bash
# Now curl should succeed
curl -s --cacert $LAB_DIR/rootCA.pem https://localhost:8443/
# Expected: full chain cert

# Confirm full chain is being sent
openssl s_client -connect localhost:8443 -showcerts 2>/dev/null | \
  grep -c "BEGIN CERTIFICATE"
# Expected: 2

openssl s_client -connect localhost:8443 -CAfile $LAB_DIR/rootCA.pem 2>&1 | \
  grep "Verify return code"
# Expected: Verify return code: 0 (ok)
```

**Key Takeaway:** "Unable to get local issuer certificate" means the chain is incomplete — the server is not sending the intermediate. Fix: concatenate `cat leaf.pem intermediate.pem > fullchain.pem` and point `ssl_certificate` at the bundle. Never include the root CA in the bundle; it's always in the client's trust store.

---

## Lab 4D: mTLS Setup and Debugging

**Objective:** Configure mutual TLS (both server and client present certificates), then reproduce the "wrong CA for client cert" error and debug it.

---

### Setup: Create Client CA and Client Certificate

```bash
cd $LAB_DIR

# Create a dedicated Client CA (separate from the server PKI)
openssl genrsa -out clientCA.key 4096

openssl req -new -x509 \
  -key clientCA.key \
  -sha256 \
  -days 1825 \
  -out clientCA.pem \
  -subj "/C=US/ST=California/O=LabCA/CN=Lab Client CA"

# Create client private key and CSR
openssl genrsa -out client.key 2048

openssl req -new \
  -key client.key \
  -out client.csr \
  -subj "/C=US/ST=California/O=LabClient/CN=test-client"

# Sign client cert with Client CA
openssl x509 -req \
  -in client.csr \
  -CA clientCA.pem \
  -CAkey clientCA.key \
  -CAcreateserial \
  -out client.pem \
  -days 365 \
  -sha256

echo "Client cert created and signed by Client CA"

# Verify client cert chain
openssl verify -CAfile $LAB_DIR/clientCA.pem client.pem
# Expected: client.pem: OK
```

### Setup: Configure nginx for mTLS (Correct Setup First)

```bash
sudo tee /etc/nginx/sites-available/tls-lab << 'EOF'
server {
    listen 8443 ssl;
    server_name localhost;

    ssl_certificate     /tmp/tls-lab/fullchain.pem;
    ssl_certificate_key /tmp/tls-lab/server.key;

    # mTLS: require client certificate
    ssl_client_certificate /tmp/tls-lab/clientCA.pem;
    ssl_verify_client on;
    ssl_verify_depth 2;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        return 200 "mTLS OK — client: $ssl_client_s_dn\n";
        add_header Content-Type text/plain;
    }
}
EOF
sudo nginx -t && sudo nginx -s reload
```

### Verify: Correct mTLS Works

```bash
# Test WITH client cert (should succeed)
curl -s \
  --cacert $LAB_DIR/rootCA.pem \
  --cert $LAB_DIR/client.pem \
  --key $LAB_DIR/client.key \
  https://localhost:8443/
# Expected: mTLS OK — client: CN=test-client, O=LabClient, ST=California, C=US

# Test WITHOUT client cert (should fail — nginx requires one)
curl -s --cacert $LAB_DIR/rootCA.pem https://localhost:8443/
# Expected:
# <html><body>400 No required SSL certificate was sent</body></html>
```

### The Break: Use Wrong CA for Client Certificate

```bash
cd $LAB_DIR

# Create a ROGUE CA (not trusted by the nginx ssl_client_certificate)
openssl genrsa -out rogueCA.key 4096

openssl req -new -x509 \
  -key rogueCA.key \
  -sha256 \
  -days 365 \
  -out rogueCA.pem \
  -subj "/CN=Rogue CA"

# Create a client cert signed by the ROGUE CA (not by clientCA)
openssl genrsa -out rogue-client.key 2048

openssl req -new -key rogue-client.key -out rogue-client.csr \
  -subj "/CN=rogue-client"

openssl x509 -req \
  -in rogue-client.csr \
  -CA rogueCA.pem \
  -CAkey rogueCA.key \
  -CAcreateserial \
  -out rogue-client.pem \
  -days 365 \
  -sha256

echo "Rogue client cert created — signed by Rogue CA, not trusted by nginx"
```

### Symptoms

```bash
# Attempt mTLS with the rogue client cert
curl -v \
  --cacert $LAB_DIR/rootCA.pem \
  --cert $LAB_DIR/rogue-client.pem \
  --key $LAB_DIR/rogue-client.key \
  https://localhost:8443/ 2>&1 | grep -E "SSL|error|400|handshake"
# Expected:
# * SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
# * TLSv1.3 (IN), TLS handshake, CERT verify (15):
# * SSL certificate problem: unable to get local issuer certificate
# OR nginx responds with HTTP 400

# Check nginx error log for detail
sudo tail -20 /var/log/nginx/error.log
# Expected:
# SSL_do_handshake() failed (SSL: error:...certificate verify failed) ...
# client certificate verify error: (20:unable to get local issuer certificate)
# client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1"
```

### Diagnosis

```bash
# Step 1: Confirm which CA the rogue cert was signed by
openssl x509 -noout -issuer -in $LAB_DIR/rogue-client.pem
# Expected: issuer=CN=Rogue CA

# Step 2: Confirm which CA nginx trusts for client certs
grep ssl_client_certificate /etc/nginx/sites-available/tls-lab
# Expected: ssl_client_certificate /tmp/tls-lab/clientCA.pem;

openssl x509 -noout -subject -in $LAB_DIR/clientCA.pem
# Expected: subject=CN=Lab Client CA
# MISMATCH: nginx trusts "Lab Client CA" but rogue cert was signed by "Rogue CA"

# Step 3: Simulate the verification failure openssl would perform
openssl verify -CAfile $LAB_DIR/clientCA.pem $LAB_DIR/rogue-client.pem
# Expected:
# rogue-client.pem: CN=rogue-client
# error 20 at 0 depth lookup: unable to get local issuer certificate
# error rogue-client.pem: verification failed

# Step 4: Use openssl s_client with client cert to see the full handshake
openssl s_client \
  -connect localhost:8443 \
  -CAfile $LAB_DIR/rootCA.pem \
  -cert $LAB_DIR/rogue-client.pem \
  -key $LAB_DIR/rogue-client.key \
  -brief 2>&1
# Expected:
# CONNECTION FAILURE
# SSL handshake has read ... bytes and written ... bytes
# Verification error: certificate verify failed
```

### Root Cause

nginx's `ssl_client_certificate` directive specifies which CA(s) to trust for verifying client certificates. The client presented a cert signed by "Rogue CA", but nginx only trusts "Lab Client CA". The client certificate chain validation fails during the TLS handshake, and nginx terminates the connection before any HTTP is exchanged (or returns 400 if configured to `ssl_verify_client optional`).

### Fix Option A: Sign Client Cert with the Correct CA

```bash
# The client needs a cert signed by clientCA (the one nginx trusts)
# Re-sign the client CSR with the correct CA
openssl x509 -req \
  -in rogue-client.csr \
  -CA $LAB_DIR/clientCA.pem \
  -CAkey $LAB_DIR/clientCA.key \
  -CAcreateserial \
  -out fixed-client.pem \
  -days 365 \
  -sha256

# Verify against the trusted CA
openssl verify -CAfile $LAB_DIR/clientCA.pem fixed-client.pem
# Expected: fixed-client.pem: OK
```

### Fix Option B: Add the Rogue CA to nginx's Trusted CA Bundle

```bash
# If the client cert is legitimate and you trust that CA,
# append it to the ssl_client_certificate file
cat $LAB_DIR/clientCA.pem $LAB_DIR/rogueCA.pem > $LAB_DIR/combined-clientCAs.pem

sudo tee /etc/nginx/sites-available/tls-lab << 'EOF'
server {
    listen 8443 ssl;
    server_name localhost;
    ssl_certificate     /tmp/tls-lab/fullchain.pem;
    ssl_certificate_key /tmp/tls-lab/server.key;
    ssl_client_certificate /tmp/tls-lab/combined-clientCAs.pem;
    ssl_verify_client on;
    ssl_protocols TLSv1.2 TLSv1.3;
    location / {
        return 200 "mTLS OK — client: $ssl_client_s_dn\n";
        add_header Content-Type text/plain;
    }
}
EOF
sudo nginx -t && sudo nginx -s reload
```

### Verify

```bash
# Test with rogue client cert (now trusted via Option B)
curl -s \
  --cacert $LAB_DIR/rootCA.pem \
  --cert $LAB_DIR/rogue-client.pem \
  --key $LAB_DIR/rogue-client.key \
  https://localhost:8443/
# Expected: mTLS OK — client: CN=rogue-client

# Or test with fixed-client.pem (Option A)
curl -s \
  --cacert $LAB_DIR/rootCA.pem \
  --cert $LAB_DIR/fixed-client.pem \
  --key $LAB_DIR/rogue-client.key \
  https://localhost:8443/
# Expected: mTLS OK — client: CN=rogue-client
```

### Cleanup

```bash
sudo rm /etc/nginx/sites-enabled/tls-lab
sudo nginx -s reload
rm -rf /tmp/tls-lab
```

**Key Takeaway:** mTLS "400 No required SSL certificate was sent" has two causes: (1) client didn't present a cert at all, or (2) client cert is signed by a CA the server doesn't trust. The diagnostic command is `openssl verify -CAfile <server-trusted-CA> <client-cert>`. The issuer field of the client cert must chain to a CA listed in `ssl_client_certificate`.

---

## Summary: TLS Debugging Lab Checklist

| Lab  | Error Message | Root Cause | Fix |
|------|--------------|-----------|-----|
| 4A   | (baseline)   | PKI construction | fullchain = leaf + intermediate |
| 4B   | certificate has expired | cert past notAfter | renew cert, use -checkend in monitoring |
| 4C   | unable to get local issuer | server sent leaf only | use fullchain.pem in ssl_certificate |
| 4D   | certificate verify failed | client cert signed by untrusted CA | re-sign with correct CA or add CA to ssl_client_certificate |

## Quick Reference: openssl s_client Cheatsheet

```bash
# Basic TLS check
openssl s_client -connect host:443

# With specific CA file
openssl s_client -connect host:443 -CAfile /etc/ssl/certs/ca-certificates.crt

# Show full certificate chain
openssl s_client -connect host:443 -showcerts

# mTLS: present client cert
openssl s_client -connect host:443 -cert client.pem -key client.key

# Check specific TLS version
openssl s_client -connect host:443 -tls1_2
openssl s_client -connect host:443 -tls1_3

# Non-interactive (just get certificate info and exit)
echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -text
```

## Interview Discussion Points

1. "Walk me through a TLS handshake." — ClientHello, ServerHello, server sends certificate chain, client validates chain, key exchange, Finished messages.
2. "How do you debug 'certificate verify failed'?" — openssl s_client, check depth/verify return, identify which cert in chain fails, check dates, check issuer chain continuity.
3. "What's in fullchain.pem vs just cert.pem?" — fullchain includes leaf + intermediate(s). Server sends this so clients can build the trust chain without already having the intermediate in their store.
4. "What's the difference between SSL_VERIFY_PEER and mTLS?" — mTLS = both sides authenticate with certificates. Standard TLS = only server authenticates. Used in service mesh, API gateways, B2B integrations.
