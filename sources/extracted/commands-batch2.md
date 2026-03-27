# Commands Batch 2 — Extracted CLI Commands
# Sources: 04-kubernetes-networking/, 05-network-security/, 06-cloud-security/, 07-debugging-playbooks/
# Format: TOOL | CMD | SOURCE | CONTEXT

---

## arping

TOOL: arping
CMD: arping
CONTEXT: ARP-level reachability test (Layer 2 debugging)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

## aws

TOOL: aws
CMD: aws ec2 describe-security-groups --group-ids $CDE_SG_ID | jq -r '.SecurityGroups[0].IpPermissions[] | select(.IpRanges[0].CidrIp == "10.0.0.0/8") | .FromPort'
CONTEXT: Find overly broad security group rule for CDE before remediation
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws ec2 revoke-security-group-ingress --group-id $CDE_SG_ID --protocol tcp --port 0-65535 --cidr 10.0.0.0/8
CONTEXT: Remove overly broad SG ingress rule from PCI CDE security group
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws ec2 authorize-security-group-ingress --group-id $CDE_SG_ID --protocol tcp --port 8443 --source-group $PAYMENT_PROXY_SG_ID
CONTEXT: Add minimal required ingress rule (payment proxy only) to CDE SG
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws ec2 authorize-security-group-ingress --group-id $CDE_SG_ID --protocol tcp --port 5432 --source-group $CDE_APP_SG_ID
CONTEXT: Add minimal required ingress rule (CDE app servers only) for DB port
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws configservice put-conformance-pack --conformance-pack-name PCI-DSS-Conformance-Pack --template-s3-uri s3://aws-conformance-packs-us-east-1/Operational-Best-Practices-for-PCI-DSS.yaml
CONTEXT: Enable PCI DSS conformance pack in AWS Config
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws configservice put-config-rule --config-rule '{"ConfigRuleName": "cde-sg-no-broad-internal", "Source": {"Owner": "CUSTOM_LAMBDA", "SourceIdentifier": "arn:aws:lambda:us-east-1:123456789012:function:check-cde-sg", "SourceDetails": [{"EventSource": "aws.config", "MessageType": "ConfigurationItemChangeNotification"}]}, "Scope": {"ComplianceResourceTypes": ["AWS::EC2::SecurityGroup"], "TagFilters": [{"Key": "pci-cde", "Value": "true"}]}}'
CONTEXT: Create custom Config rule to alert if CDE SG allows broad internal CIDR
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws securityhub get-standards-control-associations --standards-subscription-arn arn:aws:securityhub:us-east-1:123456789012:subscription/pci-dss/v/3.2.1 | jq '.StandardsControlAssociationSummaries[] | select(.AssociationStatus == "DISABLED") | .SecurityControlId'
CONTEXT: Check which PCI controls are disabled in AWS Security Hub
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws configservice describe-compliance-by-config-rule --compliance-types NON_COMPLIANT | jq '.ComplianceByConfigRules[] | .ConfigRuleName'
CONTEXT: Find all non-compliant Config rules
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws cloudtrail lookup-events --start-time "$(date -d '1 hour ago' --iso-8601=seconds)" --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::EC2::Instance | jq '.Events | length'
CONTEXT: Verify CloudTrail is logging CDE EC2 instance API events
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::123456789012:role/CardProcessorRole --action-names "s3:*" "ec2:*" "iam:*" --resource-arns "*" | jq '.EvaluationResults[] | select(.EvalDecision == "allowed") | .EvalActionName'
CONTEXT: PCI Req 7 - Check overly broad IAM permissions for card processor role
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: aws
CMD: aws secretsmanager create-secret --name prod/payments/rds-password --secret-string '{"username":"app","password":"initial-password","engine":"postgres","host":"db.cluster.local","port":5432,"dbname":"payments"}'
CONTEXT: Create RDS password secret in AWS Secrets Manager
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: aws
CMD: aws secretsmanager rotate-secret --secret-id prod/payments/rds-password --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRotation --rotation-rules AutomaticallyAfterDays=30
CONTEXT: Enable automatic 30-day rotation for RDS password via Lambda
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: aws
CMD: aws secretsmanager describe-secret --secret-id prod/payments/database | jq '{rotation: .RotationEnabled, next: .NextRotationDate, last: .LastRotatedDate}'
CONTEXT: Check rotation schedule and status for a secret
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: aws
CMD: aws secretsmanager get-secret-value --secret-id prod/payments/database
CONTEXT: Test AWS Secrets Manager access from ESO service account
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: aws
CMD: aws monitor log-analytics query --workspace $WORKSPACE_ID --analytics-query "SigninLogs | where UserPrincipalName == 'user@corp.com' | where TimeGenerated > ago(1h) | project TimeGenerated, ResultType, ResultDescription, ConditionalAccessPolicies, DeviceDetail | order by TimeGenerated desc | take 10"
CONTEXT: Diagnose Conditional Access block via Azure Log Analytics
SOURCE: 06-cloud-security/03-azure-security.md
---

TOOL: aws
CMD: az role assignment list --assignee $PRINCIPAL_ID --all | jq '.[] | {role: .roleDefinitionName, scope: .scope}'
CONTEXT: Check effective Azure RBAC permissions for a principal
SOURCE: 06-cloud-security/03-azure-security.md
---

TOOL: aws
CMD: az keyvault show --name $KV_NAME --query "properties.enableRbacAuthorization"
CONTEXT: Verify Azure Key Vault is using RBAC authorization model
SOURCE: 06-cloud-security/03-azure-security.md
---

TOOL: aws
CMD: az role assignment list --scope $(az keyvault show --name $KV_NAME --query id -o tsv)
CONTEXT: List RBAC assignments on a specific Azure Key Vault
SOURCE: 06-cloud-security/03-azure-security.md
---

TOOL: aws
CMD: az network nic show-effective-nsg --name $NIC_NAME --resource-group $RG | jq '.effectiveSecurityRules[] | select(.access == "Deny") | {name, protocol, srcPort: .sourcePortRange, dstPort: .destinationPortRange, src: .sourceAddressPrefix}'
CONTEXT: Show effective NSG deny rules on an Azure NIC for traffic debugging
SOURCE: 06-cloud-security/03-azure-security.md
---

TOOL: aws
CMD: aws ec2 modify-instance-metadata-options --instance-id i-1234567890abcdef0 --http-tokens required --http-put-response-hop-limit 1 --http-endpoint enabled
CONTEXT: Enforce IMDSv2 on EC2 instance to block SSRF attacks against metadata endpoint
SOURCE: 06-cloud-security/07-owasp-api-security.md
---

