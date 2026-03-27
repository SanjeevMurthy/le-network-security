# OWASP API Security Top 10

## Overview

APIs are the nervous system of modern distributed architecture — microservices, mobile backends, SaaS integrations, and partner ecosystems all communicate through APIs. This concentration of functionality in API endpoints makes them the highest-impact attack surface in contemporary software. The OWASP API Security Top 10 (2023 edition) reflects a critical insight: five of the ten risks are authorization failures, not injection or transport vulnerabilities. Traditional WAFs and network controls provide limited protection against business logic attacks that abuse legitimate API functionality.

For a senior SRE, API security is inseparable from reliability: a BOLA vulnerability leaking PII is also a data breach triggering incident response. An SSRF attack via an image upload feature that exfiltrates IAM credentials escalates from an application bug to a full cloud account compromise. Understanding the attack mechanics, the network-level controls that help, and the application-level fixes that are required is essential.

```mermaid
flowchart LR
    subgraph Controls["Defense Layers"]
        WAF2[WAF\nL7 rules + rate limiting]
        GW[API Gateway\nAuthn + rate limits + request validation]
        APP2[Application\nAuthZ checks + input validation]
        DB2[Database\nParameterized queries + RBAC]
    end

    subgraph Threats["OWASP API Top 10"]
        BOLA[API1: BOLA\nObject-level AuthZ]
        AUTH[API2: Broken Auth\nJWT attacks + token theft]
        BOPLA[API3: BOPLA\nMass assignment + data leak]
        RES[API4: Resource Consumption\nRate limit bypass]
        BFLA[API5: BFLA\nFunction-level AuthZ]
        BFLOW[API6: Business Flow\nAutomated abuse]
        SSRF2[API7: SSRF\nMetadata endpoint]
        MISC[API8: Misconfiguration\nCORS + debug endpoints]
        INV[API9: Inventory\nShadow APIs]
        CONS[API10: Unsafe Consumption\nThird-party trust]
    end

    WAF2 --> RES & SSRF2 & MISC
    GW --> AUTH & BFLA & RES & BFLOW
    APP2 --> BOLA & BOPLA & BFLA & CONS
    DB2 --> BOLA & BOPLA
```

---

## API1: Broken Object Level Authorization (BOLA)

**The most prevalent API vulnerability.** BOLA (also called IDOR — Insecure Direct Object Reference) occurs when an API endpoint accepts an object identifier from the client and fails to verify that the authenticated user is authorized to access that specific object.

### Attack Mechanics

```http
# User 67890 is authenticated. They guess or enumerate account IDs.
GET /api/v1/accounts/12345/transactions
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...<token for user 67890>

# Server checks: "Is this a valid token?" YES
# Server does NOT check: "Does token user 67890 own account 12345?"
# Result: Full transaction history of account 12345 returned
```

**Object ID enumeration:** Sequential integer IDs are trivially enumerable. GUIDs are not secret but are harder to guess randomly. Neither is a substitute for authorization checks.

### Network-Level Mitigations

- WAF cannot detect BOLA — the requests are syntactically valid
- Rate limiting at the API gateway slows enumeration but does not prevent it
- Behavioral analytics can detect unusual patterns (user accessing hundreds of different object IDs)

### Application Fix

```python
# WRONG: Only checks authentication
@app.route("/api/v1/accounts/<account_id>/transactions")
def get_transactions(account_id):
    account = db.get_account(account_id)  # No ownership check
    return account.transactions

# CORRECT: Checks ownership explicitly
@app.route("/api/v1/accounts/<account_id>/transactions")
def get_transactions(account_id):
    account = db.get_account(account_id)
    if account.owner_id != current_user.id:  # Enforce ownership
        abort(403)
    return account.transactions

# BETTER: Never accept user-provided IDs; derive from auth context
@app.route("/api/v1/my/transactions")  # No ID in URL
def get_my_transactions():
    account = db.get_account_by_user(current_user.id)  # ID from token, not URL
    return account.transactions
```

