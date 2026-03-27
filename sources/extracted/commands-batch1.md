# Network/Linux Commands Cheatsheet — Batch 1
# Extracted from: 00-networking-core/, 01-networking-fundamentals/, 02-linux-networking/, 03-cloud-networking/
# Format: TOOL / CMD / SOURCE / CONTEXT

---

## TOOL: arping

TOOL: arping
CMD: arping -I eth0 -A -c 3 10.0.0.100
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Send gratuitous ARP to announce IP ownership / after failover
---
TOOL: arping
CMD: arping -I eth0 -c 3 10.0.1.50
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Check for duplicate IP / ARP who-has probe
---

## TOOL: aws

TOOL: aws
CMD: aws ec2 describe-vpcs
SOURCE: 01-networking-fundamentals/02-ip-addressing-subnetting.md
CONTEXT: List all VPCs and their CIDR blocks
---
TOOL: aws
CMD: aws ec2 describe-subnets
SOURCE: 01-networking-fundamentals/02-ip-addressing-subnetting.md
CONTEXT: List subnets and available IP counts
---
TOOL: aws
CMD: aws ec2 create-vpc-peering-connection
SOURCE: 01-networking-fundamentals/02-ip-addressing-subnetting.md
CONTEXT: Create VPC peering connection between two VPCs
---
TOOL: aws
CMD: aws ec2 describe-vpn-connections --query ...
SOURCE: 01-networking-fundamentals/05-routing-protocols.md
CONTEXT: Check VPN connection status and BGP state
---
TOOL: aws
CMD: aws cloudwatch get-metric-statistics --namespace AWS/NatGateway ...
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Monitor NAT Gateway connection/port metrics for exhaustion
---
TOOL: aws
CMD: aws route53 list-hosted-zones-by-vpc ...
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: List Route53 private hosted zones associated with a VPC
---
TOOL: aws
CMD: aws route53 change-resource-record-sets ...
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Create/update Route53 DNS records
---
TOOL: aws
CMD: aws lambda get-function-configuration --function-name my-function --query 'VpcConfig'
SOURCE: 03-cloud-networking/aws/01-vpc-fundamentals.md
CONTEXT: Check Lambda VPC attachment (subnet, security group)
---
TOOL: aws
CMD: aws ec2 describe-security-groups --group-ids sg-lambda-xxx --query 'SecurityGroups[].IpPermissionsEgress'
SOURCE: 03-cloud-networking/aws/01-vpc-fundamentals.md
CONTEXT: Check Lambda SG outbound rules for RDS port
---
TOOL: aws
CMD: aws ec2 describe-security-groups --group-ids sg-rds-xxx --query 'SecurityGroups[].IpPermissions'
SOURCE: 03-cloud-networking/aws/01-vpc-fundamentals.md
CONTEXT: Check RDS SG inbound rules for Lambda SG reference
---
TOOL: aws
CMD: aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=subnet-isolated-a
SOURCE: 03-cloud-networking/aws/01-vpc-fundamentals.md
CONTEXT: Check NACL rules on isolated subnet for ephemeral port allowances
---
TOOL: aws
CMD: aws ec2 create-network-insights-path --source eni-lambda-xxx --destination eni-rds-xxx --protocol tcp --destination-port 5432
SOURCE: 03-cloud-networking/aws/01-vpc-fundamentals.md
CONTEXT: Create VPC Reachability Analyzer path for connectivity debugging
---
TOOL: aws
CMD: aws ec2 start-network-insights-analysis --network-insights-path-id nip-xxx
SOURCE: 03-cloud-networking/aws/01-vpc-fundamentals.md
CONTEXT: Run reachability analysis; output identifies blocking SG/NACL rule
---
TOOL: aws
CMD: aws directconnect describe-connections --connection-id dxcon-xxx
SOURCE: 03-cloud-networking/aws/02-vpc-connectivity.md
CONTEXT: Check Direct Connect connection state (up/down)
---
TOOL: aws
CMD: aws directconnect describe-virtual-interfaces --virtual-interface-id dxvif-xxx
SOURCE: 03-cloud-networking/aws/02-vpc-connectivity.md
CONTEXT: Check DX virtual interface BGP peer status
---
TOOL: aws
CMD: aws ec2 describe-vpn-connections --vpn-connection-id vpn-xxx --query 'VpnConnections[].VgwTelemetry'
SOURCE: 03-cloud-networking/aws/02-vpc-connectivity.md
CONTEXT: Check VPN tunnel status and last status change
---
TOOL: aws
CMD: aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id tgw-rtb-xxx --filters Name=type,Values=propagated
SOURCE: 03-cloud-networking/aws/02-vpc-connectivity.md
CONTEXT: Check TGW route table for on-premises prefix propagation
---
TOOL: aws
CMD: aws ec2 describe-route-tables --route-table-id rtb-xxx --query 'RouteTables[].PropagatingVgws'
SOURCE: 03-cloud-networking/aws/02-vpc-connectivity.md
CONTEXT: Verify VGW route propagation is enabled on VPC route table
---
TOOL: aws
CMD: aws ec2 enable-vgw-route-propagation --route-table-id rtb-xxx --gateway-id vgw-xxx
SOURCE: 03-cloud-networking/aws/02-vpc-connectivity.md
CONTEXT: Enable VGW route propagation so VPN routes appear in route table
---
TOOL: aws
CMD: aws elbv2 modify-target-group-attributes --target-group-arn arn:aws:elasticloadbalancing:... --attributes Key=deregistration_delay.timeout_seconds,Value=30
SOURCE: 03-cloud-networking/aws/03-aws-load-balancers.md
CONTEXT: Reduce deregistration delay to 30s for fast-response API target group
---
TOOL: aws
CMD: aws wafv2 associate-web-acl --web-acl-arn arn:aws:wafv2:... --resource-arn arn:aws:elasticloadbalancing:...
SOURCE: 03-cloud-networking/aws/03-aws-load-balancers.md
CONTEXT: Associate WAF Web ACL with ALB
---
TOOL: aws
CMD: aws elbv2 modify-load-balancer-attributes --load-balancer-arn arn:... --attributes Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=my-alb-logs Key=access_logs.s3.prefix,Value=prod-alb
SOURCE: 03-cloud-networking/aws/03-aws-load-balancers.md
CONTEXT: Enable ALB access logs to S3 for 504/502 debugging
---
TOOL: aws
CMD: aws elbv2 describe-load-balancer-attributes --load-balancer-arn arn:... --query 'Attributes[?Key==`idle_timeout.timeout_seconds`]'
SOURCE: 03-cloud-networking/aws/03-aws-load-balancers.md
CONTEXT: Check ALB idle timeout setting (default 60s; causes 504 on long requests)
---
TOOL: aws
CMD: aws elbv2 modify-load-balancer-attributes --load-balancer-arn arn:... --attributes Key=idle_timeout.timeout_seconds,Value=300
SOURCE: 03-cloud-networking/aws/03-aws-load-balancers.md
CONTEXT: Increase ALB idle timeout to 300s for long-running API requests
---
TOOL: aws
CMD: aws route53 associate-vpc-with-hosted-zone --hosted-zone-id Z1234567890ABC --vpc VPCRegion=us-east-1,VPCId=vpc-xxxxx
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Associate a Route53 private hosted zone with a VPC
---
TOOL: aws
CMD: aws route53resolver create-resolver-endpoint --creator-request-id unique-$(date +%s) --direction OUTBOUND --ip-addresses SubnetId=subnet-a,Ip=10.0.1.100 SubnetId=subnet-b,Ip=10.0.2.100 --security-group-ids sg-resolver-xxx
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Create Route53 Resolver outbound endpoint for hybrid DNS forwarding
---
TOOL: aws
CMD: aws route53resolver create-resolver-rule --rule-type FORWARD --domain-name corp.example.com --resolver-endpoint-id rslvr-out-xxx --target-ips Ip=192.168.1.53,Port=53 Ip=192.168.2.53,Port=53
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Create forwarding rule for on-premises domain to on-prem DNS servers
---
TOOL: aws
CMD: aws route53resolver create-firewall-domain-list --name malicious-domains --domains '["malware.example.com", "c2.attacker.net"]'
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Create DNS Firewall domain blocklist for malicious domains
---
TOOL: aws
CMD: aws route53resolver create-firewall-rule-group --name production-firewall
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Create Route53 Resolver DNS Firewall rule group
---
TOOL: aws
CMD: aws route53resolver create-firewall-rule --firewall-rule-group-id rslvr-frg-xxx --firewall-domain-list-id rslvr-fdl-xxx --priority 100 --action BLOCK --block-response NXDOMAIN
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Block DNS queries to malicious domains with NXDOMAIN response
---
TOOL: aws
CMD: aws route53resolver associate-firewall-rule-group --firewall-rule-group-id rslvr-frg-xxx --vpc-id vpc-xxx --priority 100 --name production
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Associate DNS Firewall rule group with a VPC
---
TOOL: aws
CMD: aws route53 enable-hosted-zone-dnssec --hosted-zone-id Z1234567890ABC --signing-status SIGNING
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Enable DNSSEC signing on a Route53 hosted zone
---
TOOL: aws
CMD: aws route53 create-key-signing-key --hosted-zone-id Z1234567890ABC --key-management-service-arn arn:aws:kms:us-east-1:123456:key/xxx --name production-ksk --status ACTIVE
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Create KSK for DNSSEC using a KMS CMK
---
TOOL: aws
CMD: aws route53 get-dnssec --hosted-zone-id Z1234567890ABC
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Get DS records to publish at domain registrar for DNSSEC chain of trust
---
TOOL: aws
CMD: aws route53 create-health-check --caller-reference unique-string-$(date +%s) --health-check-config '{"Type":"HTTPS","FullyQualifiedDomainName":"api.example.com","ResourcePath":"/health","Port":443,"RequestInterval":10,"FailureThreshold":3,"MeasureLatency":true,"Regions":["us-east-1","eu-west-1","ap-southeast-1"]}'
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Create Route53 health check with fast interval and multi-region probing
---
TOOL: aws
CMD: aws route53 list-health-checks --query 'HealthChecks[*].[Id,HealthCheckConfig.FullyQualifiedDomainName,HealthCheckConfig.Type]'
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: List all Route53 health checks and their configuration
---
TOOL: aws
CMD: aws route53 get-health-check-status --health-check-id abc123
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Get current health check status from all probe regions
---
TOOL: aws
CMD: aws route53 get-health-check --health-check-id abc123 --query 'HealthCheck.HealthCheckConfig'
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Check health check config for wrong path/port misconfiguration
---
TOOL: aws
CMD: aws route53 get-checker-ip-ranges --query 'CheckerIpRanges'
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Get Route53 health checker IP ranges to whitelist in WAF/SG
---
TOOL: aws
CMD: aws ec2 describe-security-groups --group-ids sg-alb-xxx --query 'SecurityGroups[].IpPermissions'
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Verify health checker IPs are not blocked by ALB security group
---
TOOL: aws
CMD: aws eks update-cluster-config --name production-cluster --resources-vpc-config endpointPublicAccess=true,endpointPrivateAccess=true,publicAccessCidrs=203.0.113.0/24,198.51.100.0/24
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Restrict EKS public API endpoint to specific CIDRs
---
TOOL: aws
CMD: aws ec2 describe-subnets --filters Name=tag:kubernetes.io/cluster/production,Values=shared --query 'Subnets[*].[SubnetId,CidrBlock,AvailableIpAddressCount]'
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Check available IPs in EKS node subnets for IP exhaustion diagnosis
---
TOOL: aws
CMD: aws network express-route show -g prod-rg -n prod-er-circuit --query "circuitProvisioningState"
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: Check ExpressRoute circuit provisioning/operational state
---

