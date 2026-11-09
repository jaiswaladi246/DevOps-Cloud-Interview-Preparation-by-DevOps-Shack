# AWS for DevOps — Cheatsheet

<!-- DEVOPS-SHACK:HEADER -->
<p align="center">
  <a href="https://www.youtube.com/@devopsshack"><img alt="YouTube DevOps Shack" src="https://img.shields.io/badge/YouTube-DevOps%20Shack-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://www.youtube.com/@shackverse"><img alt="YouTube DevOps Shack Hindi" src="https://img.shields.io/badge/Hindi-@shackverse-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://devopsshack.com"><img alt="devopsshack.com" src="https://img.shields.io/badge/Courses-devopsshack.com-C6FF4D?style=flat-square&labelColor=0B0F1A"/></a>
  <a href="https://www.instagram.com/devopsshack"><img alt="Instagram" src="https://img.shields.io/badge/Instagram-@devopsshack-E4405F?style=flat-square&logo=instagram&logoColor=white"/></a>
  <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack"><img alt="Original repo" src="https://img.shields.io/badge/Original-DevOps%20Shack%20repo-4DE8FF?style=flat-square&logo=github&labelColor=0B0F1A"/></a>
</p>
<p align="center"><sub>Part of <b><a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">DevOps &amp; Cloud Interview Preparation</a></b> — created by <b><a href="https://www.youtube.com/@devopsshack">DevOps Shack</a></b>. Free to learn from; please credit DevOps Shack when sharing.</sub></p>
<!-- /DEVOPS-SHACK:HEADER -->

> Dense CLI and config reference for day-to-day AWS work and interviews. AWS CLI v2 syntax.

[⬅ Back to AWS for DevOps](./README.md) · [🏠 Home](../README.md)

## Table of contents