---

## API2: Broken Authentication

Authentication mechanisms are exposed by design — every API consumer calls the authentication endpoint. Weaknesses include: weak token signing, accepting degenerate token formats, missing brute-force protections, and long-lived tokens.

### JWT Algorithm Confusion Attack

```python
# VULNERABLE: Server uses RS256 (asymmetric). Attacker knows the PUBLIC KEY.
# Attacker modifies header: alg: HS256 (symmetric)
# Attacker signs the token with the PUBLIC KEY as the HMAC secret
# Naive JWT library: "alg=HS256, verify with configured key" — but key is RS256 public key!

import jwt
import base64

# Attacker's exploit
public_key = get_server_public_key()  # Publicly available at /jwks.json
malicious_payload = {"user_id": "admin", "role": "superadmin", "exp": 9999999999}

# Sign with public key treated as HMAC secret
malicious_token = jwt.encode(
    malicious_payload,
    public_key,
    algorithm="HS256"  # Algorithm confusion: public key as HMAC secret
)
```

**Fix: Always specify the expected algorithm explicitly**
```python
# SECURE: Reject any token with unexpected algorithm
EXPECTED_ALGORITHM = "RS256"

def verify_token(token):
    header = jwt.get_unverified_header(token)
    if header["alg"] != EXPECTED_ALGORITHM:
        raise ValueError(f"Unexpected algorithm: {header['alg']}")

    return jwt.decode(
        token,
        public_key,
        algorithms=[EXPECTED_ALGORITHM],  # Explicit allowlist
        audience="my-api",
        issuer="https://auth.mycompany.com"
    )
```

### Network-Level Mitigations

- Rate limiting on authentication endpoints: 5 attempts per minute per IP/account
- Progressive delays: exponential backoff after failed attempts
- CAPTCHA integration for high-failure-rate patterns
- IP reputation blocking at WAF layer for credential stuffing sources

---

## API3: Broken Object Property Level Authorization (BOPLA)

Combines two 2019 risks: Excessive Data Exposure (API returns too many fields) and Mass Assignment (write side accepts too many fields). Root cause: failing to validate which properties a user can read or write.

### Attack Mechanics

**Read side (data leakage):**
```json
// API returns full ORM object including internal/sensitive fields
GET /api/v1/users/me
{
  "id": "u-123",
  "name": "Alice",
  "email": "alice@corp.com",
  "ssn": "123-45-6789",           // PII never intended for API response
  "internal_risk_score": 42,       // Internal business metric leaked
  "is_fraud_flagged": true,        // Security-sensitive field leaked
  "admin_notes": "suspicious activity noted 2026-01-15"
}
```

**Write side (mass assignment):**
```http
PATCH /api/v1/users/me
{"name": "Alice", "role": "admin", "subscription_tier": "enterprise", "credits": 99999}
```

### Fix: Explicit Schema Validation

```python
# WRONG: Serialize entire ORM model
@app.route("/api/v1/users/me")
def get_me():
    return jsonify(current_user.__dict__)  # Leaks all fields

# CORRECT: Use a response DTO with explicit field list
class UserPublicDTO(Schema):
    id = fields.Str(dump_only=True)
    name = fields.Str()
    email = fields.Email()
    # NOTE: no ssn, risk_score, is_fraud_flagged fields

@app.route("/api/v1/users/me")
def get_me():
    return UserPublicDTO().dump(current_user)  # Only allowed fields

# CORRECT write-side: Allowlist writable fields per role
class UserUpdateDTO(Schema):
    name = fields.Str(required=False)
    email = fields.Email(required=False)
    # Fields NOT here: role, subscription_tier, credits, is_admin

    class Meta:
        unknown = RAISE  # Reject any unexpected fields
```

---

## API4: Unrestricted Resource Consumption