## TOOL: az

TOOL: az
CMD: az network vnet update --resource-group prod-rg --name prod-vnet --add addressSpace.addressPrefixes "172.20.0.0/16"
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Add a second CIDR block to an existing Azure VNet
---
TOOL: az
CMD: az network asg create -g prod-rg -n asg-web-tier
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Create Application Security Group for web tier
---
TOOL: az
CMD: az network asg create -g prod-rg -n asg-db-tier
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Create Application Security Group for DB tier
---
TOOL: az
CMD: az network nic ip-config update --resource-group prod-rg --nic-name web-vm-nic --name ipconfig1 --application-security-groups asg-web-tier
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Associate a NIC with an ASG for role-based NSG rules
---
TOOL: az
CMD: az network nsg rule create --resource-group prod-rg --nsg-name prod-nsg --name AllowWebToDb --priority 200 --source-asgs asg-web-tier --destination-asgs asg-db-tier --destination-port-ranges 5432 --protocol Tcp --access Allow
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Create NSG rule using ASGs instead of CIDRs
---
TOOL: az
CMD: az network route-table route create --resource-group prod-rg --route-table-name spoke-udr --name default-to-firewall --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.0.4
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: UDR to force all traffic through Azure Firewall in hub
---
TOOL: az
CMD: az network private-dns zone create -g prod-rg -n "internal.contoso.com"
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Create Azure Private DNS Zone for internal name resolution
---
TOOL: az
CMD: az network private-dns link vnet create -g prod-rg -n prod-vnet-link -z "internal.contoso.com" -v prod-vnet --registration-enabled true
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Link Private DNS Zone to VNet with autoregistration enabled
---
TOOL: az
CMD: az network vnet subnet update -g prod-rg -n app-subnet -vnet-name prod-vnet --service-endpoints Microsoft.Storage
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Enable Service Endpoint for Azure Storage on a subnet
---
TOOL: az
CMD: az storage account network-rule add -g prod-rg -n prodstorage --vnet-name prod-vnet --subnet app-subnet
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Allow subnet access to storage account via Service Endpoint
---
TOOL: az
CMD: az network private-endpoint create -g prod-rg -n pe-prodstorage --vnet-name prod-vnet --subnet pe-subnet --private-connection-resource-id /subscriptions/.../storageAccounts/prodstorage --group-id blob --connection-name storage-pe-conn
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Create Private Endpoint for storage account (stronger isolation than Service Endpoint)
---
TOOL: az
CMD: az network watcher test-ip-flow --vm prod-vm --direction Inbound --protocol TCP --local 10.0.1.10:443 --remote 1.2.3.4:52000 --resource-group prod-rg
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Azure Network Watcher IP Flow Verify — check if NSG blocks specific traffic
---
TOOL: az
CMD: az network nic show-effective-route-table --resource-group prod-rg --name prod-vm-nic --output table
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Show effective routes (system + UDR) for a NIC — check for unexpected blackholes
---
TOOL: az
CMD: az network nic list-effective-nsg --resource-group prod-rg --name prod-vm-nic
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Show merged NSG rules (subnet + NIC level) in priority order
---
TOOL: az
CMD: az network watcher flow-log create --resource-group prod-rg --name prod-nsg-flowlog --nsg prod-nsg ...
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Enable NSG flow logs for traffic visibility and forensics
---
TOOL: az
CMD: az network vnet peering create --resource-group prod-rg --name hub-to-spoke --vnet-name hub-vnet --remote-vnet spoke-vnet --allow-gateway-transit true
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: Create hub VNet peering with gateway transit enabled
---
TOOL: az
CMD: az network vnet peering create --resource-group prod-rg --name spoke-to-hub --vnet-name spoke-vnet --remote-vnet hub-vnet --use-remote-gateways true
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: Create spoke VNet peering using hub's remote gateways for on-prem access
---
TOOL: az
CMD: az network vnet peering list --resource-group prod-rg --vnet-name hub-vnet -o table
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: List VNet peering state — both sides must show Connected
---
TOOL: az
CMD: az network vnet-gateway create --resource-group prod-rg --name prod-vpn-gw --public-ip-address vpn-pip1 vpn-pip2 --vnet hub-vnet --gateway-type Vpn --vpn-type RouteBased --sku VpnGw2AZ --asn 65001 --active-active true
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: Create active-active zone-redundant VPN Gateway with BGP
---
TOOL: az
CMD: az network routeserver create --resource-group prod-rg --name prod-route-server --hosted-subnet /subscriptions/.../subnets/RouteServerSubnet --public-ip-address rs-pip
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: Deploy Azure Route Server for dynamic BGP route injection into VNet
---
TOOL: az
CMD: az network routeserver peering create --resource-group prod-rg --routeserver prod-route-server --name nva-peer --peer-asn 65002 --peer-ip 10.0.0.5
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: Create BGP peering between Azure Route Server and NVA
---
TOOL: az
CMD: az network route-table create -g prod-rg -n spoke1-udr
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: Create route table for spoke subnet UDRs
---
TOOL: az
CMD: az network route-table route create -g prod-rg --route-table-name spoke1-udr -n default-to-firewall --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.4
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: Route all spoke traffic via Azure Firewall in hub
---
TOOL: az
CMD: az network express-route show -g prod-rg -n prod-er-circuit --query "circuitProvisioningState"
SOURCE: 03-cloud-networking/azure/02-azure-connectivity.md
CONTEXT: Check ExpressRoute circuit provisioning state during outage
---
TOOL: az
CMD: az network lb rule create -g prod-rg --lb-name internal-lb -n ha-ports-rule --protocol All --frontend-port 0 --backend-port 0 --frontend-ip-name frontend --backend-pool-name nva-pool --probe-name nva-probe
SOURCE: 03-cloud-networking/azure/03-azure-load-balancers.md
CONTEXT: Create HA ports rule on internal LB for NVA (passes all protocols)
---
TOOL: az
CMD: az network lb outbound-rule create -g prod-rg --lb-name prod-lb -n outbound-rule --address-pool backend-pool --frontend-ip-configs pip1-frontend pip2-frontend pip3-frontend --protocol All --idle-timeout 15 --outbound-ports 10000
SOURCE: 03-cloud-networking/azure/03-azure-load-balancers.md
CONTEXT: Configure outbound SNAT rule with multiple PIPs to increase SNAT port count
---
TOOL: az
CMD: az network application-gateway waf-policy create -g prod-rg -n prod-waf-policy --location eastus
SOURCE: 03-cloud-networking/azure/03-azure-load-balancers.md
CONTEXT: Create WAF policy for Application Gateway
---
TOOL: az
CMD: az network application-gateway waf-policy policy-setting update -g prod-rg --policy-name prod-waf-policy --mode Prevention --state Enabled --request-body-check true --max-request-body-size 128 --file-upload-limit-in-mb 100
SOURCE: 03-cloud-networking/azure/03-azure-load-balancers.md
CONTEXT: Set Application Gateway WAF to Prevention mode with body inspection
---
TOOL: az
CMD: az network application-gateway create -g prod-rg -n prod-appgw --sku WAF_v2 --capacity 2 --max-capacity 10 --zones 1 2 3
SOURCE: 03-cloud-networking/azure/03-azure-load-balancers.md
CONTEXT: Create zone-redundant autoscaling Application Gateway v2 with WAF
---
TOOL: az
CMD: az network application-gateway probe create -g prod-rg --gateway-name prod-appgw -n custom-health-probe --protocol Https --host-name-from-http-settings true --path /health --interval 30 --timeout 30 --threshold 3 --match-status-codes "200-399"
SOURCE: 03-cloud-networking/azure/03-azure-load-balancers.md
CONTEXT: Create custom health probe for AppGW to fix 502 from default probe misconfiguration
---
TOOL: az
CMD: az afd origin-group create -g prod-rg --profile-name prod-afd --origin-group-name prod-origins --probe-request-type GET --probe-protocol Https --probe-interval-in-seconds 30 --probe-path /health --sample-size 4 --successful-samples-required 3 --additional-latency-in-milliseconds 50
SOURCE: 03-cloud-networking/azure/03-azure-load-balancers.md
CONTEXT: Create Azure Front Door origin group with health probing
---
TOOL: az
CMD: az afd origin create -g prod-rg --profile-name prod-afd --origin-group-name prod-origins --origin-name eastus-origin --host-name api-eastus.contoso.com --priority 1 --weight 1000
SOURCE: 03-cloud-networking/azure/03-azure-load-balancers.md
CONTEXT: Add origin to Front Door origin group
---
TOOL: az
CMD: az network application-gateway show-backend-health -g prod-rg -n prod-appgw --query "backendAddressPools[?name=='api-backend-pool'].backendHttpSettingsCollection[].servers[]"
SOURCE: 03-cloud-networking/azure/03-azure-load-balancers.md
CONTEXT: Check AppGW backend health — identify unhealthy targets causing 502 errors
---
TOOL: az
CMD: az aks create -g prod-rg -n prod-cluster --network-plugin azure --network-policy calico --network-plugin-mode overlay
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Create AKS cluster with Azure CNI Overlay and Calico network policy
---
TOOL: az
CMD: az aks create -g prod-rg -n prod-cluster --network-plugin azure --network-dataplane cilium --network-plugin-mode overlay
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Create AKS cluster with Azure CNI Overlay and Cilium eBPF dataplane
---
TOOL: az
CMD: az aks enable-addons -g prod-rg -n prod-cluster --addons ingress-appgw --appgw-name prod-appgw --appgw-subnet-cidr 10.0.5.0/24
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Enable AGIC add-on on existing AKS cluster
---
TOOL: az
CMD: az network nat gateway create -g prod-rg -n aks-nat-gw --public-ip-addresses aks-nat-pip --idle-timeout 10
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Create NAT Gateway for AKS node outbound traffic with predictable IPs
---
TOOL: az
CMD: az network vnet subnet update -g prod-rg -n aks-nodes-subnet --vnet-name prod-vnet --nat-gateway aks-nat-gw
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Attach NAT Gateway to AKS node subnet for outbound SNAT
---
TOOL: az
CMD: az aks create -g prod-rg -n prod-cluster --enable-private-cluster --private-dns-zone system
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Create private AKS cluster with API server accessible only via Private Endpoint
---
TOOL: az
CMD: az network vnet subnet show -g MC_prod-rg_prod-cluster_eastus -n aks-subnet --vnet-name prod-vnet --query "ipConfigurations | length(@)"
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Check IP allocation count in AKS subnet (Azure CNI IP exhaustion diagnosis)
---
TOOL: az
CMD: az aks create -g prod-rg -n prod-cluster-v2 --network-plugin azure --network-plugin-mode overlay --pod-cidr 10.244.0.0/12
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Create new AKS cluster with Azure CNI Overlay (migration from Azure CNI to fix IP exhaustion)
---
TOOL: az
CMD: az network vnet subnet update -g prod-rg -n aks-gpu-subnet -vnet-name prod-vnet --service-endpoints Microsoft.Storage
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Enable storage service endpoint on new AKS GPU node pool subnet
---
TOOL: az
CMD: az storage account network-rule add -g prod-rg -n prodstorage --vnet-name prod-vnet --subnet aks-gpu-subnet
SOURCE: 03-cloud-networking/azure/01-vnet-fundamentals.md
CONTEXT: Add GPU node pool subnet to storage account firewall rules
---