TOOL: aws
CMD: aws ec2 modify-instance-metadata-options --instance-id $(curl -s http://169.254.169.254/latest/meta-data/instance-id) --http-tokens required
CONTEXT: Enforce IMDSv2 immediately after SSRF credential exposure incident
SOURCE: 06-cloud-security/07-owasp-api-security.md
---

TOOL: aws
CMD: aws sts revoke-session --reason "Credential exposure via SSRF"
CONTEXT: Revoke compromised role session after SSRF IAM credential leak
SOURCE: 06-cloud-security/07-owasp-api-security.md
---

TOOL: aws
CMD: aws ec2 describe-instances --instance-ids i-XXXXXXXXXXXXXXXXX --query 'Reservations[*].Instances[*].SecurityGroups'
CONTEXT: Get security groups attached to an EC2 instance
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-security-groups --group-ids sg-XXXXXXXXXXXXXXXXX --query 'SecurityGroups[*].IpPermissions'
CONTEXT: Check inbound rules on a target security group
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-security-groups --group-ids sg-XXXXXXXX --query 'SecurityGroups[*].IpPermissions'
CONTEXT: Check security group inbound rules during service-not-reachable debugging
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: aws
CMD: aws ec2 describe-vpc-peering-connections --filters "Name=status-code,Values=active" --query 'VpcPeeringConnections[*].{ID:VpcPeeringConnectionId,Status:Status.Code,Requester:RequesterVpcInfo.CidrBlock,Accepter:AccepterVpcInfo.CidrBlock}'
CONTEXT: List active VPC peering connections with CIDR info
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-transit-gateway-attachments --filters "Name=transit-gateway-id,Values=tgw-XXXXXXXX" --query 'TransitGatewayAttachments[*].{VPC:ResourceId,State:State}'
CONTEXT: Check TGW attachment states for cross-VPC connectivity debugging
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=subnet-XXXXXXXXX" --query 'RouteTables[*].Routes'
CONTEXT: Get routes for the route table associated with a specific subnet
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 create-route --route-table-id rtb-XXXXXXXX --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-XXXXXXXXXXXXXXXXX
CONTEXT: Add a VPC peering route to a route table
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 create-route --route-table-id rtb-XXXXXXXX --destination-cidr-block 10.1.0.0/16 --transit-gateway-id tgw-XXXXXXXXXXXXXXXXX
CONTEXT: Add a Transit Gateway route to a route table
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id tgw-rtb-XXXXXXXX --filters "Name=state,Values=active"
CONTEXT: Check TGW route table for active inter-VPC routes
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=subnet-XXXXXXXXX" --query 'NetworkAcls[*].{Entries:Entries,Associations:Associations}'
CONTEXT: Get NACL entries for a subnet (stateless — check both directions)
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-XXXXXXXXX --traffic-type ALL --log-group-name /vpc/flowlogs --deliver-logs-permission-arn arn:aws:iam::ACCOUNT:role/FlowLogsRole
CONTEXT: Enable VPC Flow Logs for all traffic to CloudWatch Logs
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-nat-gateways --nat-gateway-ids nat-XXXXXXXXXXXXXXXXX --query 'NatGateways[*].{State:State,SubnetId:SubnetId}'
CONTEXT: Check NAT Gateway state and subnet placement
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-nat-gateways --nat-gateway-ids nat-XXXXXXXXXXXXXXXXX --query 'NatGateways[*].{State:State,FailureMessage:FailureMessage}'
CONTEXT: Check NAT Gateway health and failure reason
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ssm start-session --target i-XXXXXXXXXXXXXXXXX
CONTEXT: SSH to private EC2 instance via SSM (no bastion needed)
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws directconnect describe-connections --connection-id dxcon-XXXXXXXX --query 'connections[*].{Name:connectionName,State:connectionState,Bandwidth:bandwidth}'
CONTEXT: Check Direct Connect physical connection state
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws directconnect describe-virtual-interfaces --query 'virtualInterfaces[*].{VIF:virtualInterfaceId,BGP:bgpPeers[*].bgpStatus,State:virtualInterfaceState}'
CONTEXT: Check Direct Connect VIF BGP peer status
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-XXXXXXXXX" --query 'RouteTables[*].Routes[?GatewayId!=`null`]'
CONTEXT: Check if DX/VPN routes are propagated to VPC route table
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 enable-vgw-route-propagation --route-table-id rtb-XXXXXXXX --gateway-id vgw-XXXXXXXX
CONTEXT: Enable VGW route propagation on a route table for DX/VPN
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-vpn-connections --vpn-connection-id vpn-XXXXXXXX --query 'VpnConnections[*].VgwTelemetry[*].{IP:OutsideIpAddress,Status:Status,Reason:StatusMessage}'
CONTEXT: Check VPN tunnel states and status messages
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-vpn-connections --vpn-connection-id vpn-XXXXXXXX --query 'VpnConnections[*].Routes'
CONTEXT: Check routes propagated via VPN connection
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=vpc-XXXXXXXXX" --query 'VpcEndpoints[*].{ID:VpcEndpointId,Service:ServiceName,State:State,Type:VpcEndpointType}'
CONTEXT: List VPC endpoints and their states
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-XXXXXXXXXXXXXXXXX --query 'VpcEndpoints[*].DnsEntries'
CONTEXT: Check DNS names for a VPC interface endpoint
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-XXXXXXXXXXXXXXXXX --query 'VpcEndpoints[*].Groups'
CONTEXT: Check security groups associated with an interface VPC endpoint
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: aws
CMD: aws elbv2 describe-target-health --target-group-arn <ARN>
CONTEXT: Check target health in AWS ALB target group
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: aws
CMD: aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkIn --dimensions Name=InstanceId,Value=i-XXXXXXXXXXXXXXXXX --start-time $(date -u -d '15 minutes ago' +%FT%TZ) --end-time $(date -u +%FT%TZ) --period 60 --statistics Average,Maximum
CONTEXT: Check EC2 network ingress metrics to detect volumetric DDoS attack
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name RequestCount --dimensions Name=LoadBalancer,Value=app/my-alb/XXXXXXXXXXXXXXXX --start-time $(date -u -d '15 minutes ago' +%FT%TZ) --end-time $(date -u +%FT%TZ) --period 60 --statistics Sum
CONTEXT: Check ALB request count spike as DDoS detection signal
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws guardduty list-findings --detector-id <DETECTOR_ID> --finding-criteria '{"Criterion":{"type":{"Eq":["UnauthorizedAccess:EC2/DDoSattack"]}}}'
CONTEXT: Check GuardDuty for DDoS-related findings
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws s3 cp s3://your-alb-logs-bucket/AWSLogs/ACCOUNT/elasticloadbalancing/region/YYYY/MM/DD/ /tmp/alb-logs/ --recursive
CONTEXT: Download ALB access logs for attack pattern analysis
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws s3 sync s3://alb-access-logs-bucket/AWSLogs/ACCOUNT/elasticloadbalancing/ /tmp/alb-logs/
CONTEXT: Sync all ALB access logs locally for DDoS investigation
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws shield create-subscription
CONTEXT: Enable AWS Shield Advanced (one-time setup)
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws shield create-protection --name "my-application" --resource-arn arn:aws:elasticloadbalancing:region:account:loadbalancer/app/my-alb/xxx
CONTEXT: Add Shield Advanced protection to an ALB
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws shield list-protections
CONTEXT: List all Shield Advanced protected resources
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws wafv2 create-rule-group --name "RateLimit" --scope REGIONAL --capacity 10 --rules '[{"Name": "IPRateLimit", "Priority": 1, "Statement": {"RateBasedStatement": {"Limit": 2000, "AggregateKeyType": "IP"}}, "Action": {"Block": {}}, "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "IPRateLimit"}}]'
CONTEXT: Create WAF rate-based rule group to block IPs exceeding 2000 req/5min
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws wafv2 update-ip-set --name "BlockedIPs" --scope REGIONAL --id <IPSET_ID> --addresses '["1.2.3.4/32","5.6.7.8/32"]' --lock-token <LOCK_TOKEN>
CONTEXT: Block specific attacking IPs by adding to WAF IP set during DDoS
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws wafv2 associate-web-acl --web-acl-arn <WEB_ACL_ARN> --resource-arn <ALB_ARN>
CONTEXT: Associate WAF Web ACL with ALB for bot protection
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: aws
CMD: aws wafv2 list-available-managed-rule-groups --scope REGIONAL
CONTEXT: List available AWS-managed WAF rule groups (bot control, OWASP, IP reputation)
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

## az

TOOL: az
CMD: az policy assignment create --name hipaa-hitrust-initiative --scope /subscriptions/$SUBSCRIPTION_ID --policy-set-definition /providers/Microsoft.Authorization/policySetDefinitions/a169a624-5599-4385-a696-c8d643089fab --enforce DoNotEnforce
CONTEXT: Apply HIPAA/HITRUST policy initiative to Azure subscription in audit mode
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: az
CMD: az policy state summarize --policy-assignment hipaa-hitrust-initiative | jq '.value[] | {resource: .resourceId, compliant: .complianceState}'
CONTEXT: Check HIPAA/HITRUST compliance state for each resource
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: az
CMD: az network nsg rule list --nsg-name <NSG_NAME> --resource-group <RG>
CONTEXT: List NSG rules for Azure network debugging
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: az
CMD: az network route-table route list --route-table-name <RT> --resource-group <RG>
CONTEXT: List routes in an Azure route table
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: az
CMD: az network vnet peering list --vnet-name <VNET> --resource-group <RG>
CONTEXT: List VNet peering connections for Azure connectivity debugging
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

## base64

TOOL: base64
CMD: echo "c3VwZXJzZWNyZXRwYXNzd29yZA==" | base64 -d
CONTEXT: Decode base64 Kubernetes Secret value - demonstrates base64 is not encryption
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: base64
CMD: kubectl get secret tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
CONTEXT: Decode and inspect TLS cert stored in a Kubernetes Secret
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

## calicoctl

TOOL: calicoctl
CMD: calicoctl get networkpolicies -A
CONTEXT: List all Calico network policies across all namespaces
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

## cat

TOOL: cat
CMD: cat /proc/sys/net/ipv4/tcp_syncookies
CONTEXT: Check if SYN cookies are enabled (DDoS SYN flood defense)
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: cat
CMD: cat /proc/sys/net/netfilter/nf_conntrack_count
CONTEXT: Check current conntrack table entry count
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: cat
CMD: cat /proc/sys/net/netfilter/nf_conntrack_max
CONTEXT: Check maximum conntrack table size
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: cat
CMD: cat /proc/net/softnet_stat
CONTEXT: Read per-CPU softirq drop and time-squeeze counters
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: cat
CMD: cat /sys/fs/cgroup/cpu/cpu.stat
CONTEXT: Check CPU throttling stats in container (throttled_time indicates cgroup limit)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: cat
CMD: cat /proc/<PID>/status | grep -i thread
CONTEXT: Check thread count for a process
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: cat
CMD: cat /etc/resolv.conf
CONTEXT: Check resolver IP and search domain configuration
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: cat
CMD: cat /proc/interrupts | grep -i eth0
CONTEXT: Check which CPUs handle network interrupts (IRQ affinity)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

## conntrack

TOOL: conntrack
CMD: conntrack -S 2>/dev/null
CONTEXT: Show conntrack statistics including insert_failed and drop counts
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: conntrack
CMD: conntrack -L | grep ESTABLISHED | wc -l
CONTEXT: Count ESTABLISHED conntrack entries to detect leaks
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: conntrack
CMD: conntrack -C
CONTEXT: Show current conntrack table entry count
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

## curl

TOOL: curl
CMD: curl -v http://service-name:8080/health
CONTEXT: Verbose connection test to get exact error (refused vs timeout vs TLS)
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: curl
CMD: curl -v --connect-timeout 5 --max-time 10 http://service-name:8080/health
CONTEXT: Timed connection test to avoid hanging during service reachability debugging
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: curl
CMD: curl -s -o /dev/null -w "%{http_code}" http://localhost:<PORT>/health
CONTEXT: Test local service health bypassing network path (60-second triage)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: curl
CMD: curl -w "\ndns_lookup:     %{time_namelookup}s\ntcp_connect:    %{time_connect}s\ntls_handshake:  %{time_appconnect}s\ntime_to_first_byte: %{time_starttransfer}s\ntotal_time:     %{time_total}s\n" -o /dev/null -s https://service:443/health
CONTEXT: Break down request latency by phase (DNS / TCP / TLS / TTFB / total)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: curl
CMD: curl -w "dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" -o /dev/null -s https://service:443/health
CONTEXT: Per-request latency breakdown for HTTP/HTTPS service (run 10x for distribution)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: curl
CMD: curl -w "dns:%{time_namelookup} tcp:%{time_connect} ttfb:%{time_starttransfer} total:%{time_total}\n" -o /dev/null -s http://service:8080/health
CONTEXT: Latency breakdown for plain HTTP service (no TLS)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: curl
CMD: curl -v https://service:443/health 2>&1 | grep -E "SSL|error|certificate|expired|verify"
CONTEXT: Quick TLS error extraction from curl verbose output
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: curl
CMD: curl -s "https://crt.sh/?q=payment-api.mycompany.com&output=json" | jq '.[] | {cn: .common_name, issuer: .issuer_name, expiry: .not_after}'
CONTEXT: Certificate transparency log check for payment API certificate history
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: curl
CMD: curl -s --connect-timeout 3 http://10.1.0.50:8080
CONTEXT: PCI segmentation test — CDE should NOT reach non-CDE web server
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: curl
CMD: curl -s --connect-timeout 3 http://169.254.169.254/latest/meta-data/
CONTEXT: PCI segmentation test — CDE should NOT reach EC2 metadata (IMDSv2 blocks)
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: curl
CMD: curl -I https://api.mycompany.com/api/v1/health | grep -E "Strict-Transport|X-Content-Type|X-Frame|Content-Security|Access-Control"
CONTEXT: Check HTTP security headers on API endpoint
SOURCE: 06-cloud-security/07-owasp-api-security.md
---

TOOL: curl
CMD: curl -H "Authorization: Bearer $ATTACKER_TOKEN" https://api.mycompany.com/api/v1/users/$VICTIM_ID/orders
CONTEXT: Test BOLA - cross-user resource access should return 403
SOURCE: 06-cloud-security/07-owasp-api-security.md
---

TOOL: curl
CMD: curl -s --connect-timeout 5 http://checkip.amazonaws.com
CONTEXT: Test internet connectivity from private subnet via NAT Gateway
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

TOOL: curl
CMD: curl -H "Host: hostname.example.com" http://<INGRESS_IP>/path
CONTEXT: Test K8s ingress routing using Host header simulation
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: curl
CMD: curl -v http://<CLUSTER_IP>:8080/health
CONTEXT: Test K8s ClusterIP directly to isolate kube-proxy vs DNS issue
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: curl
CMD: curl -v http://service-name.namespace.svc.cluster.local:8080/health
CONTEXT: Test K8s service DNS from inside the mesh
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: curl
CMD: curl -X POST https://api.mycompany.com/api/v1/webhooks -H "Authorization: Bearer $TOKEN" -d '{"callback_url": "http://169.254.169.254/latest/meta-data/"}'
CONTEXT: Test SSRF vulnerability in webhook endpoint (staging only)
SOURCE: 06-cloud-security/07-owasp-api-security.md
---

## dig

TOOL: dig
CMD: dig +short service-name
CONTEXT: Quick DNS resolution check from affected host
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig service-name
CONTEXT: Full DNS response showing status, flags, TTL, and resolver used
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig +short service-name.namespace.svc.cluster.local
CONTEXT: Test K8s DNS resolution with FQDN from inside cluster
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: dig
CMD: dig NS example.com +short
CONTEXT: Find authoritative name servers for a domain
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig @ns1.example.com service.example.com +short
CONTEXT: Query authoritative NS directly to bypass recursive cache
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig +trace service.example.com
CONTEXT: Trace full DNS delegation chain from root to answer
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig @8.8.8.8 service.example.com
CONTEXT: Test DNS against public resolver to bypass internal resolver
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig @1.1.1.1 service.example.com
CONTEXT: Test DNS against Cloudflare public resolver
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig service.example.com | grep -A5 "AUTHORITY SECTION"
CONTEXT: Check SOA MINIMUM field for negative cache TTL
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig +cd service.example.com
CONTEXT: Test DNS with DNSSEC checking disabled to diagnose DNSSEC failures
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig +dnssec +trace service.example.com
CONTEXT: Trace DNSSEC validation chain to find expired RRSIG or missing DS
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig +dnssec service.example.com
CONTEXT: Check if DNS response has valid DNSSEC signature (RRSIG + ad flag)
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig DS example.com
CONTEXT: Check DS (delegation signer) record in parent zone for DNSSEC
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: dig @10.96.0.10 service-name.namespace.svc.cluster.local
CONTEXT: Query CoreDNS directly to test K8s DNS
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: dig
CMD: time dig +short service.namespace.svc.cluster.local
CONTEXT: Measure DNS resolution time to check if DNS is the latency bottleneck
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: dig
CMD: dig +short s3.amazonaws.com
CONTEXT: Test if S3 VPC endpoint DNS is resolving to private IP
SOURCE: 07-debugging-playbooks/07-cloud-connectivity-issues.md
---

## dmesg

TOOL: dmesg
CMD: dmesg | grep "nf_conntrack: table full"
CONTEXT: Check for conntrack table overflow kernel messages
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: dmesg
CMD: dmesg | grep "SYN flooding"
CONTEXT: Check if kernel has detected SYN flooding and activated SYN cookies
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: dmesg
CMD: dmesg | grep -i "eth0\|ens\|bond\|dropped\|error" | tail -20
CONTEXT: Check driver-level network error messages
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: dmesg
CMD: dmesg | grep 'oom-kill'
CONTEXT: Check for OOM kills causing memory-related latency
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: dmesg
CMD: dmesg | grep eth0
CONTEXT: Check kernel messages for network interface errors (Layer 1 debugging)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

## ethtool

TOOL: ethtool
CMD: ethtool eth0
CONTEXT: Check link speed and duplex (physical layer health)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: ethtool
CMD: ethtool -S eth0
CONTEXT: Show NIC driver statistics including rx_missed_errors and CRC errors
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: ethtool
CMD: ethtool -S eth0 | grep -E -i '(drop|miss|error|discard|overflow|lost)'
CONTEXT: Check all NIC error and drop counters for packet loss at NIC level
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: ethtool
CMD: ethtool -g eth0
CONTEXT: Check current vs maximum NIC ring buffer sizes
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: ethtool
CMD: ethtool -G eth0 rx 4096
CONTEXT: Increase NIC rx ring buffer to reduce packet drops at high traffic rates
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

## free

TOOL: free
CMD: free -m
CONTEXT: Check memory pressure (swap usage indicates severe latency risk)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## git

TOOL: git
CMD: git log --oneline -20
CONTEXT: Check recent code changes to correlate with incident start time
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: git
CMD: git log --all --full-history -- "*.env" "*.yaml" "*.json"
CONTEXT: Scan git history for accidentally committed secret files
SOURCE: 06-cloud-security/06-secrets-management.md
---

## gitleaks

TOOL: gitleaks
CMD: gitleaks detect --source . --log-level warn
CONTEXT: Scan source code for accidentally committed secrets
SOURCE: 06-cloud-security/06-secrets-management.md
---

## helm

TOOL: helm
CMD: helm repo add stakater https://stakater.github.io/stakater-charts
CONTEXT: Add Stakater Helm repo for Reloader installation
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: helm
CMD: helm install reloader stakater/reloader -n reloader --create-namespace
CONTEXT: Install Stakater Reloader to trigger rolling restarts on secret updates
SOURCE: 06-cloud-security/06-secrets-management.md
---

## iperf3

TOOL: iperf3
CMD: iperf3 -c <SERVER_IP> -t 60
CONTEXT: Generate background load to test for bufferbloat (latency under load)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## iostat

TOOL: iostat
CMD: iostat -x 1 5
CONTEXT: Check disk I/O utilization and await time as latency source
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## ip

TOOL: ip
CMD: ip route get <SERVER_IP>
CONTEXT: Check if routing table has a route to destination
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: ip
CMD: ip route show
CONTEXT: Show full routing table to check CNI pod subnet routes
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: ip
CMD: ip -d link show flannel.1
CONTEXT: Check Flannel VXLAN tunnel interface details
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: ip
CMD: ip neigh
CONTEXT: Show ARP table for Layer 2 neighbor resolution debugging
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

## iptables

TOOL: iptables
CMD: iptables -L INPUT -n -v | grep 8080
CONTEXT: Check for iptables rules blocking a specific port (60-second triage)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: iptables
CMD: iptables -L INPUT -n -v | grep <PORT>
CONTEXT: Check for iptables DROP or REJECT rules on specific port
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: iptables
CMD: iptables -L FORWARD -n -v | grep 8080
CONTEXT: Check FORWARD chain for rules blocking forwarded traffic
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: iptables
CMD: iptables -L -n -v | grep -E "(DROP|REJECT)"
CONTEXT: List all active DROP and REJECT rules with packet counts
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: iptables
CMD: iptables -L -n -v
CONTEXT: List all iptables rules with packet counters (non-zero = active)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: iptables
CMD: iptables -L -n -v | grep -E "DROP|REJECT" | awk '$1 != "0" || $2 != "0"'
CONTEXT: Show only active DROP/REJECT rules (non-zero packet counts)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: iptables
CMD: iptables -t nat -L -n -v
CONTEXT: List NAT table rules (PREROUTING, POSTROUTING)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: iptables
CMD: iptables -t mangle -L -n -v
CONTEXT: List mangle table rules (QoS, packet marking)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: iptables
CMD: iptables -t filter -L -n -v
CONTEXT: List filter table rules (main drop location)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: iptables
CMD: iptables -t nat -L KUBE-SERVICES -n | grep <CLUSTER_IP>
CONTEXT: Check kube-proxy NAT rules exist for a K8s ClusterIP service
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: iptables
CMD: iptables -t nat -L KUBE-SVC-<HASH> -n
CONTEXT: Follow kube-proxy service chain to see endpoint load-balancing rules
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: iptables
CMD: iptables -t nat -L KUBE-SERVICES -n | grep <CLUSTER_IP>
CONTEXT: Verify kube-proxy KUBE-SERVICES chain has entry for ClusterIP
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: iptables
CMD: iptables -t nat -L KUBE-SVC-<HASH> -n -v
CONTEXT: Check kube-proxy service chain to see KUBE-SEP (endpoint) entries
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: iptables
CMD: iptables -t nat -L KUBE-SEP-<HASH> -n -v
CONTEXT: Check kube-proxy endpoint DNAT rule (shows pod IP:port destination)
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: iptables
CMD: iptables -t filter -L KUBE-FIREWALL -n -v
CONTEXT: Check kube-proxy KUBE-FIREWALL chain for K8s-level packet drops
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: iptables
CMD: iptables -I INPUT -p tcp --syn -m limit --limit 100/s --limit-burst 200 -j ACCEPT
CONTEXT: Rate limit TCP SYN packets to 100/s (burst 200) during SYN flood
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: iptables
CMD: iptables -A INPUT -p tcp --syn -j DROP
CONTEXT: Drop SYN packets exceeding rate limit (used after rate-limit ACCEPT rule)
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

## ipvsadm

TOOL: ipvsadm
CMD: ipvsadm -Ln | grep -A3 <CLUSTER_IP>
CONTEXT: Check IPVS virtual server and real server mappings for K8s service
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: ipvsadm
CMD: ipvsadm -Ln | grep -A5 <CLUSTER_IP>
CONTEXT: Check IPVS routing for K8s ClusterIP in IPVS mode kube-proxy
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

## journalctl

TOOL: journalctl
CMD: journalctl -u service --since "30 minutes ago"
CONTEXT: Check systemd service logs for recent errors at start of incident
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: journalctl
CMD: journalctl -u service-name --since "10 minutes ago"
CONTEXT: Check application logs for errors during service-not-reachable debugging
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: journalctl
CMD: journalctl -u kubelet --since "30 minutes ago" | grep -i "cni\|network"
CONTEXT: Check kubelet logs for CNI errors during K8s networking issues
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

## kill

TOOL: kill
CMD: kill -3 <PID>
CONTEXT: Generate Java thread dump to diagnose thread pool exhaustion causing latency
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## kubectl

TOOL: kubectl
CMD: kubectl rollout history deployment/app
CONTEXT: Check K8s deployment history to correlate incident with recent rollout
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: kubectl
CMD: kubectl describe pod <pod-name> | grep -A5 "Events:"
CONTEXT: Check K8s pod events to understand what changed recently
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: kubectl
CMD: kubectl get pod -l app=service-name -o wide
CONTEXT: Check pod status and node placement for service reachability debugging
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl logs <pod-name> --previous
CONTEXT: Get logs from crashed container (CrashLoopBackOff debugging)
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl describe pod <pod-name>
CONTEXT: Show pod events and conditions for debugging pod failures
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl exec -it <pod-name> -- ss -tnlp
CONTEXT: Check listening ports inside a K8s pod
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl get endpoints service-name -n namespace
CONTEXT: Check if service has healthy endpoints (most common K8s service issue)
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl get svc service-name -n namespace -o jsonpath='{.spec.selector}'
CONTEXT: Get service selector to verify it matches pod labels
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl get svc service-name -n target-namespace -o jsonpath='{.spec.clusterIP}'
CONTEXT: Get ClusterIP to test service connectivity bypassing DNS
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl get networkpolicy -n <namespace>
CONTEXT: List NetworkPolicies in a namespace to find default-deny or blocking rules
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl describe networkpolicy <policy-name> -n <namespace>
CONTEXT: Show NetworkPolicy rules (pod selector, ingress/egress)
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl get networkpolicy -A -o yaml | grep -A5 "podSelector: {}"
CONTEXT: Find default-deny NetworkPolicies (empty podSelector = applies to all pods)
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl exec -n kube-system <cilium-pod> -- cilium policy get
CONTEXT: Check Cilium-specific network policies active in the cluster
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl top pod <pod-name>
CONTEXT: Check pod CPU and memory usage (detect throttling or OOM)
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl describe pod <pod-name> | grep -A3 "Limits:\|Requests:"
CONTEXT: Check resource limits and requests on a pod
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: kubectl
CMD: kubectl top pod -n kube-system -l k8s-app=kube-dns
CONTEXT: Check CoreDNS CPU/memory usage (high CPU = DNS overloaded)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: kubectl
CMD: kubectl exec <pod> -- cat /etc/resolv.conf
CONTEXT: Check pod DNS resolver config and ndots setting
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: kubectl
CMD: kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50 | grep -i "slow\|error\|timeout"
CONTEXT: Check CoreDNS for slow queries or errors as latency source
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: kubectl
CMD: kubectl get pod <pod-name> -o wide
CONTEXT: Get pod IP and node placement for latency/networking debugging
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: kubectl
CMD: kubectl describe node <node-name> | grep -A5 "Conditions:"
CONTEXT: Check node for MemoryPressure or DiskPressure affecting pod latency
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: kubectl
CMD: kubectl describe hpa <hpa-name>
CONTEXT: Check HPA current vs desired replicas (scale-up lag causes latency spikes)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: kubectl
CMD: kubectl exec <pod-name> -- cat /etc/resolv.conf
CONTEXT: Check pod DNS config (ndots:5 causes 5x DNS queries for external names)
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl exec <pod-name> -- nslookup service-name
CONTEXT: Test DNS + search domain expansion from inside a pod
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl exec <pod-name> -- nslookup service-name.namespace.svc.cluster.local
CONTEXT: Test FQDN DNS resolution from inside a pod (bypasses ndots expansion)
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl get svc -n kube-system kube-dns
CONTEXT: Get CoreDNS ClusterIP address for direct DNS testing
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl exec <pod-name> -- dig @10.96.0.10 service-name.namespace.svc.cluster.local
CONTEXT: Query CoreDNS directly to isolate DNS failure from pod config
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
CONTEXT: Check CoreDNS logs for NXDOMAIN, SERVFAIL, or upstream failures
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl get configmap coredns -n kube-system -o yaml
CONTEXT: Check CoreDNS configuration (upstream resolvers, forward plugin)
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl get pod -n kube-system -l k8s-app=kube-dns
CONTEXT: Check CoreDNS pod status
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl top pod -n kube-system -l k8s-app=kube-dns
CONTEXT: Check CoreDNS resource usage for overload detection
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl rollout restart deployment/coredns -n kube-system
CONTEXT: Restart CoreDNS to flush stale DNS cache entries
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: kubectl
CMD: kubectl describe certificate <cert-name> -n namespace
CONTEXT: Check cert-manager Certificate status, conditions, and renewal time
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: kubectl
CMD: kubectl get certificaterequest -n namespace
CONTEXT: List CertificateRequests to see pending or failed cert issuance
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: kubectl
CMD: kubectl describe certificaterequest <name> -n namespace
CONTEXT: Get detailed failure reason for a cert-manager CertificateRequest
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: kubectl
CMD: kubectl describe secret <tls-secret-name> -n namespace
CONTEXT: Verify TLS secret contains tls.crt and tls.key
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: kubectl
CMD: kubectl get secret <tls-secret-name> -n namespace -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -subject -dates -ext subjectAltName
CONTEXT: Inspect the TLS cert stored in a K8s secret
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: kubectl
CMD: kubectl logs -n cert-manager deployment/cert-manager | grep -i "cert-name\|error\|failed" | tail -50
CONTEXT: Check cert-manager logs for certificate issuance errors
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: kubectl
CMD: kubectl get certificate -A
CONTEXT: List all certificates and their Ready status across all namespaces
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: kubectl
CMD: kubectl get peerauthentication -A
CONTEXT: Check Istio PeerAuthentication policies (STRICT vs PERMISSIVE mTLS mode)
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: kubectl
CMD: kubectl get pod -o wide -n <namespace>
CONTEXT: Get pod IPs and node placement for pod-to-pod connectivity debugging
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl run netdebug --image=nicolaka/netshoot --rm -it --restart=Never -- bash
CONTEXT: Create ephemeral debug pod with network tools for K8s debugging
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl exec -it pod-a -- ping -c3 <POD_B_IP>
CONTEXT: Test pod-to-pod ICMP connectivity
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl exec -it pod-a -- nc -zv <POD_B_IP> <PORT>
CONTEXT: Test pod-to-pod TCP connectivity on specific port
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl exec -it pod-a -- curl http://<POD_B_IP>:<PORT>/health
CONTEXT: Test HTTP connectivity between pods using pod IP directly
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl describe node <node-name> | grep -A5 "PodCIDR"
CONTEXT: Check if CNI has assigned pod CIDR to a node
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get pod -n kube-system -l k8s-app=flannel -o wide
CONTEXT: Check Flannel CNI pods are running on every node
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get pod -n kube-system -l k8s-app=calico-node -o wide
CONTEXT: Check Calico CNI pods are running on every node
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get pod -n kube-system -l k8s-app=cilium -o wide
CONTEXT: Check Cilium CNI pods are running on every node
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl logs -n kube-system -l app=flannel --tail=100
CONTEXT: Check Flannel CNI logs for network errors
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl logs -n kube-system -l k8s-app=calico-node --tail=100
CONTEXT: Check Calico CNI logs for network errors
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl logs -n kube-system -l k8s-app=cilium --tail=100
CONTEXT: Check Cilium CNI logs for network errors
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl exec -it pod-a -- ping -M do -s 1400 <POD_B_IP>
CONTEXT: Test MTU with 1400-byte ping (detects overlay MTU mismatch)
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl exec -it pod-a -- ping -M do -s 1450 <POD_B_IP>
CONTEXT: Test MTU at expected pod limit (1450 for VXLAN overlay)
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get svc <service-name> -n <namespace> -o yaml
CONTEXT: Get full service spec including selector and ports
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get svc <service-name> -n <namespace> -o jsonpath='{.spec.selector}'
CONTEXT: Get service selector to match against pod labels
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get endpoints <service-name> -n <namespace>
CONTEXT: Check service endpoints (empty = no matching ready pods)
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl describe svc <service-name> | grep Selector
CONTEXT: Get service selector for comparing to pod labels
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl describe pod <pod-name> | grep -A10 "Conditions:"
CONTEXT: Check pod readiness conditions (failing readiness = no endpoints)
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl describe pod <pod-name> | grep -A10 "Readiness:"
CONTEXT: Check readiness probe configuration and failure reason
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get svc <service-name> -n <namespace> -o jsonpath='{.spec.clusterIP}'
CONTEXT: Get ClusterIP to test connectivity bypassing CoreDNS
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl exec -it <debug-pod> -- curl http://10.96.45.123:<PORT>/health
CONTEXT: Test K8s service via ClusterIP directly (bypasses kube-proxy DNS)
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get pod -n kube-system -l k8s-app=kube-proxy -o wide
CONTEXT: Check kube-proxy pods are running on all nodes
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl logs -n kube-system -l k8s-app=kube-proxy | grep -i "error\|failed" | tail -30
CONTEXT: Check kube-proxy logs for rule sync errors
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl rollout restart daemonset/kube-proxy -n kube-system
CONTEXT: Force kube-proxy to resync iptables/IPVS rules
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get svc <service-name> -n <namespace> -o wide
CONTEXT: Check service EXTERNAL-IP status (pending = cloud LB not provisioned)
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get pod -n metallb-system -o wide
CONTEXT: Check MetalLB speaker pods for bare metal LoadBalancer service
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get ingress -n <namespace>
CONTEXT: Check ingress resource address assignment
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl describe ingress <ingress-name> -n <namespace>
CONTEXT: Show ingress rules and address for external traffic debugging
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get pod -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
CONTEXT: Check nginx ingress controller pod status
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl logs -n ingress-nginx <controller-pod> | tail -50
CONTEXT: Check nginx ingress controller logs for routing errors
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get networkpolicy -A
CONTEXT: List all NetworkPolicies across all namespaces
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get networkpolicy -n <namespace> -o yaml | grep -A5 "podSelector: {}"
CONTEXT: Find default-deny NetworkPolicies applying to all pods in a namespace
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl exec -n kube-system <cilium-pod> -- cilium monitor --type drop
CONTEXT: Monitor real-time Cilium packet drops with policy reason
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get pod -n kube-system -l k8s-app=kube-dns -o wide
CONTEXT: Check CoreDNS pod status and node placement
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
CONTEXT: Check CoreDNS logs for plugin errors, SERVFAIL, or upstream failures
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl run dnstest --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default.svc.cluster.local $COREDNS_IP
CONTEXT: Test CoreDNS directly with an ephemeral pod
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl scale deployment coredns -n kube-system --replicas=4
CONTEXT: Scale CoreDNS replicas to handle DNS overload
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: kubectl
CMD: kubectl get clusterrolebindings -o json | jq '.items[] | select(.roleRef.name == "cluster-admin") | {name: .metadata.name, subjects: .subjects}'
CONTEXT: PCI compliance check - find all cluster-admin role bindings
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

TOOL: kubectl
CMD: kubectl get secrets -A -o json | jq '.items[] | {name: .metadata.name, ns: .metadata.namespace, keys: (.data | keys)}'
CONTEXT: Audit all Kubernetes secrets and their keys across all namespaces
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: kubectl
CMD: kubectl create secret generic db-creds --from-literal=password=supersecretpassword
CONTEXT: Create a Kubernetes secret (demonstrates base64 encoding is not encryption)
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: kubectl
CMD: kubectl get secret db-creds -o yaml
CONTEXT: View Kubernetes secret in YAML (shows base64-encoded data)
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: kubectl
CMD: kubectl describe externalsecret payments-db-secret -n payments
CONTEXT: Check External Secret sync status and conditions
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: kubectl
CMD: kubectl logs -n external-secrets deploy/external-secrets --since=1h | grep -E "ERROR|WARN"
CONTEXT: Check ESO controller logs for secret sync errors
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: kubectl
CMD: kubectl get secret payments-db-creds -n payments -o jsonpath='{.data}' | jq 'to_entries[] | {key: .key, length: (.value | @base64d | length)}'
CONTEXT: Verify K8s secret was created with expected keys and non-empty values
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: kubectl
CMD: kubectl exec -n external-secrets deploy/external-secrets -- aws secretsmanager get-secret-value --secret-id prod/payments/database
CONTEXT: Test AWS Secrets Manager access from ESO's pod identity
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: kubectl
CMD: kubectl get deployment payments-api -n payments -o jsonpath='{.metadata.annotations}' | jq .
CONTEXT: Verify Reloader annotation is set on deployment for auto-restart on secret change
SOURCE: 06-cloud-security/06-secrets-management.md
---

## ls

TOOL: ls
CMD: ls /proc/<PID>/fd | wc -l
CONTEXT: Count open file descriptors for a process (high count = fd leak)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: ls
CMD: ls /proc/<PID>/task | wc -l
CONTEXT: Count actual thread count for a process
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

## mtr

TOOL: mtr
CMD: mtr -n --report --report-cycles 100 <SERVER_IP>
CONTEXT: Continuous path analysis with per-hop loss statistics (100 cycles)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## mysql

TOOL: mysql
CMD: time mysql -h <DB_HOST> -e "SELECT 1" 2>/dev/null
CONTEXT: Measure database connection latency directly
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## nc

TOOL: nc
CMD: nc -zv -w5 <SERVER_IP> <PORT>
CONTEXT: Test TCP connectivity from client to server (60-second triage step)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: nc
CMD: nc -uzv <nameserver-ip> 53
CONTEXT: Test UDP port 53 reachability for DNS timeout debugging
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: nc
CMD: for port in 5432 3306 6379 8443 22 3389; do result=$(nc -zv -w3 10.2.0.100 $port 2>&1); echo "Port $port: $result"; done
CONTEXT: PCI segmentation test - verify non-CDE cannot reach CDE ports
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

## netstat

TOOL: netstat
CMD: netstat -s | head -30
CONTEXT: Show protocol statistics since boot as baseline observation
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: netstat
CMD: netstat -s | grep -E "(receive errors|failed|overflow|dropped)"
CONTEXT: Check for packet receive errors and drops in protocol stats
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: netstat
CMD: netstat -s | grep -i "retransmit"
CONTEXT: Check TCP retransmit counters (compare two readings to calculate rate)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## nft

TOOL: nft
CMD: nft list ruleset | grep -B5 -A2 "8080"
CONTEXT: Check nftables rules matching a specific port
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: nft
CMD: nft list ruleset | grep -B5 "drop\|reject"
CONTEXT: Find nftables drop/reject rules during packet drop debugging
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

## nmap

TOOL: nmap
CMD: nmap -p 8080 <SERVER_IP>
CONTEXT: Probe port from client to detect filtered (firewall) vs closed vs open
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: nmap
CMD: nmap --script ssl-enum-ciphers -p 443 payment-api.mycompany.com | grep -E "TLSv1\.[0-1]"
CONTEXT: PCI Req 4 - verify no TLS 1.0/1.1 ciphers on payment API
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

## nslookup

TOOL: nslookup
CMD: nslookup service-name
CONTEXT: Quick DNS resolution check including search domain expansion
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

## nstat

TOOL: nstat
CMD: nstat -az | grep -i drop
CONTEXT: Check all kernel drop counters to confirm packet loss is occurring
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: nstat
CMD: nstat -az | grep -i retrans
CONTEXT: Check TCP retransmit counters for packet loss evidence
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: nstat
CMD: nstat -az | grep -E 'TcpExt(Listen|Backlog|Prune|RcvPruned|OFO|TCPSchedulerFailed)'
CONTEXT: Check socket-level drop counters (listen overflow, backlog drops, prune)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: nstat
CMD: nstat -az | grep NfConntrackFull
CONTEXT: Check if conntrack table is full (drops new connections)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: nstat
CMD: nstat -az | grep UdpInErrors
CONTEXT: Check UDP receive buffer overflow (common for DNS/syslog/metrics)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: nstat
CMD: nstat -az | grep TcpExtSyncookiesSent
CONTEXT: Check SYN cookie activation rate during SYN flood
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: nstat
CMD: nstat -az | grep -i "TcpExtListenOverflow\|TcpExtTCPSynRetrans"
CONTEXT: Check SYN queue overflow and SYN retransmit counters during SYN flood
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: nstat
CMD: nstat -az 2>/dev/null | head -20
CONTEXT: Quick overview of kernel network counters at start of incident
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

## openssl

TOOL: openssl
CMD: openssl s_client -connect service:443 -servername service </dev/null
CONTEXT: Get TLS certificate and handshake details for a service
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl s_client -connect ingress-ip:443 -servername hostname.example.com </dev/null
CONTEXT: Test TLS for a specific SNI hostname on an ingress IP
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null | openssl x509 -noout -text
CONTEXT: Get full certificate details from a live server
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl x509 -noout -text -in cert.pem
CONTEXT: Inspect a certificate file (subject, SANs, validity, issuer)
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl x509 -noout -subject -issuer -dates -ext subjectAltName -in cert.pem
CONTEXT: Quick cert summary showing expiry and SANs
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null | openssl x509 -noout -dates
CONTEXT: Check certificate expiry dates from live server
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: echo | openssl s_client -connect host:443 -servername host 2>/dev/null | openssl x509 -noout -checkend 0 && echo "cert valid" || echo "cert EXPIRED"
CONTEXT: Script-friendly cert expiry check (exit code indicates validity)
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl x509 -noout -checkend 86400 -in cert.pem && echo "OK" || echo "expires within 24h"
CONTEXT: Check if a cert expires within the next 24 hours
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null | openssl x509 -noout -ext subjectAltName
CONTEXT: List all Subject Alternative Names for hostname mismatch debugging
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -servername host -showcerts </dev/null 2>/dev/null
CONTEXT: Get full certificate chain sent by server (check for missing intermediates)
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt cert-chain.pem
CONTEXT: Verify certificate chain against system CA bundle
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl verify -CAfile root-ca.pem -untrusted intermediate.pem leaf.pem
CONTEXT: Manually verify cert chain with explicit CA and intermediate files
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl s_client -connect host:443 -cert client.crt -key client.key </dev/null
CONTEXT: Test mTLS connection with client certificate
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl x509 -noout -subject -issuer -dates -in client.crt
CONTEXT: Inspect client certificate details for mTLS debugging
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl verify -CAfile server-trusted-ca.pem client.crt
CONTEXT: Verify client cert is signed by the CA the server trusts for mTLS
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: kubectl exec <pod> -c istio-proxy -- openssl s_client -connect <service>:443 -cert /etc/certs/cert-chain.pem -key /etc/certs/key.pem -CAfile /etc/certs/root-cert.pem
CONTEXT: Test mTLS connection through Istio sidecar proxy
SOURCE: 07-debugging-playbooks/04-tls-certificate-issues.md
---

TOOL: openssl
CMD: openssl s_client -connect payment-api.mycompany.com:443 -tls1_1 2>&1 | grep "Secure Renegotiation"
CONTEXT: PCI Req 4 - verify TLS 1.1 is not accepted on payment endpoint
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

## pdns_control

TOOL: pdns_control
CMD: pdns_control purge service.example.com
CONTEXT: Flush PowerDNS cache for a specific record
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

## perf

TOOL: perf
CMD: perf
CONTEXT: Profile application for CPU performance issues causing latency
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

## ping

TOOL: ping
CMD: ping -c3 SERVER_IP
CONTEXT: Basic L3 reachability test during connection timeout debugging
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: ping
CMD: ping -c 30 <SERVER_IP>
CONTEXT: Check latency under background load to detect bufferbloat
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## ps

TOOL: ps
CMD: ps aux | grep service-name
CONTEXT: Check if a process is running during service reachability debugging
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

## rndc

TOOL: rndc
CMD: rndc flush
CONTEXT: Flush BIND DNS resolver cache
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

## sar

TOOL: sar
CMD: sar -n DEV 1 5
CONTEXT: Check network interface packet rate and bandwidth (detect volumetric DDoS)
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

## ss

TOOL: ss
CMD: ss -tnlp | grep <PORT>
CONTEXT: Check if process is listening on a port (60-second triage first step)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: ss
CMD: ss -tnlp | grep 8080
CONTEXT: Check listening sockets on port 8080 with process details
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: ss
CMD: ss -tnlp
CONTEXT: List all TCP listening sockets with process names and PIDs
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: ss
CMD: ss -tnp | grep 8080 | wc -l
CONTEXT: Count active connections to a port (detect connection exhaustion)
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: ss
CMD: ss -s
CONTEXT: Socket summary showing TCP state counts (SYN_RECV count detects SYN flood)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: ss
CMD: ss -ti dst <SERVER_IP>
CONTEXT: Check TCP connection details including retransmits and RTT to specific server
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: ss
CMD: ss -ltn | grep <PORT>
CONTEXT: Check listen queue depth (Recv-Q) vs backlog (Send-Q) for accept overflow
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: ss
CMD: ss -tnmp | head -30
CONTEXT: Check Recv-Q for all sockets (non-zero = app not reading fast enough)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: ss
CMD: ss -tnmp | grep -v "Recv-Q:0\|Recv-Q 0"
CONTEXT: Find sockets with non-empty receive queues (back-pressure from slow app)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: ss
CMD: ss -s | grep -i "SYN_RECV"
CONTEXT: Check SYN_RECV socket count to confirm SYN flood
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: ss
CMD: ss -tnp | awk '{print $2}' | sort | uniq -c | sort -rn | head -5
CONTEXT: Show TCP connection state distribution (detect CLOSE_WAIT/TIME_WAIT accumulation)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## strace

TOOL: strace
CMD: strace -p <PID> -e trace=read,recv,recvfrom,recvmsg 2>&1 | head -50
CONTEXT: Trace socket read calls to diagnose slow application reading (Recv-Q high)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

## swag

TOOL: swag
CMD: swag init
CONTEXT: Generate OpenAPI spec from Go source code for API inventory management
SOURCE: 06-cloud-security/07-owasp-api-security.md
---

## sysctl

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_syncookies=1
CONTEXT: Enable SYN cookies for SYN flood defense
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_syn_retries=2
CONTEXT: Reduce SYN retry count to free SYN queue faster during flood
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: sysctl
CMD: sysctl -w net.ipv4.tcp_synack_retries=2
CONTEXT: Reduce SYN-ACK retry count for unanswered connections during SYN flood
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: sysctl
CMD: sysctl net.core.netdev_max_backlog
CONTEXT: Check current netdev_max_backlog queue size
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: sysctl
CMD: sysctl -w net.core.netdev_max_backlog=10000
CONTEXT: Increase netdev backlog to prevent softirq drop at high packet rates
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: sysctl
CMD: sysctl net.core.netdev_budget
CONTEXT: Check softirq packet budget per poll cycle
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: sysctl
CMD: sysctl net.core.netdev_budget_usecs
CONTEXT: Check softirq time budget per poll cycle
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: sysctl
CMD: sysctl -w net.core.netdev_budget_usecs=8000
CONTEXT: Increase softirq time budget to reduce time_squeeze drops
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_max=262144
CONTEXT: Increase conntrack table size to prevent connection drops
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: sysctl
CMD: sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
CONTEXT: Reduce conntrack TIME_WAIT timeout to reclaim table entries faster
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: sysctl
CMD: sysctl -w net.core.somaxconn=65535
CONTEXT: Increase system listen backlog limit to prevent accept queue overflow
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: sysctl
CMD: sysctl -w net.core.rmem_max=26214400
CONTEXT: Increase max socket receive buffer size to prevent UDP drops
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: sysctl
CMD: sysctl -w net.core.rmem_default=26214400
CONTEXT: Increase default socket receive buffer size for UDP workloads
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

## systemctl

TOOL: systemctl
CMD: systemctl status service-name
CONTEXT: Check systemd service status during service-not-reachable debugging
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: systemctl
CMD: systemctl status svc
CONTEXT: Check service running state (5-layer model first check)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

## systemd-resolve

TOOL: systemd-resolve
CMD: systemd-resolve --flush-caches
CONTEXT: Flush systemd-resolved DNS cache after record change
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

## tcpdump

TOOL: tcpdump
CMD: tcpdump -i eth0 -nn -w /tmp/server.pcap host <CLIENT_IP> and port <PORT>
CONTEXT: Capture packets on server side for drop verification
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: tcpdump
CMD: tcpdump -i eth0 -nn -w /tmp/client.pcap host <SERVER_IP> and port <PORT>
CONTEXT: Capture packets on client side for drop verification (compare with server)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: tcpdump
CMD: tcpdump -r /tmp/server.pcap -nn | awk '{print $NF}' | grep seq | sort | uniq -d | head -10
CONTEXT: Find retransmissions in pcap file (duplicate sequence numbers)
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: tcpdump
CMD: tcpdump -i eth0 -nn -w /tmp/cap.pcap host <SERVER_IP> and port <PORT>
CONTEXT: Capture traffic to specific server for retransmission analysis
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: tcpdump
CMD: tcpdump -r /tmp/cap.pcap -nn | grep -E "\[P\.\]|seq" | head -50
CONTEXT: Analyze pcap for duplicate sequence numbers (retransmissions)
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: tcpdump
CMD: tcpdump -i any -nn port 53
CONTEXT: Capture DNS queries to observe ndots search domain amplification
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: tcpdump
CMD: tcpdump -i any -nn port 53 &
CONTEXT: Background DNS capture to count queries per request (ndots debugging)
SOURCE: 07-debugging-playbooks/03-dns-failures.md
---

TOOL: tcpdump
CMD: tcpdump -i eth0 -nn udp port 4789
CONTEXT: Capture Flannel VXLAN traffic to verify cross-node overlay
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

TOOL: tcpdump
CMD: tcpdump -i eth0 -nn udp port 8472
CONTEXT: Capture Flannel VXLAN traffic (alternate port some versions use)
SOURCE: 07-debugging-playbooks/06-kubernetes-networking-issues.md
---

## testssl.sh

TOOL: testssl.sh
CMD: testssl.sh --protocols --cipher --severity HIGH --json payment-api.mycompany.com:443 | jq '.scanResult[].findings[] | select(.severity == "HIGH" or .severity == "CRITICAL")'
CONTEXT: Check TLS compliance for high/critical severity findings on payment API
SOURCE: 06-cloud-security/08-compliance-frameworks.md
---

## top

TOOL: top
CMD: top -b -n 1 | head -20
CONTEXT: Check CPU utilization including steal time on server during latency spike
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## traceroute

TOOL: traceroute
CMD: traceroute -T -p <PORT> <SERVER_IP> -n
CONTEXT: TCP traceroute to find where packets stop (60-second triage)
SOURCE: 07-debugging-playbooks/00-debugging-methodology.md
---

TOOL: traceroute
CMD: traceroute -T -p 8080 <SERVER_IP> -n
CONTEXT: TCP traceroute to specific port (bypasses ICMP filtering)
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: traceroute
CMD: traceroute -T -p 8080 <SERVER_IP> -n --mtu
CONTEXT: TCP traceroute with MTU discovery (detects MTU black holes)
SOURCE: 07-debugging-playbooks/01-service-not-reachable.md
---

TOOL: traceroute
CMD: traceroute -T -p 443 <SERVER_IP> -n
CONTEXT: Network path analysis for latency debugging with per-hop RTT
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## trivy

TOOL: trivy
CMD: trivy image --scanners secret myapp:v1.2.3
CONTEXT: Scan container image layers for embedded secrets
SOURCE: 06-cloud-security/06-secrets-management.md
---

## vault

TOOL: vault
CMD: vault auth enable kubernetes
CONTEXT: Enable Kubernetes auth method in Vault
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: vault
CMD: vault write auth/kubernetes/config kubernetes_host="https://$K8S_HOST" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
CONTEXT: Configure Vault Kubernetes auth with cluster CA and reviewer JWT
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: vault
CMD: vault write auth/kubernetes/role/payments-api bound_service_account_names=payments-sa bound_service_account_namespaces=payments policies=payments-db-policy ttl=1h
CONTEXT: Create Vault role mapping K8s service account to Vault policy with 1h TTL
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: vault
CMD: vault policy write payments-db-policy - <<EOF
path "database/creds/payments-role" {
  capabilities = ["read"]
}
EOF
CONTEXT: Create Vault policy granting read access to dynamic database credentials
SOURCE: 06-cloud-security/06-secrets-management.md
---

TOOL: vault
CMD: vault write pki/issue/payments-role common_name="payments-api.payments.svc.cluster.local" ttl=5m
CONTEXT: Issue a short-lived (5-minute) TLS certificate from Vault PKI engine
SOURCE: 06-cloud-security/06-secrets-management.md
---

## vmstat

TOOL: vmstat
CMD: vmstat 1 5
CONTEXT: Check swap usage and I/O wait as latency indicators
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

## watch

TOOL: watch
CMD: watch -n 1 'nstat -az | grep -E "TcpRetransSegs|TcpExtTCPFastRetrans|TcpExtTCPSlowStartRetrans"'
CONTEXT: Monitor TCP retransmit counters in real time during latency investigation
SOURCE: 07-debugging-playbooks/02-high-latency.md
---

TOOL: watch
CMD: watch -n 1 'nstat -az | grep -i drop'
CONTEXT: Watch kernel drop counters increment in real time
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

TOOL: watch
CMD: watch -n 1 'iptables -L -n -v | grep -v "0     0"'
CONTEXT: Monitor iptables rules for active packet drops in real time
SOURCE: 07-debugging-playbooks/05-packet-drops.md
---

## zcat

TOOL: zcat
CMD: zcat /tmp/alb-logs/*.log.gz 2>/dev/null | tail -10000 | awk '{print $13}' | sort | uniq -c | sort -rn | head -20
CONTEXT: Find top client IPs in ALB access logs to identify DDoS sources
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: zcat
CMD: zcat /tmp/alb-logs/**/*.log.gz 2>/dev/null | awk '{print $13, $13, $16}' | sort | uniq -c | sort -rn | head -30
CONTEXT: Analyze ALB log URL and user-agent patterns during DDoS investigation
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---

TOOL: zcat
CMD: zcat /tmp/alb-logs/**/*.log.gz 2>/dev/null | awk '{print $NF}' | sort | uniq -c | sort -rn | head -20
CONTEXT: Find suspicious user agents in ALB logs (botnet detection)
SOURCE: 07-debugging-playbooks/08-ddos-incident-response.md
---