APIs that do not limit resource consumption are vulnerable to denial-of-service and financial exhaustion. This includes missing rate limits, unbounded pagination, large payloads, and expensive operations without throttling.

### Attack Scenarios

```http
# Pagination bypass: request all records
GET /api/v1/orders?page=1&per_page=1000000

# GraphQL depth attack: exponentially expensive query
query {
  user { posts { comments { author { posts { comments { author { ... } } } } } } }
}

# Large payload: 500MB file upload when limit should be 10MB
POST /api/v1/documents
Content-Length: 524288000

# Cost amplification: SMS/email trigger per request
POST /api/v1/verify-phone?phone=+15555551234  # repeated 10,000 times = $3,000 SMS bill
```

### Network-Level + Gateway Controls

```yaml
# Kong API Gateway: rate limiting and request size limits
plugins:
  - name: rate-limiting
    config:
      minute: 100          # 100 requests per minute per user
      hour: 1000
      policy: local
      limit_by: consumer

  - name: request-size-limiting
    config:
      allowed_payload_size: 10  # 10 MB maximum

  - name: request-termination
    config:
      status_code: 413
      message: "Request too large"
```

**GraphQL query complexity limit:**
```javascript
// Apollo Server: reject queries exceeding complexity budget
const server = new ApolloServer({
  validationRules: [
    createComplexityLimitRule(1000, {  // Max complexity score
      scalarCost: 1,
      objectCost: 5,
      listFactor: 10,
    })
  ]
});
```

---

## API5: Broken Function Level Authorization (BFLA)

Where BOLA is about accessing someone else's *data*, BFLA is about accessing someone else's *functionality*. A regular user calls an admin endpoint.

### Attack Mechanics

```http
# Regular user discovers admin endpoint through:
# - JavaScript bundle analysis
# - Swagger/OpenAPI docs at /api-docs
# - HTTP method enumeration
DELETE /api/admin/users/12345
Authorization: Bearer <regular_user_token>

# HTTP method override (if server accepts X-HTTP-Method-Override)
POST /api/users/12345
X-HTTP-Method-Override: DELETE
Authorization: Bearer <regular_user_token>
```

### Fix: Default-Deny Function Authorization

```python
# WRONG: Implicit allow for most routes, explicit deny for admin
@app.route("/api/admin/users/<user_id>", methods=["DELETE"])
def delete_user(user_id):
    if not current_user.is_admin:  # Positive check for admin
        abort(403)

# BETTER: Policy engine enforces function-level authorization
# OPA policy: maps routes + methods to required roles
package api.authz

default allow = false

allow {
    required_roles := data.route_permissions[input.path][input.method]
    some role
    role := input.user.roles[_]
    role == required_roles[_]
}

# route_permissions.json
{
  "/api/admin/users": {"DELETE": ["admin", "superadmin"]},
  "/api/orders": {"GET": ["user", "admin"], "POST": ["user", "admin"]},
  "/api/admin/config": {"GET": ["admin"], "PUT": ["superadmin"]}
}
```

**Network-level mitigation:** Place admin API endpoints in a separate service or subnet accessible only from trusted internal networks. WAF rule to block requests to `/api/admin/*` from external IP ranges.

---

## API6: Unrestricted Access to Sensitive Business Flows

Business logic attacks that exploit legitimate functionality at scale or out of intended sequence. Unlike technical exploits, these use the API exactly as designed — just in ways that cause business harm.

### Attack Scenarios

- **Ticket scalping:** Bot purchases all available concert tickets via API in seconds before humans can
- **Credential stuffing:** Automated login attempts using leaked credential pairs across millions of accounts
- **Referral abuse:** Script creates 10,000 fake accounts to collect sign-up bonuses
- **Inventory manipulation:** Hold items in cart (reservation) indefinitely to prevent competitors/customers from purchasing

### Mitigations