## TOOL: bpftool

TOOL: bpftool
CMD: bpftool prog load my_xdp.o /sys/fs/bpf/my_xdp type xdp 2>&1
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Load XDP program and view verifier output on failure
---
TOOL: bpftool
CMD: bpftool prog show
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: List all loaded BPF programs with IDs, tags, and load times
---
TOOL: bpftool
CMD: bpftool prog show id 42
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Show details of a specific loaded BPF program
---
TOOL: bpftool
CMD: bpftool prog dump xlated id 42
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Dump BPF bytecode instructions for a program
---
TOOL: bpftool
CMD: bpftool prog dump jited id 42
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Dump JIT-compiled x86 machine code for a BPF program
---
TOOL: bpftool
CMD: bpftool map show id 7
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Show BPF map details (type, key/value size, max entries)
---
TOOL: bpftool
CMD: bpftool map dump id 7 | head -50
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Dump BPF map entries — identify falsely blocked IPs in XDP blocklist
---
TOOL: bpftool
CMD: bpftool map delete id 7 key hex ...
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Remove a false positive entry from XDP blocklist map
---
TOOL: bpftool
CMD: bpftool map list | grep cilium
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: List BPF maps owned by Cilium CNI
---
TOOL: bpftool
CMD: bpftool prog list | grep cilium
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: List BPF programs loaded by Cilium
---
TOOL: bpftool
CMD: bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Generate vmlinux.h from kernel BTF for CO-RE BPF program development
---

## TOOL: bpftrace

TOOL: bpftrace
CMD: bpftrace -e 'kprobe:tcp_v4_connect { printf("TCP connect: %s → %s:%d\n", comm, ntop(AF_INET, args->uaddr->sa_data[2]), ((args->uaddr->sa_data[0] & 0xFF) << 8) | (args->uaddr->sa_data[1] & 0xFF)); }'
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Trace all new TCP connections with process name, dst IP and port
---
TOOL: bpftrace
CMD: bpftrace -e 'kprobe:tcp_retransmit_skb { $sk = (struct sock *)arg0; printf("RETRANSMIT: %s:%d → %d\n", ntop(2, $sk->__sk_common.skc_rcv_saddr), $sk->__sk_common.skc_num, $sk->__sk_common.skc_dport >> 8 | $sk->__sk_common.skc_dport << 8); }'
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Trace TCP retransmissions to detect packet loss in real time
---
TOOL: bpftrace
CMD: bpftrace -e 'kprobe:kfree_skb { @drops[kstack(5)] = count(); } END { print(@drops); }'
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Count packets dropped by kfree_skb grouped by kernel stack trace
---
TOOL: bpftrace
CMD: bpftrace -e 'kprobe:tcp_sendmsg { @send[comm] = sum(arg2); } kprobe:tcp_recvmsg { @recv[comm] = sum(arg2); } END { print(@send); print(@recv); }'
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Track per-process TCP send/receive byte totals
---

## TOOL: bridge

TOOL: bridge
CMD: bridge vlan show
SOURCE: 00-networking-core/01-internet-fundamentals.md
CONTEXT: Show VLAN assignments on bridge ports
---
TOOL: bridge
CMD: bridge fdb show
SOURCE: 00-networking-core/01-internet-fundamentals.md
CONTEXT: Show forwarding database (MAC address table) on bridge
---
TOOL: bridge
CMD: bridge fdb show dev flannel.1
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Show VTEP MAC/IP entries for Flannel VXLAN interface
---
TOOL: bridge
CMD: bridge fdb show dev docker0
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Show MAC addresses on Docker bridge interface
---
TOOL: bridge
CMD: bridge link show
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Show bridge port states and STP information
---

## TOOL: cilium

TOOL: cilium
CMD: cilium bpf policy list
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: List Cilium eBPF network policy maps
---
TOOL: cilium
CMD: cilium bpf lb list
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Show Cilium service load balancing BPF map entries
---
TOOL: cilium
CMD: cilium bpf ct list global
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Show Cilium connection tracking map (replaces conntrack for Cilium-managed traffic)
---
TOOL: cilium
CMD: cilium bpf endpoint list
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Show Cilium endpoint identity map
---
TOOL: cilium
CMD: cilium monitor --type drop
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Show packets dropped by Cilium NetworkPolicy in real time
---

## TOOL: conntrack

TOOL: conntrack
CMD: conntrack -L
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: List all current conntrack table entries
---
TOOL: conntrack
CMD: conntrack -E
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Watch conntrack events in real time (connections created/destroyed)
---
TOOL: conntrack
CMD: conntrack -L | grep "10.96.100.50"
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Check conntrack entries for a specific service ClusterIP
---
TOOL: conntrack
CMD: conntrack -S | grep insert_failed
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Check conntrack insert_failed counter — smoking gun for Kubernetes DNS 5s timeout
---

## TOOL: crictl

TOOL: crictl
CMD: crictl ps | grep mypod
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: List containers in a pod to get container ID for nsenter
---
TOOL: crictl
CMD: crictl inspect $CONTAINER_ID
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Get container PID for nsenter-based namespace debugging
---

## TOOL: curl

TOOL: curl
CMD: curl ifconfig.me -s
SOURCE: 00-networking-core/01-internet-fundamentals.md
CONTEXT: Check public IP address of current host
---
TOOL: curl
CMD: curl -I https://example.com
SOURCE: 00-networking-core/02-application-layer.md
CONTEXT: HTTP HEAD request — check response headers and status code
---
TOOL: curl
CMD: curl -v https://example.com
SOURCE: 00-networking-core/02-application-layer.md
CONTEXT: Verbose HTTP request — show TLS handshake, headers, and response
---
TOOL: curl
CMD: curl -s -o /dev/null -w "%{http_code}\n" https://example.com
SOURCE: 00-networking-core/02-application-layer.md
CONTEXT: Check HTTP status code only (silent mode, output code)
---
TOOL: curl
CMD: curl -v https://api.example.com/health
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Verbose HTTPS health check including TLS and HTTP layers
---
TOOL: curl
CMD: curl --http2 -v https://example.com
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Force HTTP/2 and show negotiation (ALPN h2)
---
TOOL: curl
CMD: curl -H "accept: application/dns-json" "https://cloudflare-dns.com/dns-query?name=example.com&type=A"
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: DNS-over-HTTPS (DoH) query using Cloudflare JSON API
---
TOOL: curl
CMD: curl -v https://api.example.com/health --resolve api.example.com:443:54.1.2.3
SOURCE: 03-cloud-networking/aws/04-route53-dns.md
CONTEXT: Test Route53 health check endpoint directly by overriding DNS resolution
---

