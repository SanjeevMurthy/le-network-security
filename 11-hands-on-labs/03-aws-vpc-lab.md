# Lab 03: AWS VPC Networking

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Prerequisites](#prerequisites)
- [Debugging Methodology Alignment](#debugging-methodology-alignment)
- [Lab 3A: VPC + 3-Tier Subnet Design](#lab-3a-vpc-3-tier-subnet-design)
  - [Architecture](#architecture)
  - [Setup: Create VPC](#setup-create-vpc)
  - [Setup: Create Subnets](#setup-create-subnets)
  - [Setup: Internet Gateway + Elastic IP + NAT Gateway](#setup-internet-gateway-elastic-ip-nat-gateway)
  - [Setup: Route Tables](#setup-route-tables)
  - [Verify: Routing Behavior](#verify-routing-behavior)
- [Lab 3B: Security Group + NACL Debugging](#lab-3b-security-group-nacl-debugging)
  - [Background: SG vs NACL](#background-sg-vs-nacl)
  - [Setup: Create Security Groups](#setup-create-security-groups)
  - [The Break: NACL That Forgets Ephemeral Return Ports](#the-break-nacl-that-forgets-ephemeral-return-ports)
  - [Symptoms](#symptoms)
  - [Diagnosis: VPC Flow Logs](#diagnosis-vpc-flow-logs)
  - [Root Cause](#root-cause)
  - [Fix: Add the Missing Ephemeral Port Rules](#fix-add-the-missing-ephemeral-port-rules)
- [Lab 3C: VPC Peering Troubleshooting](#lab-3c-vpc-peering-troubleshooting)
  - [Setup: Create Two VPCs](#setup-create-two-vpcs)
  - [Setup: Create VPC Peering Connection](#setup-create-vpc-peering-connection)
  - [The Break: Add Route Only on One Side](#the-break-add-route-only-on-one-side)
  - [Symptoms](#symptoms)
  - [Diagnosis](#diagnosis)
  - [Root Cause](#root-cause)
  - [Fix: Add the Missing Route in VPC-B](#fix-add-the-missing-route-in-vpc-b)
  - [Verify](#verify)
  - [Cleanup](#cleanup)
- [Summary: AWS VPC Lab Checklist](#summary-aws-vpc-lab-checklist)
- [Interview Discussion Points](#interview-discussion-points)

---

## Learning Objectives

- Design and build a production 3-tier VPC from scratch using AWS CLI
- Understand the stateless nature of NACLs and reproduce the classic "forgot ephemeral ports" failure
- Debug asymmetric VPC peering failures caused by missing route table entries

## Prerequisites

- AWS CLI v2 configured with sufficient IAM permissions (EC2, VPC full access)
- An AWS sandbox account or localstack for cost-free testing
- Packages: `awscli`, `jq`

```bash
# Verify AWS CLI is configured
aws sts get-caller-identity
# Expected: JSON with Account, UserId, Arn

# Optional: run locally with localstack (free, no AWS account needed)
pip install localstack awscli-local
localstack start -d
alias aws='awslocal'   # redirect all commands to localstack
```

> **Cost note:** NAT Gateways are ~$0.045/hour + data charges. For practice, create and delete within the same session. Total cost for one full run of this lab: <$1.

## Debugging Methodology Alignment

These labs reinforce the layered debugging model from `07-debugging-playbooks/00-debugging-methodology.md`:
- Lab 3A maps to L2/L3 design: subnet architecture, routing table decisions
- Lab 3B maps to L3 (NACL = stateless packet filter) vs L4 (Security Groups = stateful)
- Lab 3C maps to L3 asymmetric routing — a classic "works one way, not the other" scenario

---

## Lab 3A: VPC + 3-Tier Subnet Design

**Objective:** Build a production-standard VPC with public, private, and isolated (data) subnets across 2 AZs using AWS CLI. Understand routing decisions at each tier.

---

### Architecture

```
VPC: 10.0.0.0/16
├── AZ us-east-1a
│   ├── public-1a:   10.0.1.0/24   → IGW (internet-routable)
│   ├── private-1a:  10.0.10.0/24  → NAT GW (outbound only)
│   └── isolated-1a: 10.0.20.0/24  → no internet (DB tier)
└── AZ us-east-1b
    ├── public-1b:   10.0.2.0/24   → IGW
    ├── private-1b:  10.0.11.0/24  → NAT GW
    └── isolated-1b: 10.0.21.0/24  → no internet
```

### Setup: Create VPC

```bash
# Set region
export AWS_DEFAULT_REGION=us-east-1

# Create VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc}]' \
  --query 'Vpc.VpcId' --output text)
echo "VPC: $VPC_ID"
# Expected: vpc-0123456789abcdef0

# Enable DNS hostnames (required for RDS, etc.)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
```

### Setup: Create Subnets

```bash
# Public subnets
PUBLIC_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]' \
  --query 'Subnet.SubnetId' --output text)

PUBLIC_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1b}]' \
  --query 'Subnet.SubnetId' --output text)

# Private subnets (app tier)
PRIVATE_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.10.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]' \
  --query 'Subnet.SubnetId' --output text)

PRIVATE_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1b}]' \
  --query 'Subnet.SubnetId' --output text)

# Isolated subnets (DB tier — no internet)
ISOLATED_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.20.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=isolated-1a}]' \
  --query 'Subnet.SubnetId' --output text)

ISOLATED_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID --cidr-block 10.0.21.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=isolated-1b}]' \
  --query 'Subnet.SubnetId' --output text)

echo "Public: $PUBLIC_1A, $PUBLIC_1B"
echo "Private: $PRIVATE_1A, $PRIVATE_1B"
echo "Isolated: $ISOLATED_1A, $ISOLATED_1B"
```

### Setup: Internet Gateway + Elastic IP + NAT Gateway

```bash
# Create and attach Internet Gateway (for public subnets)
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=lab-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
echo "IGW: $IGW_ID"

# Allocate Elastic IP for NAT Gateway
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc \
  --query 'AllocationId' --output text)
echo "EIP Allocation: $EIP_ALLOC"

# Create NAT Gateway in public-1a (NAT GW must be in a PUBLIC subnet)
NAT_GW=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_1A \
  --allocation-id $EIP_ALLOC \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=lab-nat}]' \
  --query 'NatGateway.NatGatewayId' --output text)
echo "NAT GW: $NAT_GW"

# Wait for NAT GW to become available (takes ~60-90 seconds)
echo "Waiting for NAT Gateway to become available..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW
echo "NAT Gateway ready"
```

### Setup: Route Tables

```bash
# Public Route Table: default route -> IGW
PUBLIC_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PUBLIC_RT \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# Associate public subnets with public route table
aws ec2 associate-route-table --subnet-id $PUBLIC_1A --route-table-id $PUBLIC_RT
aws ec2 associate-route-table --subnet-id $PUBLIC_1B --route-table-id $PUBLIC_RT

# Enable auto-assign public IP for public subnets
aws ec2 modify-subnet-attribute --subnet-id $PUBLIC_1A --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUBLIC_1B --map-public-ip-on-launch

# Private Route Table: default route -> NAT GW
PRIVATE_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PRIVATE_RT \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW

aws ec2 associate-route-table --subnet-id $PRIVATE_1A --route-table-id $PRIVATE_RT
aws ec2 associate-route-table --subnet-id $PRIVATE_1B --route-table-id $PRIVATE_RT

# Isolated Route Table: NO default route (no internet)
ISOLATED_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=isolated-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)
# Do NOT add 0.0.0.0/0 here — isolated subnets have no internet route

aws ec2 associate-route-table --subnet-id $ISOLATED_1A --route-table-id $ISOLATED_RT
aws ec2 associate-route-table --subnet-id $ISOLATED_1B --route-table-id $ISOLATED_RT
```

### Verify: Routing Behavior

```bash
# Verify public route table has IGW route
aws ec2 describe-route-tables --route-table-ids $PUBLIC_RT \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,GatewayId]' \
  --output table
# Expected:
# --------------------------------
# |     DestinationCidrBlock     |
# +--------------+---------------+
# |  0.0.0.0/0  |  igw-xxxxx   |
# |  10.0.0.0/16|  local        |

# Verify private route table has NAT GW route
aws ec2 describe-route-tables --route-table-ids $PRIVATE_RT \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,NatGatewayId]' \
  --output table
# Expected:
# --------------------------------
# |  0.0.0.0/0  |  nat-xxxxx   |

# Verify isolated route table has NO default route (only local)
aws ec2 describe-route-tables --route-table-ids $ISOLATED_RT \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,GatewayId]' \
  --output table
# Expected: ONLY 10.0.0.0/16 local — no 0.0.0.0/0 entry
```

**Key Takeaway:** The 3-tier model: public subnet has IGW route (bidirectional internet), private subnet has NAT GW route (outbound-only internet), isolated subnet has no default route (air-gapped, internal only). NAT GW must live in a PUBLIC subnet — a common mistake is placing it in the private subnet.

---

## Lab 3B: Security Group + NACL Debugging

**Objective:** Reproduce the "NACL blocks return traffic" failure caused by forgetting that NACLs are stateless — you must explicitly allow ephemeral port return traffic.

---

### Background: SG vs NACL

| Feature | Security Group | NACL |
|---------|---------------|------|
| State | Stateful (tracks connections) | Stateless (each packet evaluated) |
| Scope | Instance level | Subnet level |
| Rules | Allow only | Allow and Deny |
| Return traffic | Automatically allowed | Must explicitly allow |
| Order | All rules evaluated | Lowest rule number wins |

### Setup: Create Security Groups

```bash
# Create SG for public instance (bastion)
BASTION_SG=$(aws ec2 create-security-group \
  --group-name bastion-sg \
  --description "Bastion host SG" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Allow SSH inbound
aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

# Allow all outbound (default for SGs, but being explicit)
aws ec2 authorize-security-group-egress \
  --group-id $BASTION_SG \
  --protocol -1 --cidr 0.0.0.0/0

# Create SG for private instance (app server)
APPSERVER_SG=$(aws ec2 create-security-group \
  --group-name appserver-sg \
  --description "App server SG" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Allow SSH from bastion SG (SG-to-SG reference)
aws ec2 authorize-security-group-ingress \
  --group-id $APPSERVER_SG \
  --protocol tcp --port 22 \
  --source-group $BASTION_SG

echo "Bastion SG: $BASTION_SG"
echo "AppServer SG: $APPSERVER_SG"
```

### The Break: NACL That Forgets Ephemeral Return Ports

```bash
# Create a custom NACL for the private subnet
PRIVATE_NACL=$(aws ec2 create-network-acl \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=broken-private-nacl}]' \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Associate with private subnet 1a
NACL_ASSOC=$(aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=$PRIVATE_1A" \
  --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' --output text)

aws ec2 replace-network-acl-association \
  --association-id $NACL_ASSOC \
  --network-acl-id $PRIVATE_NACL

# Add INBOUND rules:
# Rule 100: Allow SSH (TCP 22) inbound from anywhere
aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL \
  --ingress --rule-number 100 \
  --protocol tcp --port-range From=22,To=22 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Rule 200: Allow established connections (ephemeral ports inbound — return traffic from outbound requests)
aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL \
  --ingress --rule-number 200 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Rule 32767 (default): DENY all
# (default deny is automatic in new NACLs)

# BUG: Add OUTBOUND rules — allow outbound SSH response BUT forget ephemeral ports
# This means: outbound connections (e.g., curl) will be initiated, but return packets
# (arriving on high-numbered source port) won't be allowed back through
aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL \
  --egress --rule-number 100 \
  --protocol tcp --port-range From=22,To=22 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# INTENTIONALLY MISSING: outbound rule for ephemeral ports 1024-65535
# Without this, when the private instance initiates a connection:
# - SYN goes out (egress allowed by... nothing, gets blocked)
# Actually: the instance's initiated connections use random source ports on the INBOUND return
# Let's simulate: private instance tries to curl the public instance on port 80
# The SYN packet has source=<random high port>, dest=<public instance>:80
# On return: source=<public instance>:80, dest=<random high port>
# The return packet arrives as INBOUND with dest port in ephemeral range
# Our INBOUND rule 200 allows ephemeral ports inbound ✓
# But the OUTBOUND egress from the private subnet is missing for high-dest-ports:
# private instance -> port 80 (web) egress: NOT ALLOWED unless we add rule

# Better break: allow inbound SSH (100) but forget outbound ephemeral port rule
# This simulates: SSH INTO the instance works, but SSH session hangs after login
# because the RETURN packets from the instance (ephemeral source port) are blocked

echo "Broken NACL ID: $PRIVATE_NACL"
echo "NACL configured with missing outbound ephemeral port rule"
```

### Symptoms

```bash
# Symptom: connection to private instance hangs — NOT "connection refused"
# Connection refused = SG or process issue (immediate RST)
# Connection hangs = NACL is silently dropping (no RST, no ICMP, just nothing)

# To simulate without an actual instance, describe what you'd observe:
cat <<'EOF'
Expected symptoms with broken NACL:
  1. SSH to private instance from bastion: hangs at "Connecting..."
  2. NOT "Connection refused" (that would be SG or process issue)
  3. No error message — just silence until timeout
  4. traceroute: last hop is within VPC, then nothing (no ICMP TTL exceeded back)

With "connection refused": fix the process or security group
With "connection hangs":   suspect NACL or host firewall dropping silently
EOF
```

### Diagnosis: VPC Flow Logs

```bash
# Enable VPC Flow Logs to S3 or CloudWatch Logs
# Flow logs show ACCEPT/REJECT per packet — the key diagnostic tool

# Create CloudWatch log group
aws logs create-log-group --log-group-name /aws/vpc/flowlogs

# Create IAM role for flow logs (simplified — in real env, use proper policy)
FLOW_LOG_ROLE=$(aws iam create-role \
  --role-name VPCFlowLogsRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"vpc-flow-logs.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  --query 'Role.Arn' --output text 2>/dev/null || \
  aws iam get-role --role-name VPCFlowLogsRole --query 'Role.Arn' --output text)

# Enable flow logs
aws ec2 create-flow-logs \
  --resource-ids $VPC_ID \
  --resource-type VPC \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn $FLOW_LOG_ROLE

# Query flow logs — look for REJECT entries on the private subnet
# In CloudWatch Insights:
cat <<'EOF'
# CloudWatch Log Insights query to find NACl REJECTs on private subnet:
fields @timestamp, srcAddr, dstAddr, srcPort, dstPort, protocol, action, logStatus
| filter action = "REJECT"
| filter (dstAddr like "10.0.10." or srcAddr like "10.0.10.")
| sort @timestamp desc
| limit 50

# Expected output for broken NACL (missing outbound ephemeral ports):
# timestamp  srcAddr      dstAddr      srcPort  dstPort  proto  action
# 10:01:23   10.0.10.5    10.0.1.10    443      52341    6      REJECT
# 10:01:23   10.0.10.5    10.0.1.10    443      52342    6      REJECT
# These REJECTs: private instance (10.0.10.5) sending SYN to public (10.0.1.10) on port 443
# The return packets are being REJECTED (NACL outbound rule missing for port 443)
EOF

# Inspect NACL rules to confirm what's missing
aws ec2 describe-network-acls --network-acl-ids $PRIVATE_NACL \
  --query 'NetworkAcls[0].Entries[*].[RuleNumber,Protocol,PortRange,Egress,RuleAction]' \
  --output table
# Expected: shows only rule 100 (SSH) for egress, no rule for ports 1024-65535
```

### Root Cause

NACLs are **stateless**. Unlike Security Groups, they do not track connection state. When a private instance initiates an outbound connection to port 80 or 443, the return packets arrive as inbound traffic on an ephemeral port (32768-60999 on Linux). If the NACL outbound rules don't allow the outgoing connection (or the inbound rules don't allow the return traffic), packets are silently dropped — the kernel sees no RST, so TCP times out.

### Fix: Add the Missing Ephemeral Port Rules

```bash
# Add outbound rule for ephemeral ports (covers HTTP/HTTPS return traffic)
aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL \
  --egress --rule-number 200 \
  --protocol tcp --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 --rule-action allow

aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL \
  --egress --rule-number 300 \
  --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# The comprehensive fix: allow all outbound TCP on ephemeral range
# This covers return traffic for ANY inbound connection
aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL \
  --egress --rule-number 400 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Also allow UDP ephemeral ports (for DNS returns)
aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL \
  --egress --rule-number 500 \
  --protocol udp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Verify updated NACL
aws ec2 describe-network-acls --network-acl-ids $PRIVATE_NACL \
  --query 'NetworkAcls[0].Entries[*].[RuleNumber,Egress,Protocol,PortRange,RuleAction]' \
  --output table
```

**Key Takeaway:** NACL "connection hangs" = missing ephemeral port rule. The diagnostic signal: VPC Flow Logs show REJECT (NACLs) vs no flow log entry at all (SG blocks — before the connection reaches the NACL). NACLs require ephemeral port rules (1024-65535) in both directions. Security Groups do not — they are stateful.

---

## Lab 3C: VPC Peering Troubleshooting

**Objective:** Create and break a VPC peering connection by forgetting route table entries on one side, reproducing asymmetric routing.

---

### Setup: Create Two VPCs

```bash
# VPC-A (already created above as $VPC_ID, CIDR 10.0.0.0/16)
VPC_A=$VPC_ID

# Create VPC-B with a different non-overlapping CIDR
VPC_B=$(aws ec2 create-vpc \
  --cidr-block 10.1.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=vpc-b}]' \
  --query 'Vpc.VpcId' --output text)

# Create a subnet in VPC-B
SUBNET_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_B --cidr-block 10.1.1.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)

echo "VPC-A: $VPC_A (10.0.0.0/16)"
echo "VPC-B: $VPC_B (10.1.0.0/16)"
```

### Setup: Create VPC Peering Connection

```bash
# Create peering request from VPC-A to VPC-B
PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $VPC_A \
  --peer-vpc-id $VPC_B \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=lab-peering}]' \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)
echo "Peering ID: $PEERING_ID"

# Accept the peering connection (in same account, must explicitly accept)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEERING_ID

# Verify peering is active
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEERING_ID \
  --query 'VpcPeeringConnections[0].Status.Code' --output text
# Expected: active
```

### The Break: Add Route Only on One Side

```bash
# Get VPC-B's main route table
ROUTE_TABLE_B=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_B" "Name=association.main,Values=true" \
  --query 'RouteTables[0].RouteTableId' --output text)

# Add route in VPC-A's private route table: to VPC-B via peering
aws ec2 create-route \
  --route-table-id $PRIVATE_RT \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID
echo "Route added in VPC-A: 10.1.0.0/16 -> peering"

# INTENTIONALLY DO NOT add the return route in VPC-B
# VPC-B has NO route back to VPC-A's CIDR
echo "VPC-B route table: NOT updated (the break)"
```

### Symptoms

```bash
# From an instance in VPC-A private subnet:
# ping 10.1.1.x   -> SUCCESS (packet reaches VPC-B)

# From an instance in VPC-B subnet:
# ping 10.0.10.x  -> FAILS (no return path from VPC-B to VPC-A)

cat <<'EOF'
Asymmetric routing symptoms:
  VPC-A instance -> VPC-B: WORKS (A has route to B via peering)
  VPC-B instance -> VPC-A: FAILS (B has no route to A)

  TCP from VPC-A to VPC-B:
  - SYN: A->B succeeds (route exists in A)
  - SYN-ACK: B->A FAILS (no route in B to reach 10.0.0.0/16)
  - Result: connection hangs at SYN (VPC-A keeps retransmitting SYN)

  Not a peering connection issue — the peering is ACTIVE
  This is a ROUTING issue — missing route table entry
EOF
```

### Diagnosis

```bash
# Step 1: Verify peering connection status (this is fine)
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEERING_ID \
  --query 'VpcPeeringConnections[0].[Status.Code,AccepterVpcInfo.CidrBlock,RequesterVpcInfo.CidrBlock]' \
  --output table
# Expected:
# +---------+--------------+--------------+
# | active  | 10.1.0.0/16  | 10.0.0.0/16  |

# Step 2: Check routes in VPC-A — should show peering route
aws ec2 describe-route-tables --route-table-ids $PRIVATE_RT \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId]' \
  --output table
# Expected:
# +-----------------+------------------------+
# | 10.1.0.0/16     | pcx-xxxxxxxxxxxxx      |  <-- route to VPC-B exists

# Step 3: Check routes in VPC-B — THIS IS THE PROBLEM
aws ec2 describe-route-tables --route-table-ids $ROUTE_TABLE_B \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId,GatewayId]' \
  --output table
# Expected:
# +-----------------+-------+--------+
# | 10.1.0.0/16     | None  | local  |   <-- only local route, no route to 10.0.0.0/16

# DIAGNOSIS: VPC-B has no route for 10.0.0.0/16 via the peering connection
# Return traffic from VPC-B to VPC-A is being blackholed

# Step 4: Confirm with VPC Flow Logs (if enabled on both VPCs)
# Look for ACCEPT on ingress to VPC-B but no corresponding traffic from VPC-B back
```

### Root Cause

VPC peering is not transitive routing — each VPC must independently have a route table entry pointing to the other VPC's CIDR via the peering connection. The peering connection being "active" only means the logical connection exists; it does not auto-propagate routes. Packets from VPC-A reach VPC-B because VPC-A has the route. Return packets from VPC-B have no route to VPC-A and are dropped.

### Fix: Add the Missing Route in VPC-B

```bash
# Add the return route in VPC-B's route table
aws ec2 create-route \
  --route-table-id $ROUTE_TABLE_B \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID

echo "Route added in VPC-B: 10.0.0.0/16 -> peering"

# Verify both sides now have routes
echo "=== VPC-A routes to VPC-B ==="
aws ec2 describe-route-tables --route-table-ids $PRIVATE_RT \
  --query 'RouteTables[0].Routes[?VpcPeeringConnectionId!=null].[DestinationCidrBlock,VpcPeeringConnectionId]' \
  --output table

echo "=== VPC-B routes to VPC-A ==="
aws ec2 describe-route-tables --route-table-ids $ROUTE_TABLE_B \
  --query 'RouteTables[0].Routes[?VpcPeeringConnectionId!=null].[DestinationCidrBlock,VpcPeeringConnectionId]' \
  --output table
# Both should now show routes to the other VPC's CIDR
```

### Verify

```bash
# With both routes in place, test bidirectional connectivity
# (Assuming instances exist in both VPCs with appropriate Security Groups)
# VPC-A to VPC-B: aws ec2 ... ssm-session or direct ping
# VPC-B to VPC-A: should now also work

# Quick sanity check on peering and routes:
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEERING_ID \
  --query 'VpcPeeringConnections[0].Status.Code' --output text
# Expected: active

# Check security groups allow ICMP between VPCs (if testing with ping)
```

### Cleanup

```bash
# Delete in reverse dependency order
aws ec2 delete-route --route-table-id $PRIVATE_RT --destination-cidr-block 10.1.0.0/16
aws ec2 delete-route --route-table-id $ROUTE_TABLE_B --destination-cidr-block 10.0.0.0/16
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $PEERING_ID
aws ec2 delete-subnet --subnet-id $SUBNET_B
aws ec2 delete-vpc --vpc-id $VPC_B

# Clean up lab VPC (partial — full cleanup requires deleting NAT GW, EIP, etc.)
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW
aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_GW
aws ec2 release-address --allocation-id $EIP_ALLOC
aws ec2 detach-internet-gateway --vpc-id $VPC_A --internet-gateway-id $IGW_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-route-table --route-table-id $PUBLIC_RT
aws ec2 delete-route-table --route-table-id $PRIVATE_RT
aws ec2 delete-route-table --route-table-id $ISOLATED_RT
for SUBNET in $PUBLIC_1A $PUBLIC_1B $PRIVATE_1A $PRIVATE_1B $ISOLATED_1A $ISOLATED_1B; do
  aws ec2 delete-subnet --subnet-id $SUBNET
done
aws ec2 delete-vpc --vpc-id $VPC_A
```

**Key Takeaway:** "Peering active but traffic only flows one way" = missing route table on one side. Always add routes in BOTH VPCs. If there are multiple route tables (e.g., private RT and public RT), add peering routes to each table that needs reachability. VPC peering is not symmetric by default.

---

## Summary: AWS VPC Lab Checklist

| Lab  | Core Skill | Production Scenario |
|------|-----------|---------------------|
| 3A   | 3-tier VPC design | Security review: "Is this DB reachable from internet?" |
| 3B   | NACL stateless debugging | "SSH works to bastion, but hangs to private instance" |
| 3C   | VPC peering asymmetric routing | "Service in VPC-A can't reach VPC-B even though peering is active" |

## Interview Discussion Points

1. "What's the difference between a Security Group and a NACL?" — Stateful vs stateless. SG = allow only, instance-level. NACL = allow/deny, subnet-level, must handle ephemeral ports.
2. "VPC peering is active but one side can't ping the other — what do you check?" — Route tables on both sides. Peering active != routing configured.
3. "How would you design VPC networking for a 3-tier web application?" — Answer is Lab 3A: public (ALB), private (app servers), isolated (RDS).