```yaml
# AWS WAF Bot Control: block scraping bots
- name: AWSManagedRulesBotControlRuleSet
  sampledRequestsEnabled: true
  action:
    block: {}

# Application-level: business logic rate limits
# Different from API rate limits — these are per-business-flow
rate_limits:
  ticket_purchase_per_user_per_event: 4
  account_creation_per_ip_per_hour: 3
  referral_bonus_per_user: 1
  cart_hold_max_items: 10
  cart_hold_max_duration_minutes: 30
```

**Device fingerprinting:** Requires a consistent device fingerprint (not just IP) for business flows — bots that rotate IPs are still detectable via TLS fingerprint, browser properties, and behavioral timing.

---

## API7: Server-Side Request Forgery (SSRF)

SSRF occurs when an API fetches a remote resource based on user-supplied input without validating the destination. In cloud environments, this is critical because the cloud metadata service is accessible at `169.254.169.254` from any EC2 instance.

### Attack Mechanics

```http
# Feature: "import content from URL" or "webhook callback"
POST /api/webhooks
Content-Type: application/json
{"callback_url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}

# API server fetches this URL (internal network access)
# Response: AWS IAM temporary credentials leaked to attacker
{
  "Code": "Success",
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "wJal...",
  "Token": "FQoGZXIvYXdzEJr...",
  "Expiration": "2026-03-27T10:00:00Z"
}
```

**Cloud metadata targets:**
- AWS: `http://169.254.169.254/latest/meta-data/` (IMDSv1, no token required)
- GCP: `http://metadata.google.internal/computeMetadata/v1/`
- Azure: `http://169.254.169.254/metadata/instance?api-version=2021-02-01`

### Detection and Fix

```mermaid
flowchart LR
    REQ[API Request\nwith user URL] --> VALID[SSRF Validator]
    VALID -->|Check allowlist| ALLOW2[Allowed domain?]
    ALLOW2 -->|YES| FETCH[Fetch URL\nin sandboxed network]
    ALLOW2 -->|NO| REJECT[Return 422\nURL not allowed]
    FETCH -->|Redirect to private IP| REJECT
```

**SSRF filter implementation:**
```python
import ipaddress
from urllib.parse import urlparse

ALLOWED_SCHEMES = {"https"}
ALLOWED_DOMAINS = {"api.trusted-partner.com", "webhooks.external-service.com"}
BLOCKED_RANGES = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("169.254.0.0/16"),  # Link-local / metadata
    ipaddress.ip_network("127.0.0.0/8"),     # Loopback
    ipaddress.ip_network("::1/128"),          # IPv6 loopback
    ipaddress.ip_network("fc00::/7"),         # IPv6 unique local
]

def validate_url(url: str) -> bool:
    parsed = urlparse(url)

    if parsed.scheme not in ALLOWED_SCHEMES:
        return False

    if parsed.hostname not in ALLOWED_DOMAINS:
        return False

    # Resolve hostname to IP and check against blocked ranges
    import socket
    try:
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
        for blocked in BLOCKED_RANGES:
            if ip in blocked:
                return False
    except Exception:
        return False

    return True
```

**IMDSv2 enforcement (AWS):** IMDSv2 requires a PUT request with a TTL header to obtain a session token before metadata queries — SSRF attacks using simple GET requests are blocked.

```bash
# Enforce IMDSv2 on all EC2 instances via AWS Config remediation
aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890abcdef0 \
  --http-tokens required \            # IMDSv2: token required
  --http-put-response-hop-limit 1 \   # Block from containers
  --http-endpoint enabled
```

---

## API8: Security Misconfiguration

A catch-all for hardening failures. Critically important because misconfigurations are frequently exploitable with zero technical sophistication — default credentials, debug endpoints, and permissive CORS are attacked by automated scanners within hours of exposure.

### Critical Misconfigurations

**CORS wildcard with credentials:**
```http
# VULNERABLE: allows any origin to make credentialed requests
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

# CORRECT: explicit origin allowlist
Access-Control-Allow-Origin: https://app.mycompany.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST
```