## TOOL: dig

TOOL: dig
CMD: dig @8.8.8.8 example.com A +stats
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: DNS query with query time and server stats
---
TOOL: dig
CMD: dig +dnssec example.com A
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Query with DNSSEC validation — check RRSIG presence
---
TOOL: dig
CMD: dig @8.8.8.8 +dnssec broken-dnssec.example.com A
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Test DNSSEC validation — expect SERVFAIL if signatures invalid
---
TOOL: dig
CMD: dig @8.8.8.8 +cd broken-dnssec.example.com A
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Query with checking disabled (+cd) to bypass DNSSEC validation
---
TOOL: dig
CMD: dig @169.254.169.253 api.example.com
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Query AWS VPC resolver directly to test private hosted zone resolution
---
TOOL: dig
CMD: dig @8.8.8.8 api.company.com A +short
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Check public DNS resolution vs private (split-horizon debugging)
---
TOOL: dig
CMD: dig @10.0.0.2 api.company.com A +short
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Check private DNS resolver resolution for split-horizon comparison
---
TOOL: dig
CMD: dig example.com A +trace
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Trace full DNS resolution path from root to authoritative NS
---
TOOL: dig
CMD: dig example.com SOA
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Check zone SOA record — serial, refresh, retry, expiry TTLs
---
TOOL: dig
CMD: dig +dnssec example.com DNSKEY
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Check DNSKEY records for DNSSEC key material
---
TOOL: dig
CMD: dig +cd +dnssec example.com A
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Retrieve DNSSEC-signed answer with checking disabled
---
TOOL: dig
CMD: dig example.com CAA
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Check CAA records restricting which CAs can issue certificates
---
TOOL: dig
CMD: dig postgres.database.svc.cluster.local @10.96.0.10
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Test Kubernetes service DNS resolution inside pod network namespace
---

## TOOL: dmesg

TOOL: dmesg
CMD: dmesg | grep -i "eth0\|link\|carrier"
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Check kernel log for NIC link state changes
---
TOOL: dmesg
CMD: dmesg | grep -i "nf_conntrack: table full"
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Detect conntrack table full events causing dropped connections
---

## TOOL: docker

TOOL: docker
CMD: docker network inspect bridge | grep Subnet
SOURCE: 01-networking-fundamentals/02-ip-addressing-subnetting.md
CONTEXT: Show Docker bridge network subnet CIDR
---
TOOL: docker
CMD: docker inspect --format '{{.State.Pid}}' <container>
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Get container PID for nsenter to enter its network namespace
---
TOOL: docker
CMD: docker inspect --format '{{.State.Pid}}' <container_name>
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Get container PID for nsenter network namespace debugging
---

## TOOL: ethtool

TOOL: ethtool
CMD: ethtool eth0
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Show NIC speed, duplex, link status
---
TOOL: ethtool
CMD: ethtool -S eth0 | grep -E '(rx_missed|rx_no_buffer|rx_dropped)'
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Check NIC hardware counters for dropped/missed packets
---
TOOL: ethtool
CMD: ethtool -G eth0 rx 4096
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Increase NIC RX ring buffer to 4096 to absorb bursts
---
TOOL: ethtool
CMD: ethtool -g eth0
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Show current and maximum NIC ring buffer sizes
---
TOOL: ethtool
CMD: ethtool -G eth0 rx 4096 tx 4096
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Set both RX and TX ring buffers to 4096
---
TOOL: ethtool
CMD: ethtool -K eth0 gro on
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Enable Generic Receive Offload for higher throughput
---
TOOL: ethtool
CMD: ethtool -K eth0 gro off
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Disable GRO for lower latency (single-packet processing)
---
TOOL: ethtool
CMD: watch -n 1 "ethtool -S eth0 | grep -E '(miss|drop|overflow)'"
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Live monitoring of NIC drop/miss/overflow counters
---
TOOL: ethtool
CMD: ethtool -S veth0 | grep peer_ifindex
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Find peer interface index of a veth pair
---
TOOL: ethtool
CMD: ethtool -l eth0
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Check current and maximum NIC queue (channel) count
---
TOOL: ethtool
CMD: ethtool -L eth0 combined 16
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Set NIC queue count to 16 for RSS parallelism
---
TOOL: ethtool
CMD: ethtool -x eth0
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: View RSS hash indirection table
---
TOOL: ethtool
CMD: ethtool -X eth0 equal 16
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Distribute RSS evenly across 16 queues
---
TOOL: ethtool
CMD: ethtool -n eth0 rx-flow-hash tcp4
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Check which fields are used for RSS hash on TCP/IPv4
---
TOOL: ethtool
CMD: ethtool -c eth0
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: View NIC interrupt coalescing settings (rx-usecs, rx-frames)
---
TOOL: ethtool
CMD: ethtool -C eth0 rx-usecs 100 rx-frames 128
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Set high coalescing for throughput-optimized workloads
---
TOOL: ethtool
CMD: ethtool -C eth0 rx-usecs 10 rx-frames 8
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Set low coalescing for latency-sensitive workloads
---
TOOL: ethtool
CMD: ethtool -C eth0 adaptive-rx on adaptive-tx on
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Enable adaptive coalescing — NIC adjusts automatically
---
TOOL: ethtool
CMD: ethtool -C eth0 rx-usecs 0 rx-frames 1
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Ultra-low latency: interrupt on every packet (high CPU cost)
---
TOOL: ethtool
CMD: ethtool -i eth0 | grep driver
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Check NIC driver — ixgbe/mlx5 supports AF_XDP zero-copy
---

## TOOL: helm

TOOL: helm
CMD: helm upgrade cilium cilium/cilium --set encryption.enabled=true --set encryption.type=wireguard
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Enable WireGuard encryption for Cilium pod-to-pod traffic
---
TOOL: helm
CMD: helm install cilium cilium/cilium --set kubeProxyReplacement=true --set k8sServiceHost=API_SERVER_IP --set k8sServicePort=443
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Install Cilium with kube-proxy replacement mode enabled
---

## TOOL: ipcalc

TOOL: ipcalc
CMD: ipcalc 10.0.0.0/24
SOURCE: 01-networking-fundamentals/02-ip-addressing-subnetting.md
CONTEXT: Calculate subnet details: network, broadcast, host range, usable IPs
---

## TOOL: iperf3

TOOL: iperf3
CMD: iperf3 -s -p 5201
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Start iperf3 server on port 5201
---
TOOL: iperf3
CMD: iperf3 -c <server> -t 30
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Single-stream throughput test for 30 seconds
---
TOOL: iperf3
CMD: iperf3 -c <server> -t 30 -P 8
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: 8 parallel streams — tests RSS distribution and aggregate throughput
---
TOOL: iperf3
CMD: iperf3 -c <server> -t 30 -R
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Reverse direction test: server→client throughput
---
TOOL: iperf3
CMD: iperf3 -c <server> -u -b 1G -t 30
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: UDP throughput test at 1Gbps target
---

## TOOL: ip