- [CLI setup, identity and output](#cli-setup-identity-and-output)
- [EC2 and AMIs](#ec2-and-amis)
- [VPC networking](#vpc-networking)
- [IAM and STS](#iam-and-sts)
- [ELB and Auto Scaling](#elb-and-auto-scaling)
- [RDS](#rds)
- [S3](#s3)
- [CloudFront and Route 53](#cloudfront-and-route-53)
- [CloudWatch, SNS and CloudTrail](#cloudwatch-sns-and-cloudtrail)
- [Systems Manager and Secrets](#systems-manager-and-secrets)
- [Cost and tagging](#cost-and-tagging)
- [Policy snippets](#policy-snippets)
- [Numbers worth remembering](#numbers-worth-remembering)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## CLI setup, identity and output

| Task | Command |
|------|---------|
| Configure SSO profile | `aws configure sso` |
| Log in to SSO | `aws sso login --profile dev` |
| Who am I | `aws sts get-caller-identity` |
| Use a profile for one command | `aws s3 ls --profile dev` |
| Set Region for the shell | `export AWS_DEFAULT_REGION=ap-south-1` |
| List enabled Regions | `aws ec2 describe-regions --query 'Regions[].RegionName' --output text` |
| List AZs | `aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName'` |
| Output formats | `--output json\|text\|table\|yaml` |
| Filter client-side (JMESPath) | `--query 'Reservations[].Instances[].[InstanceId,State.Name]'` |
| Filter server-side | `--filters Name=tag:Project,Values=ha-web` |
| Paginate manually | `--max-items 50 --starting-token <token>` |
| Wait for a state | `aws ec2 wait instance-running --instance-ids i-...` |
| Dry run permission check | `aws ec2 run-instances ... --dry-run` |
| Generate a skeleton input | `aws ec2 run-instances --generate-cli-skeleton > in.json` |
| Debug a call | `aws s3 ls --debug 2>&1 \| head` |

Credential order: command-line flags, environment variables, assume-role/SSO profile, container credentials, instance profile (IMDS). Check `aws configure list` to see which source won.

## EC2 and AMIs

| Task | Command |
|------|---------|
| Latest AL2023 AMI | `aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text` |
| Launch an instance | `aws ec2 run-instances --image-id ami-... --instance-type t3.micro --subnet-id subnet-... --security-group-ids sg-... --iam-instance-profile Name=prof --metadata-options HttpTokens=required` |
| List running instances | `aws ec2 describe-instances --filters Name=instance-state-name,Values=running --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress,Tags[?Key==`Name`]\|[0].Value]' --output table` |
| Stop, start, terminate | `aws ec2 stop-instances --instance-ids i-...` / `start-instances` / `terminate-instances` |
| Console output (boot errors) | `aws ec2 get-console-output --instance-id i-... --output text` |
| Status checks | `aws ec2 describe-instance-status --instance-ids i-... --include-all-instances` |
| Create an AMI | `aws ec2 create-image --instance-id i-... --name app-v1 --no-reboot` |
| Copy AMI across Regions | `aws ec2 copy-image --source-image-id ami-... --source-region ap-south-1 --region us-east-1 --name app-v1` |
| Change instance type | stop, then `aws ec2 modify-instance-attribute --instance-id i-... --instance-type Value=t3.large` |
| Read user data | `aws ec2 describe-instance-attribute --instance-id i-... --attribute userData` |
| Snapshot a volume | `aws ec2 create-snapshot --volume-id vol-... --description "pre-change"` |
| Unattached volumes | `aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].[VolumeId,Size]'` |

IMDSv2 from inside an instance:

```bash
TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

## VPC networking

| Task | Command |
|------|---------|
| Create VPC | `aws ec2 create-vpc --cidr-block 10.0.0.0/16` |
| Create subnet | `aws ec2 create-subnet --vpc-id vpc-... --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a` |
| Internet gateway | `aws ec2 create-internet-gateway` then `attach-internet-gateway --internet-gateway-id igw-... --vpc-id vpc-...` |
| Default route to IGW | `aws ec2 create-route --route-table-id rtb-... --destination-cidr-block 0.0.0.0/0 --gateway-id igw-...` |
| NAT gateway | `aws ec2 create-nat-gateway --subnet-id subnet-pub --allocation-id eipalloc-...` |
| Security group rule from SG | `aws ec2 authorize-security-group-ingress --group-id sg-app --ip-permissions "IpProtocol=tcp,FromPort=80,ToPort=80,UserIdGroupPairs=[{GroupId=sg-alb}]"` |
| Remove a rule | `aws ec2 revoke-security-group-ingress --group-id sg-... --protocol tcp --port 22 --cidr 0.0.0.0/0` |
| S3 gateway endpoint | `aws ec2 create-vpc-endpoint --vpc-id vpc-... --service-name com.amazonaws.ap-south-1.s3 --route-table-ids rtb-...` |
| Interface endpoint | `aws ec2 create-vpc-endpoint --vpc-id vpc-... --vpc-endpoint-type Interface --service-name com.amazonaws.ap-south-1.ssm --subnet-ids subnet-... --security-group-ids sg-... --private-dns-enabled` |
| VPC peering | `aws ec2 create-vpc-peering-connection --vpc-id vpc-a --peer-vpc-id vpc-b` then accept and add routes both ways |
| Flow logs to CloudWatch | `aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-... --traffic-type ALL --log-group-name vpc-flow --deliver-logs-permission-arn arn:aws:iam::<acct>:role/flowlogs` |
| Reachability analysis | `aws ec2 create-network-insights-path --source i-... --destination i-... --protocol tcp --destination-port 443` |

Security group versus NACL:

| | Security group | Network ACL |
|---|---|---|
| Level | Network interface | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and deny, numbered |
| Evaluation | All rules | First match by rule number |
| Default | Deny in, allow out | Default NACL allows all |

Private subnets that need to reach AWS APIs: NAT gateway, or VPC endpoints (gateway for S3 and DynamoDB, interface for most others).

## IAM and STS

| Task | Command |
|------|---------|
| Create role | `aws iam create-role --role-name r --assume-role-policy-document file://trust.json` |
| Attach managed policy | `aws iam attach-role-policy --role-name r --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess` |
| Inline policy | `aws iam put-role-policy --role-name r --policy-name p --policy-document file://p.json` |
| Instance profile | `aws iam create-instance-profile --instance-profile-name p` and `add-role-to-instance-profile` |
| Assume a role | `aws sts assume-role --role-arn arn:aws:iam::<acct>:role/r --role-session-name s` |
| Test a policy | `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::b/k` |
| Last used info | `aws iam get-role --role-name r --query Role.RoleLastUsed` |
| Credential report | `aws iam generate-credential-report` then `aws iam get-credential-report` |
| Old access keys | `aws iam list-access-keys --user-name u` and `get-access-key-last-used --access-key-id AKIA...` |
| Decode an auth error | `aws sts decode-authorization-message --encoded-message <msg>` |
| Policy versions | `aws iam list-policy-versions --policy-arn <arn>` |

Evaluation order: explicit deny, then SCP, resource policy (same account), permission boundary, session policy, identity policy; default is deny. Cross-account needs an allow on both sides.

Temporary credentials from `assume-role` into the shell:

```bash
creds=$(aws sts assume-role --role-arn "$ROLE_ARN" --role-session-name lab --query Credentials --output json)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKeyId <<<"$creds")
export AWS_SECRET_ACCESS_KEY=$(jq -r .SecretAccessKey <<<"$creds")
export AWS_SESSION_TOKEN=$(jq -r .SessionToken <<<"$creds")
```

## ELB and Auto Scaling

| Task | Command |
|------|---------|
| Create ALB | `aws elbv2 create-load-balancer --name alb --type application --subnets s1 s2 --security-groups sg-...` |
| Target group | `aws elbv2 create-target-group --name tg --protocol HTTP --port 80 --vpc-id vpc-... --health-check-path /health` |
| Listener | `aws elbv2 create-listener --load-balancer-arn <arn> --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=<tg>` |
| HTTPS listener | `--protocol HTTPS --port 443 --certificates CertificateArn=<acm-arn> --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06` |
| Path rule | `aws elbv2 create-rule --listener-arn <arn> --priority 10 --conditions Field=path-pattern,Values='/api/*' --actions Type=forward,TargetGroupArn=<tg2>` |
| Target health | `aws elbv2 describe-target-health --target-group-arn <tg>` |
| Deregistration delay | `aws elbv2 modify-target-group-attributes --target-group-arn <tg> --attributes Key=deregistration_delay.timeout_seconds,Value=30` |
| Create launch template | `aws ec2 create-launch-template --launch-template-name lt --launch-template-data file://lt.json` |
| New template version | `aws ec2 create-launch-template-version --launch-template-name lt --source-version 1 --launch-template-data file://v2.json` |
| Create ASG | `aws autoscaling create-auto-scaling-group --auto-scaling-group-name asg --launch-template LaunchTemplateName=lt,Version='$Latest' --min-size 2 --max-size 4 --vpc-zone-identifier "s1,s2" --target-group-arns <tg> --health-check-type ELB --health-check-grace-period 180` |
| Target tracking | `aws autoscaling put-scaling-policy --auto-scaling-group-name asg --policy-name cpu --policy-type TargetTrackingScaling --target-tracking-configuration file://tt.json` |
| Scheduled action | `aws autoscaling put-scheduled-update-group-action --auto-scaling-group-name asg --scheduled-action-name day --recurrence "0 9 * * 1-5" --min-size 4` |
| Rolling deploy | `aws autoscaling start-instance-refresh --auto-scaling-group-name asg --preferences '{"MinHealthyPercentage":50,"InstanceWarmup":120}'` |
| Cancel refresh | `aws autoscaling cancel-instance-refresh --auto-scaling-group-name asg` |
| Protect an instance | `aws autoscaling set-instance-protection --auto-scaling-group-name asg --instance-ids i-... --protected-from-scale-in` |
| Take out of service for debugging | `aws autoscaling enter-standby --auto-scaling-group-name asg --instance-ids i-... --should-decrement-desired-capacity` |
| Scaling history | `aws autoscaling describe-scaling-activities --auto-scaling-group-name asg` |

ALB versus NLB: ALB is layer 7 with host, path, header and query routing, WebSockets, gRPC, and no static IP. NLB is layer 4 with TCP, UDP and TLS, static or Elastic IPs per AZ, preserves the client source IP, and handles millions of requests per second. Gateway Load Balancer fronts virtual appliances.

ALB error codes: 502 bad response from target or closed connection, 503 no healthy targets, 504 target did not respond within the idle timeout, 460 client closed the connection, 463 too many `X-Forwarded-For` addresses.

## RDS

| Task | Command |
|------|---------|
| Describe instance | `aws rds describe-db-instances --db-instance-identifier db` |
| Create Multi-AZ | `aws rds create-db-instance --db-instance-identifier db --engine postgres --db-instance-class db.t4g.micro --allocated-storage 20 --master-username admin --manage-master-user-password --multi-az --no-publicly-accessible --db-subnet-group-name sg --vpc-security-group-ids sg-...` |
| Force failover | `aws rds reboot-db-instance --db-instance-identifier db --force-failover` |
| Read replica | `aws rds create-db-instance-read-replica --db-instance-identifier db-r1 --source-db-instance-identifier db` |
| Promote replica | `aws rds promote-read-replica --db-instance-identifier db-r1` |
| Manual snapshot | `aws rds create-db-snapshot --db-instance-identifier db --db-snapshot-identifier db-pre-change` |
| Restore to a new instance | `aws rds restore-db-instance-from-db-snapshot --db-instance-identifier db-restored --db-snapshot-identifier db-pre-change` |
| Point in time restore | `aws rds restore-db-instance-to-point-in-time --source-db-instance-identifier db --target-db-instance-identifier db-pit --restore-time 2026-01-01T10:00:00Z` |
| Modify now | `aws rds modify-db-instance --db-instance-identifier db --db-instance-class db.m6g.large --apply-immediately` |
| Recent events | `aws rds describe-events --source-type db-instance --source-identifier db --duration 60` |
| Logs | `aws rds describe-db-log-files --db-instance-identifier db` |
| Enable deletion protection | `aws rds modify-db-instance --db-instance-identifier db --deletion-protection` |

Key metrics: `CPUUtilization`, `FreeableMemory`, `FreeStorageSpace`, `DatabaseConnections`, `ReadLatency`, `WriteLatency`, `ReplicaLag`, `BurstBalance` (gp2), `DiskQueueDepth`.

Multi-AZ instance deployment: synchronous standby, same endpoint, failover about 60 to 120 seconds, standby not readable. Multi-AZ DB cluster (two readable standbys) fails over faster. Read replicas: asynchronous, readable, can be cross-Region, promotable.

## S3

| Task | Command |
|------|---------|
| Create bucket (non us-east-1) | `aws s3api create-bucket --bucket b --create-bucket-configuration LocationConstraint=ap-south-1` |
| Block public access | `aws s3api put-public-access-block --bucket b --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true` |
| Versioning | `aws s3api put-bucket-versioning --bucket b --versioning-configuration Status=Enabled` |
| Sync a folder | `aws s3 sync ./dist s3://b/ --delete` |
| Copy with storage class | `aws s3 cp f s3://b/f --storage-class STANDARD_IA` |
| Presigned URL | `aws s3 presign s3://b/f --expires-in 3600` |
| List object versions | `aws s3api list-object-versions --bucket b --prefix key` |
| Lifecycle | `aws s3api put-bucket-lifecycle-configuration --bucket b --lifecycle-configuration file://lc.json` |
| Bucket policy | `aws s3api put-bucket-policy --bucket b --policy file://p.json` |
| Replication | `aws s3api put-bucket-replication --bucket b --replication-configuration file://repl.json` |
| Size of a bucket | `aws s3 ls s3://b --recursive --summarize --human-readable \| tail -2` |
| Empty and delete | `aws s3 rm s3://b --recursive` then `aws s3api delete-bucket --bucket b` |

Storage classes: Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Glacier Instant Retrieval, Glacier Flexible Retrieval, Glacier Deep Archive. Default encryption is SSE-S3; choose SSE-KMS for key-level audit and control. S3 is strongly consistent for reads after writes and lists.

## CloudFront and Route 53

| Task | Command |
|------|---------|
| Create OAC | `aws cloudfront create-origin-access-control --origin-access-control-config Name=oac,SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3` |
| Create distribution | `aws cloudfront create-distribution --distribution-config file://cf.json` |
| Invalidate cache | `aws cloudfront create-invalidation --distribution-id E... --paths "/index.html" "/static/*"` |
| Wait until deployed | `aws cloudfront wait distribution-deployed --id E...` |
| Get config and ETag | `aws cloudfront get-distribution-config --id E...` |
| List hosted zones | `aws route53 list-hosted-zones` |
| Change records | `aws route53 change-resource-record-sets --hosted-zone-id Z... --change-batch file://change.json` |
| List records | `aws route53 list-resource-record-sets --hosted-zone-id Z...` |
| Health check | `aws route53 create-health-check --caller-reference $(date +%s) --health-check-config Type=HTTP,FullyQualifiedDomainName=x.com,ResourcePath=/health` |
| Test a record | `aws route53 test-dns-answer --hosted-zone-id Z... --record-name x.com --record-type A` |

Managed cache policy IDs: `658327ea-f89d-4fab-a63d-7e88639e58f6` is CachingOptimized, `4135ea2d-6df8-44a3-9df3-4b5a84be39ad` is CachingDisabled. CloudFront certificates must live in `us-east-1`.

Alias record skeleton:

```json
{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"app.example.com","Type":"A",
 "AliasTarget":{"HostedZoneId":"<ALB CanonicalHostedZoneId>","DNSName":"dualstack.<alb-dns>","EvaluateTargetHealth":true}}}]}
```

Routing policies: simple, weighted, latency, failover, geolocation, geoproximity, multivalue answer, IP-based. Aliases to AWS resources are free and allowed at the zone apex; a CNAME is not allowed at the apex.

## CloudWatch, SNS and CloudTrail

| Task | Command |
|------|---------|
| Put metric alarm | `aws cloudwatch put-metric-alarm --alarm-name a --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=i-... --statistic Average --period 60 --evaluation-periods 3 --threshold 80 --comparison-operator GreaterThanThreshold --alarm-actions <sns-arn>` |
| List alarms in ALARM | `aws cloudwatch describe-alarms --state-value ALARM` |
| Test an alarm | `aws cloudwatch set-alarm-state --alarm-name a --state-value ALARM --state-reason test` |
| Read a metric | `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=i-... --start-time 2026-01-01T00:00:00Z --end-time 2026-01-01T01:00:00Z --period 300 --statistics Average` |
| Tail logs | `aws logs tail /aws/app --follow --since 10m` |
| Log retention | `aws logs put-retention-policy --log-group-name /aws/app --retention-in-days 14` |
| Logs Insights | `aws logs start-query --log-group-name /aws/app --start-time <epoch> --end-time <epoch> --query-string 'fields @timestamp, @message \| filter @message like /ERROR/ \| limit 20'` |
| Metric filter | `aws logs put-metric-filter --log-group-name /aws/app --filter-name errors --filter-pattern ERROR --metric-transformations metricName=Errors,metricNamespace=App,metricValue=1` |
| Create SNS topic | `aws sns create-topic --name alerts` |
| Subscribe | `aws sns subscribe --topic-arn <arn> --protocol email --notification-endpoint me@x.com` |
| Publish a test | `aws sns publish --topic-arn <arn> --message "test"` |
| Find an API call | `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=TerminateInstances` |
| Create trail | `aws cloudtrail create-trail --name org-trail --s3-bucket-name <bucket> --is-multi-region-trail` then `start-logging` |

Useful metric names: `AWS/ApplicationELB` has `HTTPCode_Target_5XX_Count`, `TargetResponseTime`, `UnHealthyHostCount`, `RequestCountPerTarget`. `AWS/EC2` has `CPUUtilization`, `StatusCheckFailed`. Memory and disk are not EC2 metrics; use the CloudWatch agent.

## Systems Manager and Secrets

| Task | Command |
|------|---------|
| Shell without SSH | `aws ssm start-session --target i-...` |
| Port forward to a private host | `aws ssm start-session --target i-... --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters '{"host":["db.internal"],"portNumber":["5432"],"localPortNumber":["5433"]}'` |
| Run a command on instances | `aws ssm send-command --document-name AWS-RunShellScript --targets Key=tag:Project,Values=ha-web --parameters 'commands=["uptime"]'` |
| Command output | `aws ssm get-command-invocation --command-id <id> --instance-id i-...` |
| Managed instances | `aws ssm describe-instance-information --query 'InstanceInformationList[].[InstanceId,PingStatus]'` |
| Put parameter | `aws ssm put-parameter --name /app/prod/db-host --type String --value x --overwrite` |
| Secure parameter | `aws ssm put-parameter --name /app/prod/token --type SecureString --value x` |
| Get parameter | `aws ssm get-parameter --name /app/prod/token --with-decryption --query Parameter.Value --output text` |
| By path | `aws ssm get-parameters-by-path --path /app/prod --recursive --with-decryption` |
| Secret value | `aws secretsmanager get-secret-value --secret-id <arn> --query SecretString --output text` |
| Create secret | `aws secretsmanager create-secret --name app/prod/db --secret-string '{"user":"u","password":"p"}'` |
| Rotate | `aws secretsmanager rotate-secret --secret-id app/prod/db` |

## Cost and tagging

| Task | Command |
|------|---------|
| Cost by service | `aws ce get-cost-and-usage --time-period Start=2026-01-01,End=2026-02-01 --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE` |
| Cost by tag | `aws ce get-cost-and-usage --time-period Start=...,End=... --granularity DAILY --metrics UnblendedCost --group-by Type=TAG,Key=Project` |
| Forecast | `aws ce get-cost-forecast --time-period Start=...,End=... --metric UNBLENDED_COST --granularity MONTHLY` |
| Activate cost tag | `aws ce update-cost-allocation-tags-status --cost-allocation-tags-status TagKey=Project,Status=Active --region us-east-1` |
| Resources by tag | `aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values=ha-web` |
| Create budget | `aws budgets create-budget --account-id <id> --budget file://b.json --notifications-with-subscribers file://n.json` |
| Rightsizing | `aws compute-optimizer get-ec2-instance-recommendations` |

Where the money usually leaks: NAT gateway hourly and per-GB, cross-AZ and internet data transfer, idle ALBs and NAT gateways, unattached EBS volumes and old snapshots, oversized RDS, CloudWatch Logs without retention, public IPv4 addresses (charged hourly).

## Policy snippets

Least-privilege S3 read of one prefix:

```json
{"Version":"2012-10-17","Statement":[
 {"Effect":"Allow","Action":"s3:GetObject","Resource":"arn:aws:s3:::my-bucket/reports/*"},
 {"Effect":"Allow","Action":"s3:ListBucket","Resource":"arn:aws:s3:::my-bucket",
  "Condition":{"StringLike":{"s3:prefix":["reports/*"]}}}]}
```

Trust policy for EC2:

```json
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
```

Deny everything over plain HTTP on a bucket:

```json
{"Effect":"Deny","Principal":"*","Action":"s3:*","Resource":["arn:aws:s3:::my-bucket","arn:aws:s3:::my-bucket/*"],
 "Condition":{"Bool":{"aws:SecureTransport":"false"}}}
```

Cross-account trust with external ID:

```json
{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::111111111111:root"},"Action":"sts:AssumeRole",
 "Condition":{"StringEquals":{"sts:ExternalId":"lab-1234"}}}
```

Require MFA for a sensitive action:

```json
{"Effect":"Deny","Action":"ec2:TerminateInstances","Resource":"*",
 "Condition":{"BoolIfExists":{"aws:MultiFactorAuthPresent":"false"}}}
```

## Numbers worth remembering

| Item | Value |
|------|-------|
| Default VPCs per Region quota | 5 (adjustable) |
| Security groups per network interface | 5 default (adjustable up to 16) |
| Rules per security group | 60 inbound, 60 outbound default |
| Reserved IPs per subnet | 5 (first four and the last) |
| Smallest and largest VPC CIDR | /28 to /16 |
| ALB idle timeout default | 60 seconds |
| ALB target health defaults | interval 30 s, healthy 5, unhealthy 2 |
| ASG default cooldown | 300 seconds |
| RDS Multi-AZ failover | typically 60 to 120 seconds |
| S3 single PUT limit, max object | 5 GB, 5 TB |
| S3 durability, Standard availability | 11 nines, 99.99 percent |
| CloudFront invalidation | first 1,000 paths a month free |
| Lambda max duration | 15 minutes |
| CloudTrail event history | 90 days of management events |
| STS assume-role session | 1 hour default, up to the role maximum (up to 12 hours) |
| Route 53 SLA | 100 percent |

Quotas change. Check Service Quotas when it matters: `aws service-quotas list-service-quotas --service-code vpc`.

## Top 15 interview one-liners

1. A Region is a geographic area with multiple AZs; AZs give high availability, Regions give disaster recovery.
2. A subnet is public only because its route table sends `0.0.0.0/0` to an internet gateway.
3. Security groups are stateful allow-lists on interfaces; NACLs are stateless numbered allow and deny rules on subnets.
4. Use roles, not access keys, for workloads: instance profiles on EC2, task roles on ECS, IRSA or Pod Identity on EKS.
5. IAM evaluation: explicit deny always wins, then something must explicitly allow; the default is deny.
6. An ASG with `--health-check-type ELB` replaces instances that fail the load balancer check, not just dead hosts.
7. ALB is layer 7 with path and host routing; NLB is layer 4 with static IPs and the client source IP preserved.
8. RDS Multi-AZ is for availability (sync standby, same endpoint, 60 to 120 second failover); read replicas are for scale and cross-Region DR.
9. Keep S3 private with Block Public Access and let CloudFront read it through Origin Access Control scoped to one distribution ARN.
10. Route 53 alias records to ALB or CloudFront are free, work at the apex, and follow the target's addresses.
11. Alarm on symptoms users feel (5xx, latency, unhealthy hosts), not only on CPU.
12. CloudTrail tells you who did what; CloudWatch tells you what the system is doing; Config tells you how it changed.
13. Private subnets need a NAT gateway or VPC endpoints to reach AWS APIs; gateway endpoints for S3 and DynamoDB are free.
14. The big surprise bills are NAT data processing, cross-AZ and internet transfer, idle load balancers and unattached volumes. Tag and budget from day one.
15. Never put secrets in user data or AMIs; read them at runtime from Secrets Manager or Parameter Store using the instance role.

[⬅ Back to AWS for DevOps](./README.md) · [🏠 Home](../README.md)

<!-- DEVOPS-SHACK:FOOTER -->
---

<div align="center">

### 🚀 Made with ❤️ by [DevOps Shack](https://www.youtube.com/@devopsshack)

**Learn DevOps, DevSecOps, Cloud and Kubernetes the practical way.**

[▶️ YouTube — DevOps Shack](https://www.youtube.com/@devopsshack) · [▶️ Hindi — @shackverse](https://www.youtube.com/@shackverse) · [🎓 devopsshack.com](https://devopsshack.com) · [📸 Instagram @devopsshack](https://www.instagram.com/devopsshack)

[🏠 Back to Home](../README.md) · [⭐ Star the original repo](https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack)

<sub>© DevOps Shack · Licensed CC BY-NC-SA 4.0 · If you found this copy anywhere other than <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">github.com/jaiswaladi246</a>, the original and latest version is maintained by DevOps Shack.</sub>

</div>
<!-- /DEVOPS-SHACK:FOOTER -->