**Debug endpoints in production:**
```bash
# Common debug endpoints that should NEVER be in production:
/debug/pprof          # Go profiling
/actuator/env         # Spring Boot: exposes env vars including secrets
/actuator/heapdump    # Spring Boot: full heap dump
/console              # H2 database console
/.well-known/          # Generally fine, but check contents
/__debug_toolbar__    # Django debug toolbar
/graphql/playground   # GraphQL playground (should be gated)
```

**TLS version enforcement:**
```nginx
# Nginx: reject TLS 1.0 and 1.1
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
```

### WAF Rules for Misconfiguration Detection

```yaml
# AWS WAF: block requests to debug endpoints
- name: BlockDebugEndpoints
  statement:
    orStatement:
      statements:
        - byteMatchStatement:
            searchString: "/actuator/"
            fieldToMatch: {uriPath: {}}
            textTransformations: [{priority: 0, type: LOWERCASE}]
            positionalConstraint: STARTS_WITH
        - byteMatchStatement:
            searchString: "/debug/pprof"
            fieldToMatch: {uriPath: {}}
            textTransformations: [{priority: 0, type: LOWERCASE}]
            positionalConstraint: STARTS_WITH
  action: {block: {}}
```

---

## API9: Improper Inventory Management

Shadow APIs — undocumented, forgotten, or automatically created endpoints — are attacked because they lack the security controls applied to known endpoints. You cannot secure what you do not know exists.

### Symptoms of Shadow APIs

```bash
# Old API versions still running
/api/v1/users  # Unmaintained, has known vulnerabilities
/api/v2/users  # Current, patched
/api/v3/users  # Beta, in development with no auth

# Development or staging endpoints exposed
/api/internal/admin  # Internal tooling accidentally exposed
/api/debug/tokens    # Debug endpoint from development

# Third-party integration endpoints
/webhooks/stripe     # Exposed but not documented
/webhooks/github     # No HMAC signature verification
```

### API Discovery and Inventory

```bash
# Generate OpenAPI spec from running application (Go example)
go install github.com/swaggo/swag/cmd/swag@latest
swag init  # Generates docs/swagger.json

# Discover APIs via traffic analysis (not source code)
# Capture production traffic patterns
kubectl sniff -n payments deploy/payments-api -o pcap | \
  tshark -r - -T fields -e http.request.uri | sort | uniq -c | sort -rn

# Check for zombie endpoints: in OpenAPI spec but receiving no traffic
# Cross-reference API gateway access logs with OpenAPI routes
```

**Automated API discovery tools:** Noname Security, Salt Security, Traceable — intercept production traffic to build an API inventory map including shadow APIs not in any documentation.

---

## API10: Unsafe Consumption of APIs

When your API integrates with third-party APIs and trusts their responses without validation, you inherit their risk. A compromised upstream API becomes a supply chain attack vector into your system.

### Attack Scenario

```
Your API ──→ Third-party Geocoding API (compromised by attacker)
                    │
                    └──→ Returns malicious data in "address" field:
                         {"city": "London", "address": "<script>stealCookies()</script>"}

Your API stores this unsanitized → Stored XSS/SQL injection in your database
Your frontend renders this → Users' browsers execute attacker's JavaScript
```

### Fix: Treat Third-Party Responses as Untrusted Input

```python
# WRONG: Trust geocoding API response directly
def store_address(user_id, url):
    response = requests.get(url)
    data = response.json()
    db.save_address(user_id, data["address"])  # Unvalidated

# CORRECT: Validate and sanitize third-party response
def store_address(user_id, url):
    # 1. Validate URL before fetching (SSRF prevention)
    if not is_allowed_geocoding_url(url):
        raise ValueError("URL not in allowlist")

    # 2. Set timeouts and size limits
    response = requests.get(url, timeout=5, stream=True)
    if int(response.headers.get("Content-Length", 0)) > 1024 * 100:  # 100KB max
        raise ValueError("Response too large")

    # 3. Validate response structure
    data = response.json()
    address_schema = AddressSchema(strict=True)
    clean_data = address_schema.load(data)  # Raises if unexpected fields or types

    # 4. Sanitize string fields
    clean_address = html.escape(clean_data["address"])
    db.save_address(user_id, clean_address)
```