TOOL: ip
CMD: ip route add 10.50.0.5/32 via 10.0.0.1 dev eth0
SOURCE: 01-networking-fundamentals/02-ip-addressing-subnetting.md
CONTEXT: Add host route for specific /32 destination
---
TOOL: ip
CMD: ip addr add 10.0.0.0/31 dev eth1
SOURCE: 01-networking-fundamentals/02-ip-addressing-subnetting.md
CONTEXT: Assign /31 point-to-point address to interface
---
TOOL: ip
CMD: ip neigh show
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Show ARP/neighbor cache (MAC-to-IP mappings)
---
TOOL: ip
CMD: ip route show
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Show main routing table
---
TOOL: ip
CMD: ip route show table main
SOURCE: 01-networking-fundamentals/05-routing-protocols.md
CONTEXT: Explicitly show main routing table
---
TOOL: ip
CMD: ip route get 8.8.8.8
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Show which route/interface would be used to reach 8.8.8.8
---
TOOL: ip
CMD: ip -s link show eth0
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Show interface statistics including TX/RX bytes and errors
---
TOOL: ip
CMD: ip neigh add ... nud permanent
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Add static permanent ARP entry (no expiry)
---
TOOL: ip
CMD: ip neigh del ... dev eth0
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Delete a specific ARP entry from the neighbor cache
---
TOOL: ip
CMD: ip neigh flush all
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Flush entire ARP cache
---
TOOL: ip
CMD: watch -n 1 'ip neigh show'
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Monitor ARP table changes in real time
---
TOOL: ip
CMD: ip -6 neigh show
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Show IPv6 NDP neighbor cache
---
TOOL: ip
CMD: ip route add 10.0.2.0/24 via 10.0.0.1
SOURCE: 01-networking-fundamentals/05-routing-protocols.md
CONTEXT: Add static route to subnet via next-hop gateway
---
TOOL: ip
CMD: ip route add 0.0.0.0/0 via 10.0.0.1
SOURCE: 01-networking-fundamentals/05-routing-protocols.md
CONTEXT: Add default route via gateway
---
TOOL: ip
CMD: ip route add 192.168.0.0/16 blackhole
SOURCE: 01-networking-fundamentals/05-routing-protocols.md
CONTEXT: Add blackhole route to silently drop traffic to a prefix
---
TOOL: ip
CMD: ip route add 10.0.4.0/24 via 10.0.0.1 metric 100
SOURCE: 01-networking-fundamentals/05-routing-protocols.md
CONTEXT: Add route with explicit metric for preference ordering
---
TOOL: ip
CMD: ip -d link show type vxlan
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Show VXLAN interfaces with detailed info (VNI, port, remote)
---
TOOL: ip
CMD: ip -d link show flannel.1 | grep -E "vxlan|mtu|qdisc"
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Show Flannel VXLAN interface details including MTU
---
TOOL: ip
CMD: ip link show flannel.1
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Show Flannel VXLAN interface basic state
---
TOOL: ip
CMD: ip link set dev flannel.1 mtu 1450
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Set VXLAN interface MTU to 1450 to account for 50-byte VXLAN overhead
---
TOOL: ip
CMD: ip link set dev vxlan.calico mtu 1450
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Set Calico VXLAN interface MTU
---
TOOL: ip
CMD: ip -d link show type geneve
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Show GENEVE tunnel interfaces with detailed parameters
---
TOOL: ip
CMD: ip route show table main | grep 10.244
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Check pod CIDR routes in main routing table (Flannel/Calico)
---
TOOL: ip
CMD: ip rule show
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Show all RPDB (Routing Policy Database) rules in priority order
---
TOOL: ip
CMD: ip route show table all
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Show routes in ALL routing tables
---
TOOL: ip
CMD: ip route get 8.8.8.8 from 10.0.0.5
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Simulate kernel route lookup for specific source IP (RPDB-aware)
---
TOOL: ip
CMD: ip route add 198.51.100.0/24 dev eth0 table isp_a
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Add route to custom table for source-based routing
---
TOOL: ip
CMD: ip route add default via 198.51.100.1 table isp_a
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Add default route in ISP-A custom table for multi-homed routing
---
TOOL: ip
CMD: ip rule add from 198.51.100.10 table isp_a
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Add RPDB rule to route traffic from specific source IP via custom table
---
TOOL: ip
CMD: ip route get 8.8.8.8 from 198.51.100.10
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Verify source-based routing: confirm correct nexthop for ISP-A source
---
TOOL: ip
CMD: ip link add vrf-mgmt type vrf table 100
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Create VRF device with dedicated routing table 100
---
TOOL: ip
CMD: ip link set vrf-mgmt up
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Bring VRF device up
---
TOOL: ip
CMD: ip link set eth0 master vrf-mgmt
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Assign interface eth0 to VRF management domain
---
TOOL: ip
CMD: ip route add 0.0.0.0/0 via 10.0.0.254 vrf vrf-mgmt
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Add default route within VRF routing table
---
TOOL: ip
CMD: ip route show vrf vrf-mgmt
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Show routes in a specific VRF
---
TOOL: ip
CMD: ip vrf exec vrf-mgmt ping 10.0.0.254
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Execute command in VRF context
---
TOOL: ip
CMD: ip link show type vrf
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: List all VRF devices
---
TOOL: ip
CMD: ip route add 10.0.0.0/8 nexthop via 192.168.1.1 dev eth0 weight 1 nexthop via 192.168.2.1 dev eth1 weight 1
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Add ECMP route with two equal-weight nexthops
---
TOOL: ip
CMD: ip route get 10.5.5.5 from 192.168.1.100 sport 45678 dport 80 proto tcp
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Simulate ECMP hash for specific 5-tuple flow
---
TOOL: ip
CMD: ip nexthop add id 1 via 192.168.1.1 dev eth0
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Create named nexthop object for resilient ECMP groups
---
TOOL: ip
CMD: ip nexthop add id 10 group 1/2/3 type resilient buckets 128 idle_timer 60 unbalanced_timer 300
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Create resilient nexthop group (kernel 5.12+) — survives nexthop failure without disrupting existing flows
---
TOOL: ip
CMD: ip route add 10.0.0.0/8 nhid 10
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Assign resilient nexthop group to route
---
TOOL: ip
CMD: ip nexthop show id 10
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Show nexthop group configuration and state
---
TOOL: ip
CMD: ip nexthop bucket show id 10
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Show bucket assignments in resilient ECMP group
---
TOOL: ip
CMD: ip route monitor
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Watch routing table changes in real time
---
TOOL: ip
CMD: ip nexthop monitor
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Monitor nexthop changes for ECMP/BFD environments
---
TOOL: ip
CMD: ip route show root 10.0.0.0/8
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Show all routes for a prefix across all tables
---
TOOL: ip
CMD: ip rule del from 10.0.0.0/8 table monitoring
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Delete overly broad RPDB rule causing wrong routing
---
TOOL: ip
CMD: ip rule add from 10.0.5.10/32 table monitoring
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Replace broad rule with narrow /32 source rule
---
TOOL: ip
CMD: ip rule add fwmark 0x10 table monitoring
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Route by firewall mark instead of source IP for robustness
---
TOOL: ip
CMD: ip link add veth0 type veth peer name veth1
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Create veth pair (used for container/namespace networking)
---
TOOL: ip
CMD: ip link add ns1 type netns
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Create a new network namespace named ns1
---
TOOL: ip
CMD: ip link set veth1 netns ns1
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Move veth1 into network namespace ns1
---
TOOL: ip
CMD: ip link show veth0
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Show veth interface state
---
TOOL: ip
CMD: ip netns exec ns1 ip link show veth1
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Show interface state inside a network namespace
---
TOOL: ip
CMD: ip tuntap add dev tun0 mode tun user $(whoami)
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Create TUN (L3) interface for VPN/tunnel applications
---
TOOL: ip
CMD: ip tuntap add dev tap0 mode tap
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Create TAP (L2) interface for virtual machine bridging
---
TOOL: ip
CMD: ip link add macvlan0 link eth0 type macvlan mode bridge
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Create macvlan interface in bridge mode on eth0
---
TOOL: ip
CMD: ip link add ipvlan0 link eth0 type ipvlan mode l3
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Create ipvlan interface in L3 mode (no ARP)
---
TOOL: ip
CMD: ip netns add myns
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Create a new named network namespace
---
TOOL: ip
CMD: ip netns list
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: List all named network namespaces
---
TOOL: ip
CMD: ip netns exec myns ip link show
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Execute command inside a named network namespace
---
TOOL: ip
CMD: ip netns del myns
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Delete a named network namespace
---
TOOL: ip
CMD: ip link add veth-host type veth peer name veth-web
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Create veth pair for connecting namespace to host
---
TOOL: ip
CMD: ip link set veth-web netns web
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Move veth peer into web namespace
---
TOOL: ip
CMD: ip netns exec web ip addr add 10.100.0.2/24 dev veth-web
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Assign IP to veth interface inside namespace
---
TOOL: ip
CMD: ip netns exec web ip link set veth-web up
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Bring veth interface up inside namespace
---
TOOL: ip
CMD: ip netns exec web ip link set lo up
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Bring loopback up inside namespace
---
TOOL: ip
CMD: ip netns exec web ip route add default via 10.100.0.1
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Add default route inside namespace
---
TOOL: ip
CMD: ip netns exec cni-abc12345 ip addr
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Check pod IP addresses via CNI-created namespace
---
TOOL: ip
CMD: ip netns exec cni-abc12345 ss -tlnp
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Check listening ports inside pod's CNI namespace
---
TOOL: ip
CMD: ip link set eth0 xdp obj my_xdp.o sec xdp
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Attach XDP program to interface
---
TOOL: ip
CMD: ip link show eth0
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Check if XDP program is attached and show mode (native/generic)
---
TOOL: ip
CMD: ip link set eth0 xdp obj my_xdp.o sec xdp mode native
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Force native XDP mode (fails if driver doesn't support)
---
TOOL: ip
CMD: ip link set eth0 xdp obj my_xdp.o sec xdp mode skbmode
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Force generic (skb) XDP mode — works on any NIC
---
TOOL: ip
CMD: ip link set eth0 xdp off
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Detach XDP program from interface
---
TOOL: ip
CMD: ip link set eth0 xdp obj xdp_redirect.o sec xdp
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Load XDP redirect program required before AF_XDP socket works
---
TOOL: ip
CMD: ip route get 139.213.49.40
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Verify connectivity to an IP after removing it from XDP blocklist
---

## TOOL: ip6tables

TOOL: ip6tables
CMD: (no explicit ip6tables commands found; IPv6 filtering via nftables in this codebase)
SOURCE: N/A
CONTEXT: IPv6 filtering handled via nftables or iptables-translate
---

## TOOL: iptables

TOOL: iptables
CMD: iptables -I INPUT -p icmp --icmp-type fragmentation-needed -j ACCEPT
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Allow ICMP fragmentation-needed messages (required for PMTUD)
---
TOOL: iptables
CMD: iptables -I FORWARD -p icmp --icmp-type fragmentation-needed -j ACCEPT
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Allow forwarded ICMP fragmentation-needed (fixes PMTUD black holes)
---
TOOL: iptables
CMD: iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN SYN -j TCPMSS --clamp-mss-to-pmtu
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Clamp TCP MSS to path MTU to prevent MTU black holes
---
TOOL: iptables
CMD: iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j SNAT --to-source 1.2.3.4
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Static SNAT — translate private source to specific public IP
---
TOOL: iptables
CMD: iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: MASQUERADE — dynamic SNAT using interface IP (for DHCP/dynamic IPs)
---
TOOL: iptables
CMD: iptables -t nat -A PREROUTING -d 1.2.3.4 -p tcp --dport 80 -j DNAT --to-destination 10.0.0.10:8080
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: DNAT — redirect inbound traffic to internal host/port
---
TOOL: iptables
CMD: iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN SYN -j TCPMSS --set-mss 1396
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Set fixed TCP MSS 1396 for VXLAN overlay (1450 MTU - 54B headers)
---
TOOL: iptables
CMD: iptables -L INPUT -n -v --line-numbers
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: List INPUT chain rules with packet counts and line numbers
---
TOOL: iptables
CMD: iptables -t nat -L KUBE-SERVICES -n
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: List kube-proxy service NAT chain rules
---
TOOL: iptables
CMD: iptables-save | wc -l
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Count total iptables rules — high count causes O(n) lookup latency
---
TOOL: iptables
CMD: iptables -V
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Check iptables version (legacy vs nft backend)
---
TOOL: iptables
CMD: iptables-translate -A INPUT -p tcp --dport 80 -j ACCEPT
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Translate iptables rule to nftables equivalent
---
TOOL: iptables
CMD: iptables-save | iptables-restore-translate -f - | nft -f -
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Migrate full iptables ruleset to nftables
---
TOOL: iptables
CMD: iptables -I INPUT 1 -j LOG --log-prefix "IPT-DEBUG: " --log-level 4 --log-uid
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Add debug logging rule to capture packet info in kernel log
---
TOOL: iptables
CMD: iptables -t raw -I PREROUTING -s 10.0.0.5 -p tcp --dport 80 -j TRACE
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Enable iptables TRACE for specific flow to follow through all chains
---
TOOL: iptables
CMD: iptables -t raw -I PREROUTING -p tcp -d 10.96.100.50 --dport 443 -j TRACE
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Trace packets to a Kubernetes ClusterIP through all iptables chains
---
TOOL: iptables
CMD: iptables -L -n | grep -i icmp
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Check if ICMP is being blocked (causing MTU black holes in VXLAN)
---
TOOL: iptables
CMD: iptables -D INPUT -p icmp -j DROP
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Remove rule blocking ICMP (fix for PMTUD/MTU black holes)
---
TOOL: iptables
CMD: iptables -t mangle -A OUTPUT -d 10.8.0.0/8 -j MARK --set-mark 0x1
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Mark packets for policy routing via fwmark
---
TOOL: iptables
CMD: iptables -t mangle -A OUTPUT -p tcp --dport 9090 -j MARK --set-mark 0x10
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Mark monitoring traffic by destination port for policy routing
---
TOOL: iptables
CMD: iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Enable internet access from containers via host MASQUERADE
---
TOOL: iptables
CMD: iptables -t nat -L DOCKER -n -v
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Show Docker NAT rules for port mapping
---
TOOL: iptables
CMD: iptables -t nat -L POSTROUTING -n -v
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Show all POSTROUTING NAT rules including Docker/Kubernetes
---
TOOL: iptables
CMD: iptables -A FORWARD -i vrf-mgmt -o vrf-data -j DROP
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Block forwarding between VRFs at iptables level (defense-in-depth)
---

## TOOL: journalctl

TOOL: journalctl
CMD: journalctl -k | grep "IPT-DEBUG"
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Read iptables LOG rule output from kernel journal
---

## TOOL: kdig

TOOL: kdig
CMD: kdig -d @1.1.1.1 +tls-ca example.com A
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: DNS-over-TLS query to Cloudflare using kdig with certificate verification
---

## TOOL: kubectl

TOOL: kubectl
CMD: kubectl get pods -n kube-system -l k8s-app=kube-dns
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Check CoreDNS pod status
---
TOOL: kubectl
CMD: kubectl logs -n kube-system -l k8s-app=kube-dns -f
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Stream CoreDNS logs for DNS query debugging
---
TOOL: kubectl
CMD: kubectl run -it --rm debug --image=busybox --restart=Never
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Launch ephemeral debug pod for network testing
---
TOOL: kubectl
CMD: kubectl exec -it debug-pod -- nslookup redis
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Test Kubernetes service DNS resolution inside cluster
---
TOOL: kubectl
CMD: kubectl port-forward -n kube-system svc/kube-dns 9153:9153
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Port-forward CoreDNS Prometheus metrics endpoint
---
TOOL: kubectl
CMD: kubectl rollout restart deployment/coredns -n kube-system
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Restart CoreDNS pods to pick up ConfigMap changes
---
TOOL: kubectl
CMD: kubectl exec -it payment-pod -- ping -c 3 -M do -s 1400 ...
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Test MTU from inside pod — check for VXLAN MTU black holes
---
TOOL: kubectl
CMD: kubectl exec -it payment-pod -- tcpdump -i eth0 icmp
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Capture ICMP packets inside pod to debug PMTUD issues
---
TOOL: kubectl
CMD: kubectl patch felixconfiguration default ...
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Patch Calico FelixConfiguration for MTU or policy settings
---
TOOL: kubectl
CMD: kubectl apply -f https://k8s.io/examples/admin/dns/nodelocaldns.yaml
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Deploy NodeLocal DNSCache to fix DNS 5-second timeout race condition
---
TOOL: kubectl
CMD: kubectl debug -it mypod --image=nicolaka/netshoot --target=mycontainer
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Attach ephemeral debug container sharing pod's network namespace
---
TOOL: kubectl
CMD: kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Enable VPC CNI prefix delegation to increase pod capacity from 58 to 898 per node
---
TOOL: kubectl
CMD: kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Set warm prefix target for VPC CNI prefix delegation mode
---
TOOL: kubectl
CMD: kubectl set env daemonset aws-node -n kube-system WARM_IP_TARGET=10 MINIMUM_IP_TARGET=5
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Pre-allocate 10 IPs per node to reduce pod start latency on burst workloads
---
TOOL: kubectl
CMD: kubectl get cninode ip-10-0-10-100.ec2.internal -o yaml
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Check VPC CNI node ENI/prefix allocation state (v1.12+)
---
TOOL: kubectl
CMD: kubectl logs -n kube-system -l k8s-app=aws-node --tail=100 | grep -i "ip\|eni\|prefix\|warm"
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Check VPC CNI (aws-node) logs for IP allocation errors
---
TOOL: kubectl
CMD: kubectl get pods --all-namespaces -o wide | awk '{print $8}' | sort | uniq -c | sort -rn
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Count pods per node to find nodes at capacity
---
TOOL: kubectl
CMD: kubectl describe node ip-10-0-10-100.ec2.internal | grep -A5 "Capacity:\|Allocatable:"
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Check node pod capacity and allocatable count
---
TOOL: kubectl
CMD: kubectl drain ip-10-0-10-100.ec2.internal --ignore-daemonsets
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Drain node before terminating to apply new prefix delegation config
---
TOOL: kubectl
CMD: kubectl get nodes -o jsonpath='{.items[*].metadata.labels}' | grep instance-type
SOURCE: 03-cloud-networking/aws/05-eks-networking.md
CONTEXT: Check node instance types to verify Nitro support for prefix delegation
---
TOOL: kubectl
CMD: kubectl get pods -A | grep -v Running
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Check for pending/failed pods — symptom of IP exhaustion
---
TOOL: kubectl
CMD: kubectl describe pod <pending-pod> | grep Events
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Check pending pod events for CNI IP allocation failure messages
---
TOOL: kubectl
CMD: kubectl get nodes -o wide
SOURCE: 03-cloud-networking/azure/04-aks-networking.md
CONTEXT: Check node status — NotReady indicates CNI/IP issues
---

## TOOL: lsof

TOOL: lsof
CMD: (no explicit lsof commands; strace/ss used for socket debugging in this codebase)
SOURCE: N/A
CONTEXT: Socket inspection done via ss -tlnp in this codebase
---

## TOOL: modprobe

TOOL: modprobe
CMD: modprobe tcp_bbr
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Load BBR congestion control kernel module
---
TOOL: modprobe
CMD: modprobe nf_log_ipv4
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Load nf_log_ipv4 module required for iptables TRACE
---

## TOOL: mtr

TOOL: mtr
CMD: mtr --report --report-cycles 20 8.8.8.8
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: MTR report with 20 cycles — shows per-hop latency and loss
---
TOOL: mtr
CMD: mtr --report --report-cycles 50 8.8.8.8
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: MTR with 50 cycles for more accurate loss statistics
---

## TOOL: nc

TOOL: nc
CMD: nc -zv -w 3 10.0.2.50 443
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: TCP connectivity test to host:port with 3s timeout
---

## TOOL: netperf

TOOL: netperf
CMD: netperf -t TCP_RR -H <server> -l 30 -- -r 1,1
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Request-Response test measuring transaction latency
---
TOOL: netperf
CMD: netperf -t TCP_STREAM -H <server> -l 30 -- -m 1460
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: TCP stream throughput test with 1460-byte messages
---

## TOOL: netstat

TOOL: netstat
CMD: netstat -s | grep -i retransmit
SOURCE: 00-networking-core/03-transport-layer.md
CONTEXT: Check TCP retransmit counters for packet loss
---
TOOL: netstat
CMD: watch -n 1 'netstat -s | grep retransmit'
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Monitor retransmit rate in real time
---

## TOOL: nft

TOOL: nft
CMD: nft list ruleset
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Show entire nftables ruleset
---
TOOL: nft
CMD: nft add table inet myfilter
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Create nftables table for dual-stack (inet) filtering
---
TOOL: nft
CMD: nft add chain ...
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Create nftables chain with type/hook/priority
---
TOOL: nft
CMD: nft add rule ...
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Add a rule to an nftables chain
---
TOOL: nft
CMD: nft -f /etc/nftables/ruleset.nft
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Load nftables ruleset from file atomically
---
TOOL: nft
CMD: nft monitor trace
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Monitor nftables trace events for packet path debugging
---

## TOOL: nslookup

TOOL: nslookup
CMD: (see dig and kubectl exec -- nslookup entries above)
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Used inside pods for Kubernetes service DNS testing
---

## TOOL: nsenter

TOOL: nsenter
CMD: nsenter -t $PID -n ip addr
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Show IP addresses inside container's network namespace
---
TOOL: nsenter
CMD: nsenter -t $PID -n ip route
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Show routes inside container's network namespace
---
TOOL: nsenter
CMD: nsenter -t $PID -n ss -tlnp
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Show listening sockets inside container's namespace
---
TOOL: nsenter
CMD: nsenter -t $PID -n netstat -tlnp
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Show listening ports using netstat inside container namespace
---
TOOL: nsenter
CMD: nsenter -t $PID -n tcpdump -i eth0 -w /tmp/capture.pcap
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Capture packets inside container namespace using host's tcpdump
---
TOOL: nsenter
CMD: nsenter -t $PID --all
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Enter ALL namespaces of a container process
---
TOOL: nsenter
CMD: nsenter -t $PID -n -u
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Enter network and UTS namespace of a container
---
TOOL: nsenter
CMD: nsenter -t $CTR_PID -n ip link show eth0 | grep -oP 'if\K[0-9]+'
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Get interface index inside container to match tc/veth on host
---
TOOL: nsenter
CMD: nsenter -t $POD_PID -n cat /etc/resolv.conf
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Read pod resolv.conf to verify NodeLocal DNSCache address
---

## TOOL: numactl

TOOL: numactl
CMD: numactl --hardware
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Show NUMA topology: nodes, CPUs per node, memory per node
---
TOOL: numactl
CMD: numactl --cpunodebind=0 --membind=0 ./my_server
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Pin application to NUMA node 0 for locality with NIC on node 0
---
TOOL: numactl
CMD: numastat -p $(pgrep my_server)
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Check per-NUMA-node memory hits/misses for a process
---

## TOOL: openssl

TOOL: openssl
CMD: openssl s_client -connect example.com:443 -tls1_3
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Test TLS 1.3 connection — verify certificate and cipher suite
---
TOOL: openssl
CMD: openssl s_client -connect 1.1.1.1:853 -servername 1.1.1.1
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Test DNS-over-TLS (DoT) connection to port 853
---

## TOOL: ovs-vsctl / ovs-ofctl

TOOL: ovs-vsctl
CMD: ovs-vsctl show
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Show OVS bridge configuration and port topology
---
TOOL: ovs-ofctl
CMD: ovs-ofctl dump-flows br0
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Show OpenFlow rules on OVS bridge br0
---
TOOL: ovs-ofctl
CMD: ovs-ofctl add-flow br0 "in_port=1,actions=output:2"
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Add OpenFlow rule to forward traffic between OVS ports
---

## TOOL: perf

TOOL: perf
CMD: perf record -g -e 'net:*' -a sleep 5
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Record network tracepoint events for 5 seconds
---
TOOL: perf
CMD: perf report
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Display perf recording report with call graph
---

## TOOL: ping

TOOL: ping
CMD: ping -c 4 10.0.1.1
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Basic connectivity test with 4 packets
---
TOOL: ping
CMD: ping -c 3 -M do -s 64 10.0.1.100
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: PMTUD test with DF bit set, small payload — should succeed
---
TOOL: ping
CMD: ping -c 3 -M do -s 1400 10.0.1.100
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: PMTUD test with DF bit, 1400-byte payload — detects MTU black holes
---

## TOOL: rdisc6

TOOL: rdisc6
CMD: rdisc6 eth0
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Solicit IPv6 Router Advertisement on interface
---

## TOOL: resolvectl

TOOL: resolvectl
CMD: resolvectl status
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Show systemd-resolved status: DNS servers, search domains, protocols per interface
---

## TOOL: ss

TOOL: ss
CMD: ss -tnlp
SOURCE: 00-networking-core/01-internet-fundamentals.md
CONTEXT: List listening TCP sockets with process names
---
TOOL: ss
CMD: ss -tnp
SOURCE: 00-networking-core/01-internet-fundamentals.md
CONTEXT: List established TCP connections with process names
---
TOOL: ss
CMD: ss -ti
SOURCE: 00-networking-core/03-transport-layer.md
CONTEXT: Show TCP connection internals: RTT, cwnd, ssthresh, retransmit counts
---
TOOL: ss
CMD: ss -u -a
SOURCE: 00-networking-core/03-transport-layer.md
CONTEXT: Show all UDP sockets (listening and connected)
---
TOOL: ss
CMD: ss -tnp
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Show TCP connections with process info for L4 debugging
---
TOOL: ss
CMD: ss -s
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Socket summary statistics (total counts by state)
---
TOOL: ss
CMD: ss -tn state time-wait | wc -l
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Count TCP TIME_WAIT sockets
---
TOOL: ss
CMD: ss -tni dst 10.0.1.100
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Show TCP internals for connections to a specific destination
---
TOOL: ss
CMD: ss -tn state close-wait | wc -l
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Count CLOSE_WAIT sockets (server not closing after client FIN)
---
TOOL: ss
CMD: ss -tnp state close-wait
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Show processes with CLOSE_WAIT connections
---
TOOL: ss
CMD: ss -ti dst <remote_ip>
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Show TCP internals for connections to a specific host during throughput test
---
TOOL: ss
CMD: ss -tm
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Show TCP socket memory usage details
---

## TOOL: strace

TOOL: strace
CMD: strace -e connect your-application 2>&1 | grep EADDRNOTAVAIL
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Detect ephemeral port exhaustion (EADDRNOTAVAIL on connect)
---
TOOL: strace
CMD: strace -e trace=setsockopt -p $APP_PID 2>&1 | grep -i rcvbuf
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Detect if application calls setsockopt(SO_RCVBUF) disabling auto-tuning
---

## TOOL: sysctl

TOOL: sysctl
CMD: sysctl net.ipv4.tcp_congestion_control
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Show current TCP congestion control algorithm
---
TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_congestion_control=bbr
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Switch TCP congestion control to BBR
---
TOOL: sysctl
CMD: sysctl -w net.core.default_qdisc=fq
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Set default qdisc to fq (required for BBR pacing)
---
TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_fin_timeout=15
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Reduce FIN_WAIT2 timeout to reclaim sockets faster
---
TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_tw_reuse=1
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Allow reuse of TIME_WAIT sockets for new outbound connections
---
TOOL: sysctl
CMD: sysctl net.netfilter.nf_conntrack_max
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Check conntrack table maximum size
---
TOOL: sysctl
CMD: sysctl net.netfilter.nf_conntrack_count
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Check current number of conntrack entries
---
TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_max=524288
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Increase conntrack table to 512K entries to prevent exhaustion
---
TOOL: sysctl
CMD: sysctl -w net.ipv4.ip_local_port_range="1024 65535"
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Expand ephemeral port range to maximize outbound connections
---
TOOL: sysctl
CMD: sysctl net.ipv4.fib_multipath_hash_policy
SOURCE: 01-networking-fundamentals/05-routing-protocols.md
CONTEXT: Check ECMP hash policy (0=L3, 1=L3+L4, 2=L3+L4+inner)
---
TOOL: sysctl
CMD: sysctl -w net.ipv4.fib_multipath_hash_policy=1
SOURCE: 01-networking-fundamentals/05-routing-protocols.md
CONTEXT: Set ECMP to L3+L4 hash for per-flow distribution
---
TOOL: sysctl
CMD: sysctl net.ipv4.ip_no_pmtu_disc
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Check PMTUD setting (0=enabled, 1=disabled)
---
TOOL: sysctl
CMD: sysctl -w net.core.netdev_budget=600
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Increase NAPI budget — process more packets per softirq poll
---
TOOL: sysctl
CMD: sysctl -w net.core.netdev_max_backlog=10000
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Increase backlog queue size to prevent drops during bursts
---
TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_max=262144
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Set conntrack table size for high-traffic nodes
---
TOOL: sysctl
CMD: sysctl net.netfilter.nf_log.2=nf_log_ipv4
SOURCE: 02-linux-networking/03-iptables-nftables.md
CONTEXT: Enable nf_log_ipv4 for iptables TRACE functionality
---
TOOL: sysctl
CMD: sysctl -w net.ipv4.ip_forward=1
SOURCE: 02-linux-networking/02-virtual-networking.md
CONTEXT: Enable IP forwarding for routing/NAT on Linux
---
TOOL: sysctl
CMD: sysctl net.ipv4.tcp_rmem
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Show TCP receive buffer sizes (min, default, max)
---
TOOL: sysctl
CMD: sysctl net.ipv4.tcp_wmem
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Show TCP send buffer sizes (min, default, max)
---
TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_rmem="4096 131072 67108864"
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Set TCP receive buffer max to 64MB for high-bandwidth connections
---
TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_wmem="4096 16384 67108864"
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Set TCP send buffer max to 64MB
---
TOOL: sysctl
CMD: sysctl -w net.core.rmem_max=67108864
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Set system-wide receive buffer maximum (SO_RCVBUF ceiling)
---
TOOL: sysctl
CMD: sysctl -w net.core.wmem_max=67108864
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Set system-wide send buffer maximum
---
TOOL: sysctl
CMD: sysctl net.ipv4.tcp_mem
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Show global TCP memory pressure thresholds (pages)
---
TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_mem="4000000 6000000 8000000"
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Set TCP global memory limits to allow large buffer usage
---
TOOL: sysctl
CMD: sysctl net.ipv4.tcp_window_scaling
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Check TCP window scaling enabled (should be 1)
---
TOOL: sysctl
CMD: sysctl -w net.core.busy_poll=50
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Enable busy polling for ultra-low latency (50µs poll period)
---
TOOL: sysctl
CMD: sysctl -w net.core.busy_read=50
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Set busy read poll time for SO_BUSY_POLL sockets
---
TOOL: sysctl
CMD: sysctl net.ipv4.ip_forward
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Check if IP forwarding is enabled (must be 1 for routing)
---

## TOOL: tc

TOOL: tc
CMD: tc qdisc show dev eth0
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Show current qdisc on interface
---
TOOL: tc
CMD: tc qdisc replace dev eth0 root fq_codel
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Set fq_codel as root qdisc for AQM (Active Queue Management)
---
TOOL: tc
CMD: tc qdisc replace dev eth0 root fq_codel target 5ms interval 100ms quantum 1514 limit 10240
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Configure fq_codel with explicit target latency and queue limit
---
TOOL: tc
CMD: tc -s qdisc show dev eth0
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Show qdisc statistics including drops and bytes
---
TOOL: tc
CMD: tc qdisc add dev eth0 root handle 1: htb default 30
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Create HTB root qdisc for hierarchical bandwidth control
---
TOOL: tc
CMD: tc class add dev eth0 parent 1: classid 1:1 htb rate 1gbit
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Add HTB root class with 1Gbps total bandwidth
---
TOOL: tc
CMD: tc class add dev eth0 parent 1:1 classid 1:10 htb rate 500mbit ceil 800mbit burst 15k
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Add HTB leaf class with guaranteed 500Mbps, burst to 800Mbps
---
TOOL: tc
CMD: tc qdisc add dev eth0 parent 1:10 handle 10: fq_codel
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Attach fq_codel to HTB leaf class for AQM within bandwidth class
---
TOOL: tc
CMD: tc filter add dev eth0 parent 1: protocol ip u32 match ip src 10.244.1.0/24 flowid 1:10
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Classify pod CIDR traffic into specific HTB class
---
TOOL: tc
CMD: tc qdisc add dev eth0 root tbf rate 10mbit burst 32kbit latency 400ms
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Token Bucket Filter for simple rate limiting
---
TOOL: tc
CMD: tc qdisc replace dev eth0 root fq
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Set fair-queue qdisc (required for BBR pacing)
---
TOOL: tc
CMD: tc qdisc add dev eth0 root netem delay 100ms
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Add 100ms artificial delay (netem for testing)
---
TOOL: tc
CMD: tc qdisc add dev eth0 root netem delay 100ms 20ms distribution normal
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Add delay with jitter using normal distribution
---
TOOL: tc
CMD: tc qdisc add dev eth0 root netem loss 1%
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Add 1% random packet loss for testing
---
TOOL: tc
CMD: tc qdisc add dev eth0 root netem delay 50ms 10ms loss 0.5%
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Combined delay + jitter + loss simulation
---
TOOL: tc
CMD: tc qdisc add dev eth0 root netem corrupt 0.1%
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Add 0.1% packet corruption for robustness testing
---
TOOL: tc
CMD: tc qdisc add dev eth0 root netem delay 100ms reorder 25% gap 5
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Add packet reordering (25% of packets reordered after gap of 5)
---
TOOL: tc
CMD: ip link add ifb0 type ifb
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Create IFB (Intermediate Functional Block) for ingress shaping
---
TOOL: tc
CMD: ip link set ifb0 up
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Bring IFB interface up
---
TOOL: tc
CMD: tc qdisc add dev eth0 handle ffff: ingress
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Add ingress qdisc to redirect traffic to IFB for shaping
---
TOOL: tc
CMD: tc filter add dev eth0 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Redirect all ingress traffic to IFB for ingress rate limiting
---
TOOL: tc
CMD: tc class show dev eth0
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Show all HTB classes and their bandwidth stats
---
TOOL: tc
CMD: tc filter show dev eth0
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Show all tc filters and their match rules
---
TOOL: tc
CMD: tc monitor
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Watch tc changes (qdisc/class/filter) in real time
---
TOOL: tc
CMD: tc qdisc del dev eth0 root
SOURCE: 02-linux-networking/04-tc-traffic-control.md
CONTEXT: Remove all tc qdiscs from interface (reset to default pfifo_fast)
---
TOOL: tc
CMD: tc qdisc add dev eth0 clsact
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Add clsact qdisc for TC eBPF program attachment
---
TOOL: tc
CMD: tc filter add dev eth0 ingress bpf da obj my_tc.o sec tc_ingress
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Attach TC eBPF program to ingress hook
---
TOOL: tc
CMD: tc filter add dev eth0 egress bpf da obj my_tc.o sec tc_egress
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Attach TC eBPF program to egress hook
---
TOOL: tc
CMD: tc filter show dev eth0 ingress
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: List TC ingress filters (check for unexpected eBPF programs)
---
TOOL: tc
CMD: tc filter show dev eth0 egress
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: List TC egress filters
---
TOOL: tc
CMD: tc -s filter show dev eth0 ingress
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Show TC ingress filters with statistics
---
TOOL: tc
CMD: tc filter del dev eth0 ingress
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Remove TC eBPF program from ingress hook
---
TOOL: tc
CMD: tc qdisc del dev eth0 clsact
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Remove clsact qdisc (detaches all TC eBPF programs)
---

## TOOL: tcpdump

TOOL: tcpdump
CMD: tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack|tcp-fin) != 0' -n
SOURCE: 00-networking-core/03-transport-layer.md
CONTEXT: Capture TCP handshake and teardown packets only
---
TOOL: tcpdump
CMD: tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0' -w /tmp/syn.pcap
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Capture SYN packets for window scale option analysis
---
TOOL: tcpdump
CMD: tcpdump -e -i eth0 arp
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Capture ARP packets showing Ethernet frames
---
TOOL: tcpdump
CMD: tcpdump -e -i eth0 arp -n
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Capture ARP packets without name resolution
---
TOOL: tcpdump
CMD: tcpdump -i eth0 -w /tmp/payment_debug.pcap host 10.0.1.100 and port 5432 -s 96
SOURCE: 01-networking-fundamentals/03-tcp-udp-deep-dive.md
CONTEXT: Capture PostgreSQL traffic with 96-byte snap length for header analysis
---
TOOL: tcpdump
CMD: tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: Capture only TCP SYN packets
---
TOOL: tcpdump
CMD: tcpdump -i eth0 -n 'icmp6 and (ip6[40] == 135 or ip6[40] == 136)'
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Capture IPv6 NDP Neighbor Solicitation (135) and Advertisement (136)
---
TOOL: tcpdump
CMD: tcpdump -i eth0 -n 'icmp6 and ip6[40] == 134' -v
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Capture IPv6 Router Advertisement messages (type 134) verbosely
---
TOOL: tcpdump
CMD: tcpdump -i eth0 -w /tmp/vxlan.pcap udp port 4789
SOURCE: 01-networking-fundamentals/07-vlan-vxlan-overlay-networks.md
CONTEXT: Capture VXLAN encapsulated traffic on UDP port 4789
---