---

## Real-World Production Scenario

### SSRF via Image Upload Feature Leaking AWS IAM Credentials

**The vulnerability:** A social media platform's profile photo upload feature accepts image URLs as well as file uploads. The URL fetch happens server-side.

**Attack chain:**
```http
# Step 1: Attacker submits a URL pointing to AWS metadata endpoint
POST /api/v1/users/profile-photo
{"image_url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}

# Step 2: Server fetches the URL from EC2 instance with IMDSv1 enabled
# Returns credential response

# Step 3: "Processed" image URL leaked in API response (a common bug)
GET /api/v1/users/current
{
  "profile_photo_url": "/uploads/12345.json",  # Contains credential data
  "profile_photo_raw": {"AccessKeyId": "ASIA...", "SecretAccessKey": "..."}
}

# Step 4: Attacker uses credentials
aws s3 ls --profile stolen  # List all S3 buckets
aws ec2 describe-instances  # Inventory all EC2 instances
aws iam list-roles           # Discover IAM roles for privilege escalation
```

**Detection signals:**
- CloudTrail: API calls from unusual source IP (not VPC range) using instance IAM role
- GuardDuty: `CredentialAccess:IAMUser/AnomalousBehavior` — credentials used from external IP
- VPC Flow Logs: outbound connections from EC2 to `169.254.169.254` (visible in logs)
- Application logs: unusual response from image URL fetch (Content-Type: application/json, not image/*)

**Remediation:**
```bash
# Immediate: Enforce IMDSv2 to block future SSRF attacks
aws ec2 modify-instance-metadata-options \
  --instance-id $(curl -s http://169.254.169.254/latest/meta-data/instance-id) \
  --http-tokens required

# Revoke the compromised role session
aws sts revoke-session --reason "Credential exposure via SSRF"

# Application fix: SSRF allowlist + Content-Type validation
def process_image_url(url: str):
    if not validate_url(url):  # SSRF filter (as above)
        raise HTTPException(422, "URL not permitted")

    response = requests.get(url, timeout=3)

    # Validate it's actually an image
    if not response.headers.get("Content-Type", "").startswith("image/"):
        raise HTTPException(422, "URL does not point to an image")

    # Process image from bytes, not from URL directly
    return process_image_bytes(response.content)
```

---

## Failure Modes

| Failure | Symptoms | Detection | Fix |
|---|---|---|---|
| BOLA via sequential IDs | Data leakage from other users' objects | Abnormal access patterns (many different IDs from one user) | UUID object IDs + ownership checks in every handler |
| JWT algorithm confusion | Forged tokens accepted with admin privileges | Anomalous admin actions from unexpected users | Explicit algorithm configuration; reject tokens with unexpected `alg` |
| Mass assignment via PATCH | Users escalate their own privileges | Unexpected role changes in audit log | Allowlist writable fields per endpoint; `additionalProperties: false` in schema |
| Rate limit bypass via IP rotation | Brute force succeeds; scraping undetected | High request volume from many IPs to same endpoint | Rate limit per user/account, not just per IP |
| SSRF to internal service | Internal service credentials or data exfiltrated | Unusual outbound connections from app servers; GuardDuty alerts | SSRF filter + IMDSv2 + run URL-fetching code in isolated network segment |
| CORS wildcard with credentials | Cross-origin credential theft | Browser security headers check; manual testing | Explicit origin allowlist; never combine `*` with credentials |
| Debug endpoint exposed | Actuator/health endpoint leaks env vars or credentials | Automated scanner findings | Route filtering at API gateway; WAF rules for `/actuator/*` |
| GraphQL depth attack | Application OOM, 503 errors | High CPU/memory on API servers; query timing anomalies | Query complexity limits; depth limits; persisted queries only |

---

## Debugging Guide

```bash
# Test API for common misconfigurations
# Check security headers
curl -I https://api.mycompany.com/api/v1/health | \
  grep -E "Strict-Transport|X-Content-Type|X-Frame|Content-Security|Access-Control"

# Check for debug endpoints
for endpoint in /actuator /actuator/env /actuator/heapdump /debug/pprof /.env /api-docs; do
  response=$(curl -s -o /dev/null -w "%{http_code}" https://api.mycompany.com$endpoint)
  echo "$endpoint: $response"
done

# Test BOLA: access another user's resource
# (Use test accounts in staging)
VICTIM_ID="user-12345"
curl -H "Authorization: Bearer $ATTACKER_TOKEN" \
  https://api.mycompany.com/api/v1/users/$VICTIM_ID/orders

# Test rate limiting
for i in {1..100}; do
  curl -s -o /dev/null -w "%{http_code}" \
    https://api.mycompany.com/api/v1/login \
    -X POST -d '{"email":"test@test.com","password":"wrong"}'
done | sort | uniq -c

# SSRF test (in staging only)
curl -X POST https://api.mycompany.com/api/v1/webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"callback_url": "http://169.254.169.254/latest/meta-data/"}'
```

---

## Security Considerations

- **Authorization is the hardest problem.** Authentication tells you who the user is. Authorization tells you what they can do with which objects. BOLA, BOPLA, and BFLA are all authorization failures — five of the OWASP API Top 10 are authorization issues. Invest in a centralized authorization layer (OPA, Cedar, Casbin) rather than scattering `if (user.role == "admin")` checks throughout the codebase.
- **WAFs are a complement, not a substitute.** WAFs block known malicious patterns but cannot detect BOLA (valid requests to wrong object) or BFLA (valid requests to wrong function). They are effective for SSRF blocking, rate limiting, and known vulnerability patterns.
- **Never log raw request/response bodies at INFO level in production.** API logs frequently capture authorization tokens, passwords in request bodies (login endpoints), and sensitive response data. Use structured logging with explicit field allowlists. Ensure PII fields (`email`, `ssn`, `credit_card`) are masked or excluded from logs.
- **SSRF is a cloud account takeover vulnerability.** In cloud environments, a successful SSRF to the metadata endpoint can yield IAM credentials with significant permissions. Treat any SSRF as a critical severity incident requiring immediate containment and IAM credential rotation.

---

## Interview Questions

### Basic

**Q: What is the difference between BOLA and BFLA?**
A: BOLA (Broken Object Level Authorization) is about accessing someone else's *data object* — user A accessing user B's orders, a customer accessing another customer's account. The function being called is authorized, but the specific data object is not. BFLA (Broken Function Level Authorization) is about accessing someone else's *functionality* — a regular user calling an admin endpoint that deletes users, a read-only API key calling a write endpoint. BOLA is a data access violation; BFLA is a capability violation.

**Q: How does IMDSv2 prevent SSRF attacks targeting AWS metadata?**
A: IMDSv1 responds to any GET request to `169.254.169.254` — including those made by an SSRF vulnerability. IMDSv2 requires a two-step flow: first, a PUT request with a `X-aws-ec2-metadata-token-ttl-seconds` header to obtain a session token, then a GET request with `X-aws-ec2-metadata-token` header containing that token. An SSRF vulnerability that only makes GET requests (fetching a URL the attacker provides) cannot complete the PUT step, so it cannot obtain the metadata service session token. The `http-put-response-hop-limit: 1` setting ensures the PUT cannot be made from inside a container to the host metadata service.

**Q: What is the JWT algorithm confusion attack?**
A: RSA-signed JWTs use an asymmetric key pair: the private key signs, the public key verifies. If the server accepts the algorithm from the token header rather than enforcing a specific expected algorithm, an attacker can change `alg` from `RS256` (asymmetric) to `HS256` (symmetric HMAC). A naive JWT library configured with the RSA public key will, when seeing `alg: HS256`, use the same key as the HMAC secret. Since the attacker knows the public key, they can forge a valid-looking HS256 signature. Prevention: always specify the expected algorithm at verification time and reject any token with a different algorithm.

### Intermediate

**Q: A BOLA vulnerability was found in production. How do you triage the blast radius?**
A: (1) **Determine exploitability:** Were object IDs sequential integers (easily enumerable) or UUIDs (random but still discoverable via other means)? Sequential = high probability of exploitation. (2) **Check access logs:** Query the last 90 days of API logs for requests where the authenticated user ID does not match the resource owner ID. This identifies whether the vulnerability was discovered and exploited. (3) **Identify affected objects:** What data was potentially accessible? PII? Payment data? HIPAA-covered? This determines notification obligations. (4) **Contain:** If evidence of exploitation exists, rotate any secrets or tokens that could have been exposed, and notify affected users per your breach notification policy. (5) **Fix immediately:** Add ownership check to every affected endpoint. Add an integration test that asserts cross-user access returns 403.

**Q: How would you implement defense-in-depth against credential stuffing on an API login endpoint?**
A: Multiple layers: (1) **Rate limiting** at the API gateway: 5 attempts per account per minute, 20 per IP per minute. Separate buckets for IP and account to prevent blocking legitimate users via IP rotation. (2) **Credential pair detection:** If the same `(email, password)` pair is tried against multiple accounts or from multiple IPs, it is likely a credential stuffing list — not a human typo. (3) **CAPTCHA** after 3 failed attempts from the same client fingerprint. (4) **Bot detection** at the WAF layer — Cloudflare Bot Management, AWS WAF Bot Control use TLS fingerprint and behavioral signals. (5) **Breached password detection:** Check submitted passwords against HaveIBeenPwned API during authentication — if the password appears in a breach list, require the user to change it even on successful authentication. (6) **Anomaly detection in SIEM:** Alert on login success rate drops (credential stuffing sometimes succeeds at 0.1-1% rate before being blocked) or sudden spikes in authentication latency.

### Advanced / Staff Level

**Q: Design an API security architecture for a financial services API handling 10,000 requests/second.**
A: (1) **Gateway layer:** API gateway (Kong/AWS API GW) handles: TLS termination, OAuth 2.0 token introspection/validation, rate limiting (per-user, per-IP, per-endpoint tiers), request size limits, mTLS for partner integrations, request/response logging with PII masking. (2) **WAF:** AWS WAF or Cloudflare in front of gateway — managed rules for OWASP Top 10, bot control, SSRF pattern detection, geofencing for regulatory compliance. (3) **AuthZ service:** Centralized OPA or Cedar service evaluating authorization. API handlers call the AuthZ service for every request — no inline permission checks. OPA bundle updates via OCI registry for policy-as-code deployment. In-process OPA SDKs for sub-millisecond evaluation at 10K RPS. (4) **Application layer:** All database queries parameterized. Response DTOs with explicit field allowlists. Input validation via JSON Schema with `additionalProperties: false`. All third-party API responses validated against strict schemas. (5) **Observability:** Every API call logged with user ID, resource ID, action, decision, and latency. SIEM rules for BOLA patterns (user accessing > 100 different object IDs in 1 minute), BFLA patterns (role changing on own account), and SSRF patterns (requests to RFC1918/169.254.x.x ranges). (6) **Continuous testing:** DAST (OWASP ZAP, Burp Suite) in CI/CD pipeline testing for BOLA, BFLA, injection. Dedicated API security testing sprint quarterly with bug bounty program for external researchers.