## TOOL: tracepath

TOOL: tracepath
CMD: tracepath -n 10.0.1.100
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: Trace path with MTU discovery — reports MTU at each hop
---

## TOOL: traceroute

TOOL: traceroute
CMD: traceroute -n 8.8.8.8
SOURCE: 01-networking-fundamentals/01-osi-tcpip-model.md
CONTEXT: UDP traceroute without DNS resolution
---
TOOL: traceroute
CMD: traceroute -n -T -p 443 8.8.8.8
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: TCP traceroute on port 443 — passes through TCP-friendly firewalls
---
TOOL: traceroute
CMD: traceroute -n -I 8.8.8.8
SOURCE: 01-networking-fundamentals/04-arp-icmp-ndp.md
CONTEXT: ICMP traceroute (same as Windows tracert)
---

## TOOL: tshark

TOOL: tshark
CMD: tshark -r /tmp/syn.pcap -Y 'tcp.flags.syn == 1' -T fields -e tcp.options.wscale
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Extract TCP window scale option from captured SYN packets
---

## TOOL: nstat

TOOL: nstat
CMD: nstat -z | grep TcpExtListenDrops
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Check accept queue overflow drops (somaxconn too small)
---
TOOL: nstat
CMD: nstat -z | grep ListenOverflows
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Check listen queue overflow counter
---
TOOL: nstat
CMD: nstat -z | grep TcpExtTCPMemoryPressures
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Detect when TCP enters memory pressure mode (reduces buffer sizes)
---
TOOL: nstat
CMD: nstat -z | grep -E '(PruneCalled|RcvPruned|OfoPruned)'
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Check if kernel is pruning receive buffers due to memory pressure
---

## TOOL: delv

TOOL: delv
CMD: delv @8.8.8.8 example.com A +rtrace
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: DNSSEC-validating lookup with resolution trace
---

## TOOL: rndc

TOOL: rndc
CMD: rndc flushname api.company.com
SOURCE: 01-networking-fundamentals/08-dns-comprehensive.md
CONTEXT: Flush specific name from BIND resolver cache
---

## TOOL: readlink

TOOL: readlink
CMD: readlink /proc/$/ns/net
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Show network namespace inode of current process
---

## TOOL: ln

TOOL: ln
CMD: ln -s /proc/$container_pid/ns/net /run/netns/debug-container
SOURCE: 02-linux-networking/05-network-namespaces.md
CONTEXT: Create named symlink to container netns for use with ip netns exec
---

## TOOL: cat (proc/sys)

TOOL: cat
CMD: cat /proc/net/softnet_stat
SOURCE: 02-linux-networking/01-linux-network-stack.md
CONTEXT: Show per-CPU softirq network stats: processed, drops, throttled
---
TOOL: cat
CMD: cat /proc/sys/net/ipv4/ip_local_port_range
SOURCE: 01-networking-fundamentals/06-nat-pat-ephemeral-ports.md
CONTEXT: Show ephemeral port range (default 32768-60999)
---
TOOL: cat
CMD: cat /proc/net/sockstat
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Show socket usage statistics by protocol
---
TOOL: cat
CMD: cat /proc/net/fib_triestat
SOURCE: 02-linux-networking/06-linux-routing-policy.md
CONTEXT: Show routing table FIB statistics and memory usage
---
TOOL: cat
CMD: cat /sys/class/net/eth0/device/numa_node
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Check which NUMA node the NIC is physically attached to
---
TOOL: cat
CMD: cat /proc/irq/<irq>/smp_affinity_list
SOURCE: 02-linux-networking/07-socket-tuning-performance.md
CONTEXT: Check IRQ CPU affinity setting
---
TOOL: cat
CMD: ls /sys/kernel/btf/vmlinux
SOURCE: 02-linux-networking/08-ebpf-xdp-comprehensive.md
CONTEXT: Check if kernel has BTF support for CO-RE BPF programs
---

---
# BATCH1 DONE: 47 unique tools found, 364 total command entries
