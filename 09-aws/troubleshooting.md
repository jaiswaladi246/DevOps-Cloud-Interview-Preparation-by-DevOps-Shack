# AWS — Troubleshooting

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

> Real AWS error messages, symptoms and how to diagnose them with the aws CLI · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to AWS](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [IAM and Access Errors](#iam-and-access-errors) — Q1–Q25
- [VPC and Connectivity](#vpc-and-connectivity) — Q26–Q50
- [EC2, Auto Scaling and Load Balancing](#ec2-auto-scaling-and-load-balancing) — Q51–Q75
- [S3, EBS and EFS](#s3-ebs-and-efs) — Q76–Q100
- [RDS, Aurora and DynamoDB](#rds-aurora-and-dynamodb) — Q101–Q125
- [ECS, EKS, ECR and Lambda](#ecs-eks-ecr-and-lambda) — Q126–Q150
- [Route 53, CloudFront, API Gateway and Messaging](#route-53-cloudfront-api-gateway-and-messaging) — Q151–Q175
- [CloudFormation, CloudWatch, KMS, Secrets and Cost](#cloudformation-cloudwatch-kms-secrets-and-cost) — Q176–Q200

---

## IAM and Access Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied` for a user who has S3 read access. What is wrong? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The policy grants object actions but not `s3:ListAllMyBuckets`; or the user is on a different profile than expected.

**Diagnose →**

```bash
aws sts get-caller-identity
aws iam list-attached-user-policies --user-name alice
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::111111111111:user/alice --action-names s3:ListAllMyBuckets
```

**Fix →**

1. Confirm you are using the intended profile with `AWS_PROFILE`
2. Add `s3:ListAllMyBuckets` on `*` if bucket listing is genuinely required
3. Otherwise instruct the user to `aws s3 ls s3://bucket` directly

**Prevent →** Document the minimal policy template for S3 users; bucket-scoped listing needs `s3:ListBucket` on the bucket ARN.

</details>

### Q2. `ExpiredToken: The security token included in the request is expired` in a long CI job. How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Assumed-role sessions last 1 hour by default and the job exceeded it, or cached credentials are stale.

**Diagnose →**

```bash
aws sts get-caller-identity
env | grep AWS_
aws iam get-role --role-name ci-deploy --query Role.MaxSessionDuration
```

**Fix →**

1. Raise role `MaxSessionDuration` and request `--duration-seconds` accordingly
2. Re-authenticate mid-job or split the job
3. Unset stale `AWS_SESSION_TOKEN` variables left over from a prior step

**Prevent →** Use OIDC with fresh credentials per job and keep jobs short.

</details>

### Q3. `User: arn:aws:sts::111111111111:assumed-role/dev/bob is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::222222222222:role/deploy`. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Caller lacks an `sts:AssumeRole` allow, or the target trust policy excludes the caller, or an SCP denies it.

**Diagnose →**

```bash
aws iam get-role --role-name deploy --query Role.AssumeRolePolicyDocument
aws iam list-role-policies --role-name dev
aws organizations list-policies-for-target --target-id 111111111111 --filter SERVICE_CONTROL_POLICY
```

**Fix →**

1. Add allow on the target ARN to the caller's policy
2. Fix the trust policy principal
3. Remove blocking SCP conditions

**Prevent →** Keep a trust-policy template in IaC and test with a CI assume-role smoke test.

</details>

### Q4. `An error occurred (AccessDenied) ... because no identity-based policy allows the s3:GetObject action` yet a bucket policy allows the user. Cross-account? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** For cross-account, both the caller's account identity policy and the bucket policy must allow.

**Diagnose →**

```bash
aws s3api get-bucket-policy --bucket shared
aws iam list-attached-role-policies --role-name reader
aws iam simulate-principal-policy --policy-source-arn <arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::shared/*
```

**Fix →**

1. Add `s3:GetObject` to the caller role's identity policy
2. Keep the bucket policy as is

**Prevent →** Treat cross-account access as two changes in your runbook.

</details>

### Q5. IAM policy validation: `MalformedPolicyDocument: Policy document should not specify a principal`. What happened? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A resource-based policy element was placed in an identity policy.

**Diagnose →**

```bash
aws accessanalyzer validate-policy --policy-document file://p.json --policy-type IDENTITY_POLICY
```

**Fix →**

1. Remove `Principal` from the identity policy
2. Use a trust policy for roles instead
3. Re-run validation

**Prevent →** Validate policies in CI by policy type.

</details>

### Q6. `The security token included in the request is invalid` when using `aws` CLI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong region or partition, a mangled session token, or credentials of a deleted key.

**Diagnose →**

```bash
aws configure list
env | grep -i aws
aws sts get-caller-identity --debug 2>&1 | head -50
```

**Fix →**

1. Unset environment credentials and rerun `aws configure` or `aws sso login`
2. Check for trailing whitespace in the token
3. Re-create the access key if deleted

**Prevent →** Prefer SSO and avoid copy-pasting tokens.

</details>

### Q7. SSO login works in the browser but `aws sso login` hangs or gives `Error loading SSO Token: Token for ... does not exist`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or stale cache in `~/.aws/sso/cache`, wrong `sso_session`, or a headless terminal.

**Diagnose →**

```bash
cat ~/.aws/config
ls ~/.aws/sso/cache
aws sso login --profile dev --use-device-code --debug
```

**Fix →**

1. Remove stale cache files and log in again
2. Use `--use-device-code` on remote machines
3. Verify `sso_region` and `sso_start_url`

**Prevent →** Use `aws configure sso-session` templates for the team.

</details>

### Q8. An SCP is suspected of denying actions but console errors are vague. How do you confirm? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Explicit deny from SCP shows in the error text of recent API versions and in CloudTrail.

**Diagnose →**

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances --max-results 5
aws organizations describe-effective-policy --policy-type SERVICE_CONTROL_POLICY --target-id 111111111111
aws organizations list-policies-for-target --target-id ou-xxxx --filter SERVICE_CONTROL_POLICY
```

**Fix →**

1. Adjust the SCP, test in a sandbox OU first
2. Use an exception for the required principal via condition keys

**Prevent →** Keep SCPs in IaC with a test suite of allowed and denied calls.

</details>

### Q9. `iam:PassRole` error: `User is not authorized to perform: iam:PassRole on resource: arn:aws:iam::...:role/task-role`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The deployer can launch a service but lacks permission to hand the role to it.

**Diagnose →**

```bash
aws iam get-role --role-name task-role
aws iam simulate-principal-policy --policy-source-arn <deployer-arn> --action-names iam:PassRole --resource-arns <role-arn>
```

**Fix →**

1. Add `iam:PassRole` on that specific role ARN with condition `iam:PassedToService`
2. Do not use wildcard resources

**Prevent →** Scope PassRole narrowly; it is a privilege escalation path.

</details>

### Q10. A newly created IAM user cannot see anything in the console. Why? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New users have no permissions.

**Diagnose →**

```bash
aws iam list-attached-user-policies --user-name new
aws iam list-groups-for-user --user-name new
```

**Fix →**

1. Attach a group with required policies
2. Wait a few seconds for propagation

**Prevent →** Manage by groups or SSO, not by individual attachments.

</details>

### Q11. Policy changes do not seem to apply for several minutes. Is something broken? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IAM is eventually consistent, and existing sessions keep old permissions until refreshed.

**Diagnose →**

```bash
aws iam get-policy-version --policy-arn <arn> --version-id v2
aws sts get-caller-identity
aws iam get-role --role-name r
```

**Fix →**

1. Retry with backoff
2. Re-assume the role to get a fresh session
3. Avoid create-then-use in the same script without waiting

**Prevent →** Build retries into IaC and deployment pipelines.

</details>

### Q12. `AccessDenied` with message `explicit deny in a resource-based policy` for a KMS decrypt in a cross-account pipeline. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The key policy lacks the caller or has a deny condition on `kms:ViaService` or encryption context.

**Diagnose →**

```bash
aws kms get-key-policy --key-id <key> --policy-name default
aws kms list-grants --key-id <key>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=Decrypt
```

**Fix →**

1. Add the role ARN to the key policy or a grant
2. Add the identity policy allow for `kms:Decrypt`
3. Check encryption context

**Prevent →** Review key policies when granting cross-account S3 or EBS access.

</details>

### Q13. The error `You are not authorized to perform this operation. Encoded authorization failure message: ...`. How do you read it? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** EC2 encodes failure details that must be decoded.

**Diagnose →**

```bash
aws sts decode-authorization-message --encoded-message <msg> --query DecodedMessage --output text | jq .
```

**Fix →**

1. Read `explicitDeny`, `matchedStatements`, and context keys
2. Fix the policy statement that matched

**Prevent →** Grant `sts:DecodeAuthorizationMessage` to engineers.

</details>

### Q14. An IAM role created by CloudFormation cannot be assumed by a Lambda: `The role defined for the function cannot be assumed by Lambda`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Trust policy missing `lambda.amazonaws.com`, or the role was just created.

**Diagnose →**

```bash
aws iam get-role --role-name fn-role --query Role.AssumeRolePolicyDocument
```

**Fix →**

1. Add `lambda.amazonaws.com` to trust policy
2. Add `DependsOn` or wait; retry creation

**Prevent →** Use SAM or CDK helpers that wire trust automatically.

</details>

### Q15. Your IAM role limit error: `LimitExceeded: Cannot exceed quota for PoliciesPerRole: 10`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Managed policy attachments per role are capped at 10 by default (up to 20 by quota).

**Diagnose →**

```bash
aws iam list-attached-role-policies --role-name big
aws service-quotas get-service-quota --service-code iam --quota-code L-0DA4ABF3
```

**Fix →**

1. Consolidate policies into fewer customer managed ones
2. Request a quota increase
3. Split the role by function

**Prevent →** Design roles per workload, not mega-roles.

</details>

### Q16. `Policy size exceeds quota: Maximum policy size of 6144 characters` when attaching an inline policy. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Role inline policies total limit is 10,240 chars; managed policies cap at 6,144.

**Diagnose →**

```bash
wc -c policy.json
aws iam list-role-policies --role-name r
```

**Fix →**

1. Use wildcards by prefix with care
2. Split into several managed policies
3. Use tag-based ABAC to shrink statements

**Prevent →** Prefer ABAC and resource tags over listing ARNs.

</details>

### Q17. Access Analyzer reports an external access finding for an S3 bucket you thought was private. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bucket policy or ACL grants a principal outside the zone of trust, possibly via `Principal: *`.

**Diagnose →**

```bash
aws accessanalyzer list-findings --analyzer-arn <arn>
aws s3api get-bucket-policy-status --bucket b
aws s3api get-bucket-acl --bucket b
```

**Fix →**

1. Remove the offending statement
2. Re-enable Block Public Access
3. Archive the finding if intentional

**Prevent →** Alert on new findings through EventBridge.

</details>

### Q18. Cross-account role session shows `AccessDenied` only for `sts:TagSession` calls from an SSO flow. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Trust policy lacks `sts:TagSession` which is needed when sessions carry tags.

**Diagnose →**

```bash
aws iam get-role --role-name target
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole
```

**Fix →**

1. Add `sts:TagSession` to the trust policy action list
2. Retry

**Prevent →** Include `sts:TagSession` whenever you use session tags or ABAC.

</details>

### Q19. `The config profile (prod) could not be found`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Profile name not defined in `~/.aws/config`.

**Diagnose →**

```bash
aws configure list-profiles
cat ~/.aws/config
```

**Fix →**

1. Add the profile or fix the name
2. Check `AWS_PROFILE` env value

**Prevent →** Standardize profile names across the team.

</details>

### Q20. GitHub Actions OIDC fails: `Not authorized to perform sts:AssumeRoleWithWebIdentity`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Trust policy `sub` or `aud` conditions mismatch the workflow's token.

**Diagnose →**

```bash
aws iam get-role --role-name gh-deploy --query Role.AssumeRolePolicyDocument
# in workflow print the token claims with a debug step (never in production logs)
```

**Fix →**

1. Align `sub` with `repo:org/repo:ref:refs/heads/main` or environment form
2. Ensure `aud` equals `sts.amazonaws.com`
3. Add `id-token: write` permission

**Prevent →** Test trust policy with a staging branch pattern.

</details>

### Q21. Instance role credentials stopped working inside a container on an EC2 host: `Unable to locate credentials`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IMDSv2 hop limit of 1 blocks containers from reaching metadata.

**Diagnose →**

```bash
curl -s -X PUT http://169.254.169.254/latest/api/token -H 'X-aws-ec2-metadata-token-ttl-seconds: 60'
aws ec2 describe-instances --instance-ids i-0abc --query 'Reservations[].Instances[].MetadataOptions'
```

**Fix →**

1. Set hop limit to 2: `aws ec2 modify-instance-metadata-options --instance-id i-0abc --http-put-response-hop-limit 2 --http-tokens required`
2. Or use task or pod roles

**Prevent →** Bake hop limit into launch templates.

</details>

### Q22. Organization CloudTrail stopped delivering logs. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bucket policy changed, KMS key permissions broke, or the trail was disabled.

**Diagnose →**

```bash
aws cloudtrail get-trail-status --name org-trail
aws s3api get-bucket-policy --bucket log-archive
aws kms describe-key --key-id <key>
```

**Fix →**

1. Restore bucket policy allowing `cloudtrail.amazonaws.com`
2. Fix KMS key policy
3. `aws cloudtrail start-logging --name org-trail`

**Prevent →** Alarm on `LatestDeliveryError` and protect via SCP.

</details>

### Q23. Console says `You don't have permission to view this` for CloudWatch Logs but CLI works. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Console needs extra read permissions such as `logs:DescribeLogGroups`.

**Diagnose →**

```bash
aws iam simulate-principal-policy --policy-source-arn <arn> --action-names logs:DescribeLogGroups logs:GetLogEvents
```

**Fix →**

1. Add console-required describe actions
2. Use the AWS managed `CloudWatchLogsReadOnlyAccess`

**Prevent →** Test console flows for each new role.

</details>

### Q24. `InvalidClientTokenId` when calling an API in a new opt-in region. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The region is disabled for the account or credentials are for the wrong partition.

**Diagnose →**

```bash
aws account get-region-opt-status --region-name ap-south-2
aws sts get-caller-identity --region ap-south-2
```

**Fix →**

1. Enable the region: `aws account enable-region --region-name ap-south-2`
2. Wait for enablement

**Prevent →** Document region opt-ins in the landing zone.

</details>

### Q25. Root user lost MFA device. How to recover? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Use the account recovery flow.

**Diagnose →**

```bash
Sign in as root and choose the troubleshooting link
Verify email and phone
Contact AWS Support if required
```

**Fix →**

1. Re-register MFA after verification
2. Review recent activity

**Prevent →** Register multiple MFA devices for root.

</details>

## VPC and Connectivity

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. `ssh: connect to host 3.250.1.1 port 22: Connection timed out`. Walk the checks. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Timeout means packets are dropped: SG, NACL, route, public IP, or an on-instance firewall.

**Diagnose →**

```bash
aws ec2 describe-instances --instance-ids i-0abc --query 'Reservations[].Instances[].[PublicIpAddress,SecurityGroups,SubnetId]'
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-0abc
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=subnet-0abc
aws ec2 describe-network-insights-analyses
```

**Fix →**

1. Add SG rule for your IP
2. Ensure route `0.0.0.0/0` to the IGW
3. Fix NACL rules
4. Prefer SSM Session Manager

**Prevent →** Remove public SSH and use SSM.

</details>

### Q27. `ssh: connect to host ... port 22: Connection refused`. How is it different from a timeout? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Refused means the packet arrived but nothing is listening, or the OS firewall rejected.

**Diagnose →**

```bash
aws ec2 get-console-output --instance-id i-0abc --latest --output text | tail -50
aws ssm start-session --target i-0abc
sudo ss -ltnp | grep 22
sudo systemctl status sshd
```

**Fix →**

1. Start `sshd`
2. Fix `sshd_config`
3. Check `ufw` or `iptables`

**Prevent →** Bake sshd settings into the AMI.

</details>

### Q28. `Permission denied (publickey)` on a new EC2 instance. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong key pair, wrong username, or wrong permissions on the key file.

**Diagnose →**

```bash
ssh -vvv -i key.pem ec2-user@host
chmod 400 key.pem
aws ec2 describe-instances --instance-ids i-0abc --query 'Reservations[].Instances[].KeyName'
```

**Fix →**

1. Use correct user: `ec2-user`, `ubuntu`, `admin`
2. Use the right key
3. Use EC2 Instance Connect or SSM to inject a new key

**Prevent →** Document usernames per AMI.

</details>

### Q29. Private instances cannot reach the internet; `curl: (28) Connection timed out after 10000 milliseconds`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NAT gateway missing, route absent, or NAT in a subnet without an IGW route.

**Diagnose →**

```bash
aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=vpc-0abc
aws ec2 describe-route-tables --route-table-ids rtb-0abc
aws ec2 describe-subnets --subnet-ids <nat-subnet>
curl -v https://example.com
```

**Fix →**

1. Create the NAT in a public subnet
2. Fix routes
3. Check NACL and SG egress

**Prevent →** Test egress in IaC pipelines.

</details>

### Q30. NAT gateway shows `ErrorPortAllocation` in CloudWatch and connections drop. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many concurrent connections to the same destination from one NAT IP.

**Diagnose →**

```bash
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway --metric-name ErrorPortAllocation --dimensions Name=NatGatewayId,Value=nat-0abc --start-time 2026-11-08T00:00:00Z --end-time 2026-11-09T00:00:00Z --period 300 --statistics Sum
aws ec2 describe-nat-gateways
```

**Fix →**

1. Add secondary EIPs or more NAT gateways
2. Spread across AZs
3. Reuse connections and reduce churn

**Prevent →** Alarm on port allocation errors.

</details>

### Q31. VPC peering is `active` but ping fails. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing routes, SG rule, or NACL.

**Diagnose →**

```bash
aws ec2 describe-vpc-peering-connections
aws ec2 describe-route-tables --filters Name=route.vpc-peering-connection-id,Values=pcx-0abc
aws ec2 describe-security-groups --group-ids sg-0abc
```

**Fix →**

1. Add routes both ways
2. Allow ICMP or app port in SG
3. Fix NACL

**Prevent →** Use Reachability Analyzer in CI.

</details>

### Q32. VPC Reachability Analyzer says `BLACKHOLE`. What does it mean? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A route points to a deleted or inactive target.

**Diagnose →**

```bash
aws ec2 create-network-insights-path --source i-0abc --destination i-0def --protocol tcp --destination-port 443
aws ec2 start-network-insights-analysis --network-insights-path-id nip-0abc
aws ec2 describe-route-tables
```

**Fix →**

1. Remove or fix the blackhole route (deleted NAT, TGW or peering)
2. Re-run the analysis

**Prevent →** Alert on blackhole routes via Config.

</details>

### Q33. Transit Gateway attached VPCs cannot talk to each other. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing association or propagation in TGW route tables, or VPC routes.

**Diagnose →**

```bash
aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id tgw-rtb-0abc
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id tgw-rtb-0abc --filters Name=state,Values=active
aws ec2 describe-route-tables
```

**Fix →**

1. Associate attachments with the right route table
2. Enable propagation
3. Add VPC routes to the TGW

**Prevent →** Model TGW routing with diagrams.

</details>

### Q34. Site-to-Site VPN tunnel is `DOWN` on one side. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IKE mismatch, wrong pre-shared key, or on-prem device idle with no DPD.

**Diagnose →**

```bash
aws ec2 describe-vpn-connections --vpn-connection-ids vpn-0abc --query 'VpnConnections[].VgwTelemetry'
# check device logs for IKE phase 1 and 2
```

**Fix →**

1. Align IKE version and proposals
2. Reapply the configuration download
3. Enable DPD and keepalives

**Prevent →** Monitor `TunnelState`.

</details>

### Q35. Elastic IP association fails: `Resource ... is already associated with a different resource`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** EIP attached elsewhere.

**Diagnose →**

```bash
aws ec2 describe-addresses --allocation-ids eipalloc-0abc
```

**Fix →**

1. Disassociate then associate, or use `--allow-reassociation`

**Prevent →** Track EIPs in IaC.

</details>

### Q36. `Unable to resolve host` inside a VPC though internet is available. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DNS support disabled, or custom DHCP option set points to unreachable DNS.

**Diagnose →**

```bash
aws ec2 describe-vpc-attribute --vpc-id vpc-0abc --attribute enableDnsSupport
aws ec2 describe-dhcp-options
cat /etc/resolv.conf
dig @169.254.169.253 example.com
```

**Fix →**

1. Enable DNS support and hostnames
2. Fix DHCP options

**Prevent →** Keep default DHCP options unless needed.

</details>

### Q37. Private hosted zone names do not resolve from a peered VPC. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Zone not associated with that VPC.

**Diagnose →**

```bash
aws route53 get-hosted-zone --id Z123
dig db.internal.example.com
```

**Fix →**

1. Associate the VPC (cross-account needs authorization)
2. Enable DNS resolution on peering

**Prevent →** Use Resolver rules shared via RAM.

</details>

### Q38. Interface endpoint created but SDK calls still go to the internet. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Private DNS disabled or no DNS support.

**Diagnose →**

```bash
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-0abc
dig sqs.eu-west-1.amazonaws.com
```

**Fix →**

1. Enable PrivateDns on the endpoint
2. Fix VPC DNS attributes

**Prevent →** Verify DNS answers in post-deploy test.

</details>

### Q39. S3 gateway endpoint blocks access: `AccessDenied` through VPC endpoint. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Endpoint policy or bucket policy restricts.

**Diagnose →**

```bash
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.eu-west-1.s3
aws s3api get-bucket-policy --bucket b
```

**Fix →**

1. Allow the bucket in endpoint policy
2. Add needed AWS managed buckets such as package repos

**Prevent →** Test endpoint policies with real workloads.

</details>

### Q40. Security group references do not work across peered VPCs in different regions. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cross-region peering cannot reference SGs.

**Diagnose →**

```bash
aws ec2 describe-security-groups --group-ids sg-0abc
```

**Fix →**

1. Use CIDR rules for cross-region peers

**Prevent →** Document limits.

</details>

### Q41. Security group rule limit: `RulesPerSecurityGroupLimitExceeded`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many rules in one SG.

**Diagnose →**

```bash
aws ec2 describe-security-groups --group-ids sg-0abc --query 'SecurityGroups[].IpPermissions | length(@)'
```

**Fix →**

1. Use prefix lists
2. Split groups
3. Request quota

**Prevent →** Use managed prefix lists.

</details>

### Q42. Return traffic is blocked even though SG allows; NACL is suspected. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NACL is stateless; ephemeral ports not allowed.

**Diagnose →**

```bash
aws ec2 describe-network-acls --network-acl-ids acl-0abc
# check flow logs for REJECT on high ports
```

**Fix →**

1. Allow ephemeral `1024-65535` outbound and inbound as needed

**Prevent →** Keep NACLs default unless required.

</details>

### Q43. Instances in a new subnet cannot get an IP: `InsufficientFreeAddressesInSubnet`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Subnet exhausted.

**Diagnose →**

```bash
aws ec2 describe-subnets --subnet-ids subnet-0abc --query 'Subnets[].AvailableIpAddressCount'
```

**Fix →**

1. Launch in another subnet
2. Add subnets
3. Add secondary CIDR

**Prevent →** Monitor available IPs.

</details>

### Q44. Asymmetric routing breaks a stateful firewall behind GWLB. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Flow returns via a different AZ.

**Diagnose →**

```bash
aws ec2 describe-transit-gateway-vpc-attachments --query 'TransitGatewayVpcAttachments[].Options'
# check flow logs per AZ
```

**Fix →**

1. Enable appliance mode on the TGW attachment
2. Keep route per AZ consistent

**Prevent →** Test failover per AZ.

</details>

### Q45. `Network Load Balancer` targets show unhealthy but the app works when tested directly. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SG on targets blocks NLB source, or health check port mismatch.

**Diagnose →**

```bash
aws elbv2 describe-target-health --target-group-arn <arn>
aws elbv2 describe-target-groups --target-group-arns <arn>
```

**Fix →**

1. Allow health check source ranges
2. Fix port and protocol

**Prevent →** Use client IP preservation rules carefully.

</details>

### Q46. Direct Connect virtual interface BGP session stays `Down`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** VLAN, ASN or MD5 mismatch.

**Diagnose →**

```bash
aws directconnect describe-virtual-interfaces
# check router BGP summary
```

**Fix →**

1. Align VLAN ID, ASN, auth key and IPs
2. Contact the colo provider if layer 2 is down

**Prevent →** Keep a config checklist.

</details>

### Q47. `RequestLimitExceeded` from the EC2 API during mass operations. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** API throttling.

**Diagnose →**

```bash
aws ec2 describe-instances --max-items 50
```

**Fix →**

1. Add exponential backoff and jitter
2. Use pagination
3. Reduce polling

**Prevent →** Use the SDK retry modes (`adaptive`).

</details>

### Q48. Cannot delete a VPC: `The vpc has dependencies and cannot be deleted`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ENIs, endpoints, NAT or gateways still exist.

**Diagnose →**

```bash
aws ec2 describe-network-interfaces --filters Name=vpc-id,Values=vpc-0abc
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-0abc
aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=vpc-0abc
```

**Fix →**

1. Delete dependencies in order: instances, endpoints, NAT, IGW, subnets, SGs, route tables, VPC

**Prevent →** Manage VPCs with IaC for clean teardown.

</details>


### Q49. SSH and small HTTP requests work across a VPN / Transit Gateway, but large file transfers and TLS handshakes hang. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An MTU mismatch. Instances use jumbo frames (9001 MTU) inside the VPC, but Site-to-Site VPN supports at most 1446 bytes and Transit Gateway 8500. Small packets fit; big ones need fragmentation. Path MTU Discovery fails because ICMP "fragmentation needed" (type 3 code 4) is blocked by a security group, NACL or on-prem firewall.

**Diagnose →**

```bash
ip link show eth0 | grep mtu                 # 9001 on most Nitro instances
ping -M do -s 1418 10.20.0.15                # 1418 + 28 = 1446, should pass
ping -M do -s 1472 10.20.0.15                # fails => MTU is the problem
tracepath 10.20.0.15                         # shows the pmtu hop by hop
aws ec2 describe-security-groups --group-ids sg-0abc --query 'SecurityGroups[].IpPermissions[?IpProtocol==`icmp`]'
```

**Fix →**

1. Allow inbound ICMP type 3 code 4 (Destination Unreachable / Fragmentation Needed) in the SGs and NACLs on both sides
2. Enable TCP MSS clamping on the customer gateway (for example MSS 1379 for IPsec)
3. As a short-term workaround, lower the instance MTU: `sudo ip link set dev eth0 mtu 1400`

**Prevent →** Treat ICMP unreachable as required traffic in baseline SGs, and document per-path MTU (VPN 1446, TGW 8500, internet 1500).

</details>

### Q50. `InvalidSubnet.Conflict: The CIDR '10.0.1.0/24' conflicts with another subnet` when creating a subnet. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CIDR overlaps an existing subnet in the same VPC. A related error, `InvalidSubnet.Range: The CIDR '10.1.0.0/24' is invalid`, means the block is outside the VPC CIDR(s) or smaller than /28.

**Diagnose →**

```bash
aws ec2 describe-vpcs --vpc-ids vpc-0abc --query 'Vpcs[].CidrBlockAssociationSet[].CidrBlock'
aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-0abc \
  --query 'Subnets[].[SubnetId,CidrBlock,AvailabilityZone]' --output table
```

**Fix →**

1. Pick a free, non-overlapping range inside the VPC CIDR
2. If the VPC is full, add a secondary CIDR: `aws ec2 associate-vpc-cidr-block --vpc-id vpc-0abc --cidr-block 10.1.0.0/16`
3. Then create the subnet inside the new range

**Prevent →** Plan subnets with IPAM or a documented CIDR map. Calculate subnets in Terraform with `cidrsubnet()` rather than typing them by hand.

</details>

---

## EC2, Auto Scaling and Load Balancing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. `InsufficientInstanceCapacity: We currently do not have sufficient m5.large capacity in the Availability Zone you requested (us-east-1a).` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AWS has no spare capacity for that instance type in that AZ right now. Your account is fine; this is not a quota problem.

**Diagnose →**

```bash
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters Name=instance-type,Values=m5.large --region us-east-1
aws autoscaling describe-scaling-activities --auto-scaling-group-name web-asg --max-items 5
```

**Fix →**

1. Retry in a few minutes or launch in another AZ
2. Try a similar type (`m6i.large`, `m5a.large`, `m7i.large`)
3. For an ASG, use a mixed instances policy with several types and subnets in every AZ

**Prevent →** Never pin critical workloads to one type in one AZ. For events where you know you will need capacity, buy an On-Demand Capacity Reservation.

</details>

### Q52. An instance shows `1/2 checks passed` and you cannot reach it. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** If the **system** status check fails, the problem is AWS hardware or the host (power, network, physical host). If the **instance** status check fails, the problem is inside the OS: a kernel panic, a broken network config, a full disk or bad `/etc/fstab`.

**Diagnose →**

```bash
aws ec2 describe-instance-status --instance-ids i-0abc \
  --query 'InstanceStatuses[].[SystemStatus.Status,InstanceStatus.Status]'
aws ec2 get-console-output --instance-id i-0abc --latest --output text | tail -50
aws ec2 get-console-screenshot --instance-id i-0abc --query ImageData --output text | base64 -d > shot.jpg
```

**Fix →**

1. System check failed: stop and start the instance (not reboot), which moves it to new hardware
2. Instance check failed: read the console output, then fix through the EC2 Serial Console or by attaching the root volume to a rescue instance

**Prevent →** Add a CloudWatch alarm on `StatusCheckFailed_System` with the `recover` action, and run stateless instances behind an ASG.

</details>

### Q53. `VcpuLimitExceeded: You have requested more vCPU capacity than your current vCPU limit of 32 allows for the instance bucket that the specified instance type belongs to.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You have hit the account's vCPU-based On-Demand quota for that instance family group in this Region (for example, "Running On-Demand Standard (A, C, D, H, I, M, R, T, Z) instances").

**Diagnose →**

```bash
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].InstanceType' --output text | tr '\t' '\n' | sort | uniq -c
```

**Fix →**

1. Terminate unused instances
2. Request an increase: `aws service-quotas request-service-quota-increase --service-code ec2 --quota-code L-1216C47A --desired-value 128`

**Prevent →** Watch quota usage with CloudWatch `AWS/Usage` metrics and Trusted Advisor. Raise quotas before launches and DR tests, not during them.

</details>

### Q54. A new instance goes `pending` → `terminated` within seconds with state reason `Client.InternalError: Client error on launch`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Usually the EBS volumes are encrypted with a customer managed KMS key that the launching principal (or the ASG service-linked role) cannot use. It can also mean a snapshot is missing or the volume limit was exceeded.

**Diagnose →**

```bash
aws ec2 describe-instances --instance-ids i-0abc --query 'Reservations[].Instances[].StateReason'
aws ec2 describe-images --image-ids ami-0abc --query 'Images[].BlockDeviceMappings[].Ebs'
aws kms get-key-policy --key-id <key-id> --policy-name default
```

**Fix →**

1. Allow the launching principal and `AWSServiceRoleForAutoScaling` to use the key: `kms:CreateGrant`, `kms:Decrypt`, `kms:GenerateDataKeyWithoutPlaintext`, `kms:ReEncrypt*` and `kms:DescribeKey`
2. For cross-account AMIs, update the key policy in the owning account and create a grant for the service-linked role

**Prevent →** Keep a tested key-policy template for EBS keys that names the ASG service-linked role.

</details>

### Q55. Session Manager: `An error occurred (TargetNotConnected) when calling the StartSession operation: i-0abc is not connected.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SSM Agent is not registered. Common reasons: the instance profile does not have `AmazonSSMManagedInstanceCore`, the private subnet has no NAT or `ssm`, `ssmmessages` and `ec2messages` endpoints, the agent is stopped, or IMDS is disabled.

**Diagnose →**

```bash
aws ssm describe-instance-information --filters Key=InstanceIds,Values=i-0abc
aws ec2 describe-iam-instance-profile-associations --filters Name=instance-id,Values=i-0abc
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-0abc --query 'VpcEndpoints[].ServiceName'
# on the box via serial console:
sudo systemctl status amazon-ssm-agent; sudo tail /var/log/amazon/ssm/amazon-ssm-agent.log
```

**Fix →**

1. Attach a role that has `AmazonSSMManagedInstanceCore`
2. Add the three interface endpoints, with private DNS on and an SG that allows 443 from the instances
3. Restart the agent, or turn on Default Host Management Configuration

**Prevent →** Bake the agent into AMIs and turn on Default Host Management Configuration across the organization.

</details>

### Q56. User data did not run: packages are missing and the app never started. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The script has no shebang, or it was saved with CRLF line endings. It may have failed part-way. Or it already ran once: by default, user data runs only on the first boot, so editing it on a stopped instance does nothing.

**Diagnose →**

```bash
aws ec2 describe-instance-attribute --instance-id i-0abc --attribute userData \
  --query UserData.Value --output text | base64 -d | head
sudo cat /var/log/cloud-init-output.log
sudo grep -i error /var/log/cloud-init.log
cloud-init status --long
```

**Fix →**

1. Start the script with `#!/bin/bash` and add `set -euxo pipefail` so failures show up
2. Convert the file to LF line endings
3. To re-run it for testing, run `sudo cloud-init clean && sudo reboot`

**Prevent →** Keep user data small: bootstrap a config tool or pull a versioned script. Better still, bake an AMI with Packer.

</details>

### Q57. `curl http://169.254.169.254/latest/meta-data/` returns `401 - Unauthorized` on a new instance. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance requires IMDSv2 (`HttpTokens=required`), and the request has no session token. Older SDKs and tools that only speak IMDSv1 break the same way.

**Diagnose →**

```bash
aws ec2 describe-instances --instance-ids i-0abc \
  --query 'Reservations[].Instances[].MetadataOptions'
TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600')
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

**Fix →**

1. Update scripts to use the token flow, and upgrade SDKs and agents
2. For containers on the host, set `HttpPutResponseHopLimit` to 2: `aws ec2 modify-instance-metadata-options --instance-id i-0abc --http-put-response-hop-limit 2`

**Prevent →** Make IMDSv2 the default at account level (`aws ec2 modify-instance-metadata-defaults`), and track the `MetadataNoToken` metric before you enforce it.

</details>

### Q58. `InvalidAMIID.NotFound: The image id '[ami-0abc123]' does not exist`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AMI IDs are Regional, so the ID belongs to another Region. Other causes: the AMI was deregistered, or it is private and not shared with your account.

**Diagnose →**

```bash
aws configure get region
aws ec2 describe-images --image-ids ami-0abc123 --region eu-west-1
aws ec2 describe-image-attribute --image-id ami-0abc123 --attribute launchPermission
```

**Fix →**

1. Use the AMI ID for the right Region, or copy the AMI there: `aws ec2 copy-image --source-region eu-west-1 --source-image-id ami-0abc123 --region us-east-1 --name app-v12`
2. Look up public AMIs through SSM parameters instead of hard-coding IDs: `aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`

**Prevent →** Resolve AMIs dynamically (SSM parameter or a data source filtered by tag) and copy golden AMIs to every Region you use.

</details>

### Q59. An ASG keeps launching and terminating instances in a loop; activity history says `an instance was taken out of service in response to an ELB system health check failure`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ASG uses ELB health checks, but the health check grace period is shorter than the app's boot time. It can also be a wrong health check path, or a target SG that does not allow traffic from the ALB.

**Diagnose →**

```bash
aws autoscaling describe-scaling-activities --auto-scaling-group-name web-asg --max-items 10
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names web-asg \
  --query 'AutoScalingGroups[].[HealthCheckType,HealthCheckGracePeriod]'
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

**Fix →**

1. Raise the grace period above the real boot time: `aws autoscaling update-auto-scaling-group --auto-scaling-group-name web-asg --health-check-grace-period 300`
2. Fix the health check path and port, and the SG rule from the ALB SG
3. To debug a live instance, suspend `ReplaceUnhealthy` for a while: `aws autoscaling suspend-processes --auto-scaling-group-name web-asg --scaling-processes ReplaceUnhealthy`

**Prevent →** Use a cheap `/healthz` endpoint, plus a lifecycle hook or warm pool for slow boots.

</details>

### Q60. ASG activity fails: `Value (app-profile) for parameter iamInstanceProfile.name is invalid. Invalid IAM Instance Profile name`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance profile does not exist, or it was just created and has not propagated yet. Another common cause: the role was created in the console with a different profile name, or created through the CLI without a profile.

**Diagnose →**

```bash
aws iam get-instance-profile --instance-profile-name app-profile
aws iam list-instance-profiles-for-role --role-name app-role
```

**Fix →**

1. Create the profile and add the role to it: `aws iam create-instance-profile --instance-profile-name app-profile` then `aws iam add-role-to-instance-profile --instance-profile-name app-profile --role-name app-role`
2. Wait a few seconds after creating it, then retry the launch

**Prevent →** In IaC, create `aws_iam_instance_profile` explicitly and reference it, so the dependency order is correct.

</details>

### Q61. An ASG instance refresh fails with `Instance refresh failed: ... could not maintain the minimum healthy percentage`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New instances from the updated launch template never become healthy, so the refresh cannot replace more instances without dropping below `MinHealthyPercentage`. Typical reasons: a bad AMI, broken user data or a failing health check. A checkpoint or alarm-based rollback may also have stopped it.

**Diagnose →**

```bash
aws autoscaling describe-instance-refreshes --auto-scaling-group-name web-asg --max-records 1
aws autoscaling describe-scaling-activities --auto-scaling-group-name web-asg --max-items 10
aws elbv2 describe-target-health --target-group-arn <tg-arn> \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State,TargetHealth.Reason]'
```

**Fix →**

1. Roll back: `aws autoscaling rollback-instance-refresh --auto-scaling-group-name web-asg`, or cancel it and fix the template
2. Debug one new instance (console output, cloud-init log), then start the refresh again with `--preferences '{"MinHealthyPercentage":90,"InstanceWarmup":300,"AutoRollback":true}'`

**Prevent →** Turn on `AutoRollback` with CloudWatch alarms, use checkpoints for canary-style refreshes, and test AMIs in staging first.

</details>

### Q62. ALB returns `502 Bad Gateway`. Where do you look? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target closed the connection, sent a malformed response, or reset during a request. Common reasons: the app crashed, or its keep-alive timeout is shorter than the ALB idle timeout (60s), so the target drops idle connections the ALB still reuses. A Lambda target that returns an invalid response also causes 502.

**Diagnose →**

```bash
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name HTTPCode_ELB_502_Count \
  --dimensions Name=LoadBalancer,Value=app/web/abc123 --statistics Sum --period 60 \
  --start-time $(date -u -d '-1 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ)
aws elbv2 describe-load-balancer-attributes --load-balancer-arn <alb-arn>
# access logs: elb_status_code=502 with target_status_code="-" means no response from target
```

**Fix →**

1. Set the app keep-alive timeout above the ALB idle timeout (for example, Node.js `server.keepAliveTimeout = 65000`, Nginx `keepalive_timeout 75s`)
2. Fix the app crashes and response headers

**Prevent →** Turn on ALB access logs to S3 and alarm on the ratio of `HTTPCode_ELB_5XX_Count` to request count.

</details>

### Q63. ALB returns `504 Gateway Timeout` on a report endpoint. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target did not respond before the ALB idle timeout (60s by default). Either the request is too slow, or the target is unreachable (blocked by an SG or NACL), so the connection attempt times out.

**Diagnose →**

```bash
aws elbv2 describe-load-balancer-attributes --load-balancer-arn <alb-arn> \
  --query "Attributes[?Key=='idle_timeout.timeout_seconds']"
# access logs: high target_processing_time, or request_processing_time -1
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/web/abc123 --extended-statistics p99 --period 60 \
  --start-time $(date -u -d '-1 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ)
```

**Fix →**

1. As a short-term fix, raise the idle timeout: `aws elbv2 modify-load-balancer-attributes --load-balancer-arn <arn> --attributes Key=idle_timeout.timeout_seconds,Value=180`
2. As the real fix, make long jobs asynchronous: return 202, process through a queue, and let the client poll or receive a webhook

**Prevent →** Track `TargetResponseTime` p99 per route, and keep synchronous requests well under the timeout.

</details>

### Q64. ALB returns `503 Service Temporarily Unavailable` immediately. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target group behind the matched listener rule has no registered targets, or all of its targets are in AZs the ALB is not enabled in. If every target is unhealthy, the ALB "fails open" and still routes traffic, so 503 usually means *no targets*.

**Diagnose →**

```bash
aws elbv2 describe-rules --listener-arn <listener-arn> \
  --query 'Rules[].[Priority,Actions[0].TargetGroupArn]'
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws elbv2 describe-load-balancers --load-balancer-arns <alb-arn> --query 'LoadBalancers[].AvailabilityZones'
```

**Fix →**

1. Register targets, or attach the ASG or ECS service to the right target group
2. Enable the ALB in the AZs where the targets run

**Prevent →** Alarm on `HealthyHostCount < 1` per target group, and check that targets are attached in your IaC.

</details>

### Q65. Target health shows `Target.ResponseCodeMismatch: Health checks failed with these codes: [404]`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The health check path does not exist on the app, or it redirects (301/302) to HTTPS or a login page. By default, the matcher only accepts `200`.

**Diagnose →**

```bash
aws elbv2 describe-target-groups --target-group-arns <tg-arn> \
  --query 'TargetGroups[].[HealthCheckPath,HealthCheckPort,Matcher.HttpCode]'
# from an instance in the same subnet:
curl -i http://10.0.1.25:8080/health
```

**Fix →**

1. Point the health check at a real endpoint: `aws elbv2 modify-target-group --target-group-arn <tg-arn> --health-check-path /healthz`
2. Or widen the matcher if the endpoint legitimately returns another code: `--matcher HttpCode=200-299`

**Prevent →** Every service exposes an unauthenticated `/healthz` that does not redirect, and IaC defines it next to the target group.

</details>

### Q66. Target health shows `Target.Timeout: Request timed out` for every target. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ALB cannot reach the targets. The usual reasons: the target SG does not allow the health check port from the ALB SG, a NACL blocks it, the app listens only on `127.0.0.1`, or the health check uses the wrong port.

**Diagnose →**

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws ec2 describe-security-groups --group-ids <target-sg> --query 'SecurityGroups[].IpPermissions'
# on the target:
sudo ss -lntp | grep 8080      # 127.0.0.1:8080 means not reachable from the ALB
```

**Fix →**

1. Allow the app port from the ALB SG: `aws ec2 authorize-security-group-ingress --group-id <target-sg> --protocol tcp --port 8080 --source-group <alb-sg>`
2. Bind the app to `0.0.0.0`
3. Check that the NACL allows the ephemeral return ports

**Prevent →** Use SG-to-SG references in IaC rather than CIDRs, and run Reachability Analyzer from the ALB ENI to the target.

</details>

### Q67. ALB access logs show many `460` responses and some `463`. What do they mean? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** These are ALB-specific codes. **460** means the client closed the connection before the idle timeout ran out, usually because the client timeout is shorter than the backend latency. **463** means the `X-Forwarded-For` header has more than 30 IP addresses, usually from a proxy loop or a misbehaving client.

**Diagnose →**

```bash
aws s3 cp s3://alb-logs/AWSLogs/111111111111/elasticloadbalancing/us-east-1/2026/11/09/ . --recursive
zcat *.log.gz | awk '$9==460 {print $13,$14}' | sort | uniq -c | sort -rn | head   # method + URL
zcat *.log.gz | awk '$9==460 {print $7}' | sort -n | tail                         # target_processing_time
```

**Fix →**

1. For 460, find the slow endpoints and fix their latency, or align the client and SDK timeouts with the server
2. For 463, find the proxy chain that keeps appending to XFF, and break the loop (for example, CloudFront → ALB → nginx → the same ALB)

**Prevent →** Load ALB logs into Athena with a saved table, and alarm on the p99 latency of the top routes.

</details>

### Q68. Creating an HTTPS listener fails with `CertificateNotFound: Certificate 'arn:aws:acm:...' not found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ACM certificate is in a different Region from the ALB, it is still `PENDING_VALIDATION`, or the ARN belongs to another account.

**Diagnose →**

```bash
aws acm list-certificates --region us-east-1 \
  --query 'CertificateSummaryList[].[DomainName,Status,CertificateArn]'
aws acm describe-certificate --certificate-arn <arn> \
  --query 'Certificate.DomainValidationOptions[].[DomainName,ValidationStatus,ResourceRecord]'
```

**Fix →**

1. Request or import the certificate in the ALB's Region
2. Finish DNS validation by creating the CNAME that ACM shows (Route 53 can do it in one click)
3. Use the ISSUED ARN in the listener

**Prevent →** Manage certificates in IaC with `aws_acm_certificate_validation`, so the listener waits until the certificate is issued.

</details>

### Q69. Behind an NLB, one target gets most of the traffic while the others sit idle. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An NLB balances per connection (flow hash), not per request. A few long-lived connections, such as gRPC, HTTP/2, WebSockets or pooled DB clients, stick to whichever target they first hit. Cross-zone load balancing is off by default on an NLB, so an AZ with fewer targets gets proportionally more load per target.

**Diagnose →**

```bash
aws elbv2 describe-load-balancer-attributes --load-balancer-arn <nlb-arn> \
  --query "Attributes[?Key=='load_balancing.cross_zone.enabled']"
aws elbv2 describe-target-health --target-group-arn <tg-arn>
# per-target ActiveFlowCount / NewFlowCount in CloudWatch (AWS/NetworkELB)
```

**Fix →**

1. Turn on cross-zone: `aws elbv2 modify-load-balancer-attributes --load-balancer-arn <nlb-arn> --attributes Key=load_balancing.cross_zone.enabled,Value=true`
2. For HTTP/2 or gRPC, use an ALB, which balances per request. Or make clients recycle connections regularly (max connection age)
3. Spread targets evenly across AZs

**Prevent →** Pick the LB for the protocol: an ALB for request-level balancing, an NLB for raw TCP, static IPs and very high throughput.

</details>

### Q70. Spot instances disappear and the state reason is `Server.SpotInstanceTermination`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AWS reclaimed the Spot capacity. You get a 2-minute interruption notice, and possibly an earlier rebalance recommendation.

**Diagnose →**

```bash
aws ec2 describe-instances --instance-ids i-0abc --query 'Reservations[].Instances[].StateReason'
aws ec2 describe-spot-instance-requests --filters Name=instance-id,Values=i-0abc \
  --query 'SpotInstanceRequests[].Status'
# on the instance, the notice endpoint:
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/spot/instance-action
```

**Fix →**

1. Diversify across many instance types and all AZs, and use the `price-capacity-optimized` allocation strategy
2. Handle the interruption notice: drain from the LB and checkpoint work. On EKS, use Karpenter or the Node Termination Handler
3. Turn on Capacity Rebalancing on the ASG

**Prevent →** Run only stateless, interruptible workloads on Spot, and keep an On-Demand base capacity for the minimum.

</details>

### Q71. After a kernel update and reboot, the instance never comes back: status check fails and SSH times out. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The new kernel or initramfs does not boot (missing NVMe or ENA modules), GRUB points at a bad entry, or `/etc/fstab` references a device that no longer exists.

**Diagnose →**

```bash
aws ec2 get-console-output --instance-id i-0abc --latest --output text | tail -80
# interactive: EC2 Serial Console (needs account-level enablement and a password user)
aws ec2 enable-serial-console-access
aws ec2-instance-connect send-serial-console-ssh-public-key --instance-id i-0abc --serial-port 0 --ssh-public-key file://key.pub
```

**Fix →**

1. Use the Serial Console to pick the previous kernel in GRUB, then fix it
2. Or use a rescue instance: stop the instance, detach the root volume, attach it to a rescue instance in the same AZ, mount it, fix `grub` or `fstab` (`nofail`, UUIDs), then reattach as `/dev/xvda` and start
3. AWSSupport-ExecuteEC2Rescue automation can do this for you

**Prevent →** Patch through AMI rebuilds and rolling replacement instead of in place, and take snapshots before kernel changes.

</details>

### Q72. Changing an old instance to `c5.large` fails: `Enhanced networking with the Elastic Network Adapter (ENA) is required for the 'c5.large' instance type. Ensure that your instance 'i-0abc' is enabled for ENA.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance came from an old AMI built for Xen (for example m4 or t2). Nitro types need the ENA driver for networking and NVMe drivers for EBS, and the instance attribute `enaSupport` must be true.

**Diagnose →**

```bash
aws ec2 describe-instances --instance-ids i-0abc --query 'Reservations[].Instances[].EnaSupport'
# on the instance:
modinfo ena; lsmod | grep -E 'ena|nvme'
```

**Fix →**

1. Install or confirm the ENA and NVMe drivers inside the OS, and add them to the initramfs
2. Stop the instance and run `aws ec2 modify-instance-attribute --instance-id i-0abc --ena-support`
3. Change the type, and change `/etc/fstab` to use UUIDs, because device names become `/dev/nvme*`

**Prevent →** Use current AMIs (AL2023, Ubuntu 22.04/24.04) and move off previous-generation types in planned windows.

</details>

### Q73. A `t3.medium` web server becomes slow every afternoon, but CPU never goes above ~20%. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance is a burstable type in `standard` credit mode. Once `CPUCreditBalance` reaches 0, it is throttled to its baseline (20% for t3.medium). CloudWatch then shows CPU flat at the baseline.

**Diagnose →**

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUCreditBalance \
  --dimensions Name=InstanceId,Value=i-0abc --statistics Minimum --period 300 \
  --start-time $(date -u -d '-1 day' +%FT%TZ) --end-time $(date -u +%FT%TZ)
aws ec2 describe-instance-credit-specifications --instance-ids i-0abc
```

**Fix →**

1. Switch to unlimited mode: `aws ec2 modify-instance-credit-specification --instance-credit-specification 'InstanceId=i-0abc,CpuCredits=unlimited'` (you pay for surplus credits)
2. Or move to a fixed-performance type (m7i, c7g) if the load is sustained

**Prevent →** Alarm on `CPUCreditBalance` and use T instances only for workloads that really are bursty.

</details>

### Q74. During scale-in, the ASG terminates instances in the middle of long batch jobs. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Scale-in picks instances using the termination policy and knows nothing about the work on them. Without lifecycle hooks or protection, jobs are killed.

**Diagnose →**

```bash
aws autoscaling describe-scaling-activities --auto-scaling-group-name workers --max-items 10
aws autoscaling describe-lifecycle-hooks --auto-scaling-group-name workers
aws autoscaling describe-auto-scaling-instances --query 'AutoScalingInstances[].[InstanceId,ProtectedFromScaleIn]'
```

**Fix →**

1. The worker sets scale-in protection while it is busy: `aws autoscaling set-instance-protection --instance-ids i-0abc --auto-scaling-group-name workers --protected-from-scale-in` (and clears it when done)
2. Add a `autoscaling:EC2_INSTANCE_TERMINATING` lifecycle hook. The instance finishes or checkpoints its job, then calls `complete-lifecycle-action`
3. Scale on queue depth per instance instead of CPU

**Prevent →** Design jobs to be idempotent and checkpointed, and size the hook `HeartbeatTimeout` to the longest job.

</details>

### Q75. You updated the launch template with a new AMI, but new ASG instances still boot the old AMI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ASG references a specific template version, or `$Default`, and you created a new version without making it the default. New versions are not used until the ASG points at them.

**Diagnose →**

```bash
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names web-asg \
  --query 'AutoScalingGroups[].[LaunchTemplate,MixedInstancesPolicy.LaunchTemplate.LaunchTemplateSpecification]'
aws ec2 describe-launch-template-versions --launch-template-id lt-0abc \
  --query 'LaunchTemplateVersions[].[VersionNumber,DefaultVersion,LaunchTemplateData.ImageId]'
```

**Fix →**

1. Point the ASG at `$Latest`, or set the new version as default: `aws ec2 modify-launch-template --launch-template-id lt-0abc --default-version 7`
2. Start an instance refresh to replace the instances that are already running

**Prevent →** Have the pipeline create the version, update the ASG and start an instance refresh in one step.

</details>

---

## S3, EBS and EFS

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. `BucketAlreadyExists: The requested bucket name is not available. The bucket namespace is shared by all users of the system.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** S3 bucket names are global across all AWS accounts, and another account already owns this name. If *you* own it, you get `BucketAlreadyOwnedByYou` instead. That error is common when IaC re-runs a create in a different Region.

**Diagnose →**

```bash
aws s3api head-bucket --bucket my-app-logs   # 403 = exists in someone else's account, 404 = free
aws s3api list-buckets --query "Buckets[?Name=='my-app-logs']"
```

**Fix →**

1. Pick a unique name, for example with the account ID and Region: `my-app-logs-111111111111-us-east-1`
2. For `BucketAlreadyOwnedByYou`, import the bucket into state instead of creating it

**Prevent →** Use a naming convention with the account and Region in it, or the S3 account-regional namespace option where it is available. Never rely on a deleted name being free again.

</details>

### Q77. `PermanentRedirect: The bucket you are attempting to access must be addressed using the specified endpoint.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client is calling the wrong Regional endpoint for this bucket, usually because the profile Region is different from the bucket's Region, or an old SDK hard-codes `s3.amazonaws.com`.

**Diagnose →**

```bash
aws s3api get-bucket-location --bucket my-bucket     # null LocationConstraint = us-east-1
aws configure get region
aws s3 ls s3://my-bucket --debug 2>&1 | grep -i 'x-amz-bucket-region'
```

**Fix →**

1. Pass the right Region: `aws s3 ls s3://my-bucket --region eu-central-1`
2. Configure the SDK client with the bucket's Region

**Prevent →** Keep Region per environment in config, and include the Region in bucket names to make mismatches obvious.

</details>

### Q78. A partner account uploads objects into your bucket, but your own admins get `403 Forbidden` reading them. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Object Ownership is `ObjectWriter` (the legacy setting), so the uploading account owns the objects and the bucket policy cannot grant access to them. This happens when the uploader did not set `bucket-owner-full-control`.

**Diagnose →**

```bash
aws s3api get-bucket-ownership-controls --bucket shared-ingest
aws s3api get-object-acl --bucket shared-ingest --key incoming/file.csv
```

**Fix →**

1. Turn on Bucket owner enforced (ACLs disabled): `aws s3api put-bucket-ownership-controls --bucket shared-ingest --ownership-controls 'Rules=[{ObjectOwnership=BucketOwnerEnforced}]'`
2. That takes ownership of all objects, including existing ones. Re-check the bucket policy grants

**Prevent →** Use `BucketOwnerEnforced` everywhere (the default for new buckets since 2023), and control access only through policies.

</details>

### Q79. `aws s3 cp` gets `An error occurred (AccessDenied) when calling the GetObject operation` but the IAM policy allows `s3:GetObject` and the bucket policy is fine. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The object is encrypted with SSE-KMS using a customer managed key, and the caller has no `kms:Decrypt` on that key (through the key policy or IAM). For uploads, the missing permission is `kms:GenerateDataKey`.

**Diagnose →**

```bash
aws s3api head-object --bucket data --key reports/q3.parquet \
  --query '[ServerSideEncryption,SSEKMSKeyId]'
aws kms get-key-policy --key-id <key-arn> --policy-name default --output text
# CloudTrail: look for kms Decrypt with errorCode AccessDenied
```

**Fix →**

1. Grant `kms:Decrypt` on the key ARN to the role in IAM, and make sure the key policy allows IAM policies (the root principal statement) or names the role
2. For cross-account access, both the key policy and the caller's IAM policy must allow it

**Prevent →** Every S3 access policy template includes the matching KMS statement, and S3 Bucket Keys are on to reduce KMS calls.

</details>

### Q80. An S3 static website endpoint returns `403 Forbidden` with `Code: AccessDenied`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Block Public Access is on at the account or bucket level, or no bucket policy grants `s3:GetObject` to `*`. Also check that the index document exists exactly as named (keys are case-sensitive) and that objects are not KMS-encrypted (website endpoints cannot serve SSE-KMS objects).

**Diagnose →**

```bash
aws s3control get-public-access-block --account-id 111111111111
aws s3api get-public-access-block --bucket my-site
aws s3api get-bucket-policy --bucket my-site
aws s3api get-bucket-website --bucket my-site
aws s3 ls s3://my-site/index.html
```

**Fix →**

1. In production, keep the bucket private and put CloudFront with OAC in front of it
2. If a public website endpoint is truly intended, turn off BPA for that bucket only and add a read-only public policy for `arn:aws:s3:::my-site/*`

**Prevent →** Keep account-level BPA on, and serve all public content through CloudFront.

</details>

### Q81. High-throughput writes get `503 SlowDown: Please reduce your request rate.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You are above the per-prefix request rate (about 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD per second per partitioned prefix), and S3 has not repartitioned yet. All writes going to one hot prefix such as `logs/2026-11-09/` make it worse.

**Diagnose →**

```bash
aws s3api put-bucket-metrics-configuration --bucket data --id all --metrics-configuration Id=all
# then CloudWatch AWS/S3 metrics: AllRequests, 5xxErrors per FilterId
aws s3api list-objects-v2 --bucket data --prefix logs/2026-11-09/ --max-items 5
```

**Fix →**

1. Retry with exponential backoff. SDK retries do this, and `adaptive` mode helps
2. Spread keys across more prefixes (for example, a hash or shard prefix)
3. Batch small objects into larger ones

**Prevent →** Design key layouts for parallelism, and ramp traffic up gradually so S3 can scale partitions.

</details>

### Q82. `BucketNotEmpty: The bucket you tried to delete is not empty. You must delete all versions in the bucket.` even after `aws s3 rm --recursive`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Versioning is on. `aws s3 rm` only adds delete markers, and all older versions and delete markers remain.

**Diagnose →**

```bash
aws s3api get-bucket-versioning --bucket old-bucket
aws s3api list-object-versions --bucket old-bucket --max-items 5
```

**Fix →**

1. For small buckets, delete versions in batches:

```bash
aws s3api list-object-versions --bucket old-bucket \
  --query '{Objects: [Versions[].{Key:Key,VersionId:VersionId}, DeleteMarkers[].{Key:Key,VersionId:VersionId}][]}' \
  --max-items 1000 --output json > del.json
aws s3api delete-objects --bucket old-bucket --delete file://del.json
```

2. For big buckets, add a lifecycle rule that expires current and noncurrent versions and delete markers, wait a day, then delete the bucket

**Prevent →** In IaC, use `force_destroy` only for ephemeral buckets, and give every versioned bucket a noncurrent-version lifecycle rule.

</details>

### Q83. A browser upload to S3 fails: `has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bucket has no CORS configuration, or it does not list the origin, method (PUT/POST) or headers the browser sends in the preflight. Presigned PUTs with custom headers such as `Content-Type` need `AllowedHeaders`.

**Diagnose →**

```bash
aws s3api get-bucket-cors --bucket uploads
curl -i -X OPTIONS "https://uploads.s3.amazonaws.com/test.txt" \
  -H 'Origin: https://app.example.com' -H 'Access-Control-Request-Method: PUT'
```

**Fix →**

```bash
aws s3api put-bucket-cors --bucket uploads --cors-configuration '{
 "CORSRules":[{"AllowedOrigins":["https://app.example.com"],"AllowedMethods":["PUT","GET"],
 "AllowedHeaders":["*"],"ExposeHeaders":["ETag"],"MaxAgeSeconds":3000}]}'
```

If CloudFront is in front, forward the `Origin` header in the origin request policy.

**Prevent →** Keep the CORS config in IaC with explicit origins per environment. Never use `*` together with credentials.

</details>

### Q84. Presigned URLs return `SignatureDoesNotMatch` for some users and `Request has expired` for others. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `SignatureDoesNotMatch` means the request is different from what was signed: another HTTP method, a missing `Content-Type`, the wrong Region endpoint, or the URL was re-encoded. `Request has expired` means the expiry passed, or the URL was signed with temporary role credentials that expired earlier (a URL is valid only as long as the signing credentials).

**Diagnose →**

```bash
aws s3 presign s3://bucket/file.zip --expires-in 3600 --region us-east-1
# inspect X-Amz-Expires, X-Amz-Date, X-Amz-Credential in the URL
aws sts get-caller-identity   # signed with a role session? check its expiry
```

**Fix →**

1. Sign with the exact method and headers the client will send, using a Regional endpoint
2. For long-lived links, sign with credentials that last at least as long. SigV4 allows a maximum of 7 days
3. Fix client clock skew

**Prevent →** Generate URLs on demand with a short expiry, and serve large public downloads through CloudFront signed URLs.

</details>

### Q85. S3 Cross-Region Replication is configured, but new objects show `ReplicationStatus: FAILED`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The replication role does not have `s3:ReplicateObject` on the destination, or the destination bucket policy does not allow the role (cross-account). Objects are SSE-KMS but the rule has no `SseKmsEncryptedObjects` or KMS permissions in the destination Region. Versioning is off on one side. Or Object Lock is in use without the right configuration.

**Diagnose →**

```bash
aws s3api get-bucket-replication --bucket src
aws s3api head-object --bucket src --key path/obj --query ReplicationStatus
aws s3api get-bucket-versioning --bucket dst
# enable replication metrics + S3 event notifications for s3:Replication:OperationFailedReplication
```

**Fix →**

1. Fix the role permissions (source read, destination replicate, KMS decrypt on the source key and encrypt on the destination key)
2. Fix the destination bucket policy for cross-account
3. Re-replicate the failed and existing objects with S3 Batch Replication

**Prevent →** Turn on Replication Time Control metrics and alarm on `OperationsFailedReplication`.

</details>

### Q86. A lifecycle rule "expire after 30 days" has been on for months, yet storage is still growing. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bucket is versioned. Expiring current objects only adds delete markers, so noncurrent versions build up. Other causes: the rule's prefix or tag filter does not match the keys, or the objects are below the minimum size for transitions.

**Diagnose →**

```bash
aws s3api get-bucket-lifecycle-configuration --bucket logs
aws s3api get-bucket-versioning --bucket logs
aws s3api list-object-versions --bucket logs --prefix app/ --max-items 10 \
  --query 'Versions[?IsLatest==`false`].[Key,LastModified]'
```

**Fix →** Add `NoncurrentVersionExpiration` (for example, after 7 days) and `ExpiredObjectDeleteMarker: true`. Correct the filter prefix.

**Prevent →** Use S3 Storage Lens to watch noncurrent bytes per bucket, and use one standard lifecycle template for versioned buckets.

</details>

### Q87. The bill shows far more S3 storage than `aws s3 ls --summarize` reports. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Incomplete multipart uploads (the parts are stored and billed, but `ls` does not show them), noncurrent versions, or storage-class overhead (minimum object size and duration charges for IA and Glacier).

**Diagnose →**

```bash
aws s3api list-multipart-uploads --bucket data --query 'Uploads[].[Key,Initiated]' --output table
aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name BucketSizeBytes \
  --dimensions Name=BucketName,Value=data Name=StorageType,Value=StandardStorage \
  --statistics Average --period 86400 --start-time $(date -u -d '-3 day' +%FT%TZ) --end-time $(date -u +%FT%TZ)
```

**Fix →**

1. Abort stale uploads: `aws s3api abort-multipart-upload --bucket data --key <key> --upload-id <id>`
2. Add a lifecycle rule with `AbortIncompleteMultipartUpload` `DaysAfterInitiation: 7`

**Prevent →** Put that lifecycle rule on every bucket by default, and use Storage Lens for "incomplete MPU bytes".

</details>

### Q88. `s3api put-object` fails: `EntityTooLarge: Your proposed upload exceeds the maximum allowed size`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One PUT can be at most 5 GB. `aws s3api put-object` does a single PUT, while `aws s3 cp` switches to multipart automatically. Objects larger than that can only be uploaded with multipart upload.

**Diagnose →**

```bash
ls -lh backup.tar
aws configure get default.s3.multipart_threshold
```

**Fix →**

1. Use `aws s3 cp backup.tar s3://bucket/` (multipart, parallel parts)
2. Tune it for speed: `aws configure set default.s3.multipart_chunksize 64MB` and `max_concurrent_requests 20`

**Prevent →** Use the high-level transfer managers (CLI `s3 cp`, SDK TransferManager) for anything larger than about 100 MB.

</details>

### Q89. Someone added a bucket policy with `"Effect":"Deny"` unless `aws:SourceVpce` matches. Now even admins get `AccessDenied`, including on `PutBucketPolicy`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An explicit deny in the bucket policy beats every allow, including admin IAM policies. Calls from outside the VPC endpoint (console, laptop CLI) are denied, and that includes changing the policy.

**Diagnose →**

```bash
aws s3api get-bucket-policy --bucket locked --output text   # likely denied too
# CloudTrail: GetBucketPolicy / PutBucketPolicy errorCode AccessDenied
```

**Fix →**

1. Call from a principal that matches the condition, for example an instance in the VPC that uses that endpoint, and run `aws s3api delete-bucket-policy --bucket locked`
2. If nothing can match, sign in as the account **root user**, which can always delete a bucket policy that locks it out (with Organizations, the management account can use centralized root access for member accounts)

**Prevent →** Always exempt a break-glass role in deny conditions (`aws:PrincipalArn` with `StringNotLike`), and test policies with Access Analyzer policy validation before applying them.

</details>

### Q90. You attached a new EBS volume, but `df -h` does not show it and `/dev/sdf` does not exist. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Attaching a volume does not format or mount it. On Nitro instances, the device shows up as `/dev/nvme1n1`, not `/dev/sdf`.

**Diagnose →**

```bash
aws ec2 describe-volumes --volume-ids vol-0abc --query 'Volumes[].Attachments'
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,SERIAL     # SERIAL shows vol0abc...
sudo file -s /dev/nvme1n1                       # "data" = no filesystem
```

**Fix →**

```bash
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir -p /data && sudo mount /dev/nvme1n1 /data
echo "UUID=$(sudo blkid -s UUID -o value /dev/nvme1n1) /data xfs defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

**Prevent →** Use UUIDs and `nofail` in fstab, and do the formatting in user data or configuration management.

</details>

### Q91. You grew an EBS volume from 20 to 50 GiB, but the server still says `No space left on device`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You resized the volume, but not the partition and filesystem inside it.

**Diagnose →**

```bash
aws ec2 describe-volumes-modifications --volume-ids vol-0abc --query 'VolumesModifications[].ModificationState'
lsblk                 # disk 50G, partition still 20G
df -hT /
```

**Fix →**

```bash
sudo growpart /dev/nvme0n1 1
sudo xfs_growfs -d /          # XFS (Amazon Linux)
# or
sudo resize2fs /dev/nvme0n1p1 # ext4 (Ubuntu)
```

If `df` shows the disk is not full, check inodes: `df -i`.

**Prevent →** Alarm on disk usage from the CloudWatch agent (`disk_used_percent`), and send logs off-box so they do not fill the disk.

</details>

### Q92. `modify-volume` fails: `You've reached the maximum modification rate per volume limit. Wait at least 6 hours between modifications per EBS volume.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** EBS allows one modification per volume every 6 hours, and the previous change must have reached `completed` or `optimizing`.

**Diagnose →**

```bash
aws ec2 describe-volumes-modifications --volume-ids vol-0abc \
  --query 'VolumesModifications[].[ModificationState,StartTime,EndTime,TargetSize,TargetIops]'
```

**Fix →**

1. If you need space now, attach a second volume and extend with LVM, or move data out
2. Otherwise wait, and change size, IOPS and throughput **in one call**: `aws ec2 modify-volume --volume-id vol-0abc --size 200 --iops 6000 --throughput 500`

**Prevent →** Right-size with headroom, and batch all attribute changes into one modification.

</details>

### Q93. `InvalidVolume.ZoneMismatch: The volume 'vol-0abc' is not in the same availability zone as instance 'i-0def'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An EBS volume exists in one AZ and can only attach to instances in that AZ.

**Diagnose →**

```bash
aws ec2 describe-volumes --volume-ids vol-0abc --query 'Volumes[].AvailabilityZone'
aws ec2 describe-instances --instance-ids i-0def --query 'Reservations[].Instances[].Placement.AvailabilityZone'
```

**Fix →**

```bash
aws ec2 create-snapshot --volume-id vol-0abc --description move-az
aws ec2 create-volume --snapshot-id snap-0123 --availability-zone us-east-1b --volume-type gp3
```

Then attach the new volume.

**Prevent →** Keep shared or portable data on EFS or S3, and pin stateful instances (and their ASG subnets) to the AZ of their volumes.

</details>

### Q94. A database on gp2 is fast in the morning, then latency jumps and `VolumeQueueLength` climbs. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** gp2 gets baseline IOPS at 3 per GiB (minimum 100) and bursts to 3,000 IOPS from a credit bucket. Once `BurstBalance` reaches 0, the volume falls back to baseline. Another possibility is the instance's EBS bandwidth limit (`EBSIOBalance%` / `EBSByteBalance%` on smaller instances).

**Diagnose →**

```bash
for m in BurstBalance VolumeQueueLength VolumeReadOps VolumeWriteOps; do
aws cloudwatch get-metric-statistics --namespace AWS/EBS --metric-name $m \
  --dimensions Name=VolumeId,Value=vol-0abc --statistics Average --period 300 \
  --start-time $(date -u -d '-6 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ) --output text | tail -3; done
```

**Fix →**

1. Migrate online to gp3, which gives 3,000 IOPS and 125 MiB/s baseline independent of size: `aws ec2 modify-volume --volume-id vol-0abc --volume-type gp3 --iops 6000 --throughput 250`
2. If the instance is the bottleneck, move to an instance type with more EBS bandwidth

**Prevent →** Use gp3 as the default volume type and alarm on queue length and latency (`VolumeAverageReadLatency`).

</details>

### Q95. A volume restored from a snapshot is very slow for the first hour of a DR test. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Snapshot restores are lazy-loaded: each block is fetched from S3 the first time it is read, so first reads have high latency.

**Diagnose →**

```bash
aws ec2 describe-fast-snapshot-restores --filters Name=snapshot-id,Values=snap-0abc
# on the instance: high await on first reads
iostat -x 5
```

**Fix →**

1. Pre-warm by reading every block: `sudo fio --filename=/dev/nvme1n1 --rw=read --bs=1M --iodepth=32 --ioengine=libaio --direct=1 --name=init`
2. For critical paths, turn on Fast Snapshot Restore (paid per AZ-hour): `aws ec2 enable-fast-snapshot-restores --availability-zones us-east-1a --source-snapshot-ids snap-0abc`
3. Or use a provisioned initialization rate where supported

**Prevent →** Count hydration time in your RTO, and test DR with real reads, not just "the instance is running".

</details>

### Q96. Sharing an EBS snapshot with the DR account fails because it is encrypted with the default `aws/ebs` key. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You cannot share snapshots encrypted with the AWS managed key, because its key policy cannot be changed to grant other accounts.

**Diagnose →**

```bash
aws ec2 describe-snapshots --snapshot-ids snap-0abc --query 'Snapshots[].[Encrypted,KmsKeyId]'
aws kms describe-key --key-id <key-id> --query 'KeyMetadata.KeyManager'   # AWS = managed key
```

**Fix →**

1. Copy the snapshot and re-encrypt it with a customer managed key: `aws ec2 copy-snapshot --source-region us-east-1 --source-snapshot-id snap-0abc --encrypted --kms-key-id <cmk-arn>`
2. Allow the DR account in the CMK key policy, then share: `aws ec2 modify-snapshot-attribute --snapshot-id snap-0new --attribute createVolumePermission --operation-type add --user-ids 222222222222`
3. In the DR account, copy it again with its own key

**Prevent →** Set EBS default encryption to a CMK (`aws ec2 modify-ebs-default-kms-key-id`), or use AWS Backup cross-account copy.

</details>

### Q97. `mount -t efs` hangs, then fails with `mount.nfs4: Connection timed out`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The mount target's SG does not allow NFS (TCP 2049) from the client, there is no mount target in the client's AZ, or DNS resolution of the file system name is off in the VPC.

**Diagnose →**

```bash
aws efs describe-mount-targets --file-system-id fs-0abc --query 'MountTargets[].[AvailabilityZoneName,IpAddress,LifeCycleState]'
aws efs describe-mount-target-security-groups --mount-target-id fsmt-0abc
nc -zv fs-0abc.efs.us-east-1.amazonaws.com 2049
```

**Fix →**

1. Allow TCP 2049 from the client SG on the mount target SG
2. Create a mount target in every AZ where clients run
3. Turn on `enableDnsSupport` and `enableDnsHostnames` on the VPC

**Prevent →** Create mount targets for every AZ in IaC, and use `amazon-efs-utils` with `_netdev,tls` in fstab.

</details>

### Q98. `mount.nfs4: access denied by server while mounting fs-0abc.efs...`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An EFS file system policy requires IAM authorization or TLS, and the mount did not use `-o tls,iam`. Or the client role has no `elasticfilesystem:ClientMount`/`ClientWrite`. Or an access point's root directory or POSIX user does not match what is expected.

**Diagnose →**

```bash
aws efs describe-file-system-policy --file-system-id fs-0abc
aws efs describe-access-points --file-system-id fs-0abc
```

**Fix →**

1. Mount with efs-utils: `sudo mount -t efs -o tls,iam,accesspoint=fsap-0abc fs-0abc:/ /mnt/efs`
2. Grant the instance or task role `elasticfilesystem:ClientMount` and `ClientWrite` (plus `ClientRootAccess` if needed)

**Prevent →** Standardize on access points per app, with IAM auth and TLS required by policy.

</details>

### Q99. An app on EFS slows to a crawl under load; `PercentIOLimit` sits at 100% or throughput flat-lines. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** In Bursting mode, throughput depends on stored size, and `BurstCreditBalance` has run out. `PercentIOLimit` at 100% means the file system has hit the I/O operation limit for General Purpose performance mode. Lots of small-file metadata operations also have high per-operation latency on NFS.

**Diagnose →**

```bash
aws efs describe-file-systems --file-system-id fs-0abc --query 'FileSystems[].[ThroughputMode,PerformanceMode,SizeInBytes.Value]'
# CloudWatch AWS/EFS: BurstCreditBalance, PermittedThroughput, MeteredIOBytes, PercentIOLimit
```

**Fix →**

1. Switch to Elastic throughput: `aws efs update-file-system --file-system-id fs-0abc --throughput-mode elastic`
2. Spread I/O across more clients and threads, and avoid hot `ls` loops over huge directories
3. Cache read-heavy content locally

**Prevent →** Use Elastic throughput by default for spiky workloads, and keep databases off EFS.

</details>

### Q100. `put-bucket-notification-configuration` fails: `Unable to validate the following destination configurations`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** S3 checks that it may publish to the destination. The Lambda has no resource policy allowing `s3.amazonaws.com`, or the SNS topic or SQS queue policy does not allow S3, or the destination's KMS key does not let S3 use it.

**Diagnose →**

```bash
aws lambda get-policy --function-name thumbnailer
aws sqs get-queue-attributes --queue-url <url> --attribute-names Policy KmsMasterKeyId
```

**Fix →**

```bash
aws lambda add-permission --function-name thumbnailer --statement-id s3invoke \
  --action lambda:InvokeFunction --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::uploads --source-account 111111111111
```

For SQS or SNS, add a policy statement for `s3.amazonaws.com` with an `aws:SourceArn` condition. With SSE, the key policy must allow S3 `kms:GenerateDataKey` and `kms:Decrypt`.

**Prevent →** Create the permission and the notification together in IaC (with `depends_on`), or route events through EventBridge.

</details>

---

## RDS, Aurora and DynamoDB

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. The app cannot connect to RDS: `psql: error: connection to server at "mydb.abc.us-east-1.rds.amazonaws.com" (10.0.3.12), port 5432 failed: Connection timed out`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A timeout means the packets are dropped. The DB SG does not allow 5432 from the app SG, a NACL blocks it, or the client is outside the VPC (for example, a laptop) while the DB is not publicly accessible (which is how it should be).

**Diagnose →**

```bash
aws rds describe-db-instances --db-instance-identifier mydb \
  --query 'DBInstances[].[Endpoint.Address,Endpoint.Port,PubliclyAccessible,VpcSecurityGroups,DBSubnetGroup.Subnets[].SubnetIdentifier]'
nc -zv mydb.abc.us-east-1.rds.amazonaws.com 5432
aws ec2 describe-security-groups --group-ids <db-sg> --query 'SecurityGroups[].IpPermissions'
```

**Fix →**

1. Allow 5432 on the DB SG from the app SG
2. From a laptop, use SSM port forwarding through a bastion: `aws ssm start-session --target i-0abc --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters host=mydb...,portNumber=5432,localPortNumber=5432`

**Prevent →** Keep databases in private subnets and grant access by SG reference only.

</details>

### Q102. `FATAL: password authentication failed for user "postgres"` and nobody knows the master password. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The password is wrong or was rotated (Secrets Manager managed password), or the username is wrong. The master username is set at creation and is not always `postgres`.

**Diagnose →**

```bash
aws rds describe-db-instances --db-instance-identifier mydb \
  --query 'DBInstances[].[MasterUsername,MasterUserSecret.SecretArn]'
aws secretsmanager get-secret-value --secret-id <secret-arn> --query SecretString --output text
```

**Fix →**

1. If RDS manages the secret, read the current value from Secrets Manager
2. Otherwise reset the password: `aws rds modify-db-instance --db-instance-identifier mydb --master-user-password '<new>' --apply-immediately`
3. Update the apps, and restart connection pools

**Prevent →** Use `--manage-master-user-password` so RDS stores and rotates the password in Secrets Manager, and give apps their own least-privilege DB users.

</details>

### Q103. `FATAL: no pg_hba.conf entry for host "10.0.1.25", user "app", database "orders", no encryption`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The parameter group sets `rds.force_ssl=1` (the default for RDS PostgreSQL 15+), and the client connects without TLS. You cannot edit `pg_hba.conf` on RDS; it is controlled through parameters.

**Diagnose →**

```bash
aws rds describe-db-parameters --db-parameter-group-name pg16-app \
  --query "Parameters[?ParameterName=='rds.force_ssl']"
```

**Fix →**

1. Connect with TLS: `psql "host=... dbname=orders user=app sslmode=verify-full sslrootcert=global-bundle.pem"`
2. Download the CA bundle: `curl -O https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem`
3. Set `sslmode=require` or `verify-full` in the app's JDBC or driver URL

**Prevent →** Require TLS everywhere, and ship the RDS CA bundle in your base images.

</details>

### Q104. App logs show `FATAL: remaining connection slots are reserved for non-replication superuser connections` (or MySQL `ERROR 1040 (HY000): Too many connections`). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You have hit `max_connections`. Usually too many app replicas each open a large pool, or Lambda functions open a new connection per invocation, or connections leak.

**Diagnose →**

```bash
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=mydb --statistics Maximum --period 60 \
  --start-time $(date -u -d '-2 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ)
psql -c "select usename, application_name, state, count(*) from pg_stat_activity group by 1,2,3 order by 4 desc;"
```

**Fix →**

1. Shrink the pool per pod: total = replicas × pool size, which must stay under max_connections
2. Put RDS Proxy (or PgBouncer) in front for Lambda and bursty clients
3. Kill idle connections and set `idle_in_transaction_session_timeout`

**Prevent →** Budget connections per service, alarm at 80% of max, and use RDS Proxy for serverless callers.

</details>

### Q105. RDS status shows `storage-full` and writes fail. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `FreeStorageSpace` reached zero. Common reasons: growing tables, unvacuumed bloat, a large temp sort, binlogs or WAL kept by a stalled replication slot, or verbose logs.

**Diagnose →**

```bash
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=mydb --statistics Minimum --period 300 \
  --start-time $(date -u -d '-1 day' +%FT%TZ) --end-time $(date -u +%FT%TZ)
psql -c "select slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) from pg_replication_slots;"
```

**Fix →**

1. Add storage now: `aws rds modify-db-instance --db-instance-identifier mydb --allocated-storage 500 --apply-immediately`
2. Drop stale replication slots, and vacuum or clean up data
3. Turn on storage autoscaling: `--max-allocated-storage 1000`

**Prevent →** Alarm on `FreeStorageSpace` below 20%, turn on autoscaling, and alarm on `OldestReplicationSlotLag` / `TransactionLogsDiskUsage`.

</details>

### Q106. A read replica's `ReplicaLag` keeps growing and reports show stale data. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The replica is smaller than the primary, or IOPS-bound. Heavy write bursts or long-running queries on the replica cause conflicts (PostgreSQL `max_standby_streaming_delay`). MySQL replication may apply single-threaded.

**Diagnose →**

```bash
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=mydb-replica --statistics Maximum --period 60 \
  --start-time $(date -u -d '-3 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ)
aws rds describe-db-instances --db-instance-identifier mydb-replica --query 'DBInstances[].[DBInstanceClass,StorageType,Iops]'
```

**Fix →**

1. Size the replica the same as the primary, with the same storage performance
2. For MySQL, turn on parallel apply (`replica_parallel_workers`)
3. Move heavy analytics to a dedicated replica or to Aurora, where replicas share storage and lag is usually well under a second

**Prevent →** Alarm on lag, and route reads that must not be stale to the primary.

</details>

### Q107. After a Multi-AZ failover, some app servers keep failing with connection errors for 10+ minutes while others recover in a minute. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A failover flips the endpoint's DNS CNAME to the standby. Clients that cache DNS for a long time (JVM `networkaddress.cache.ttl` set to forever, or a misconfigured local resolver) or keep stale pooled connections keep hitting the old IP.

**Diagnose →**

```bash
aws rds describe-events --source-identifier mydb --source-type db-instance --duration 120
dig +short mydb.abc.us-east-1.rds.amazonaws.com
# on the bad host
getent hosts mydb.abc.us-east-1.rds.amazonaws.com
```

**Fix →**

1. Set the JVM DNS TTL to a low value (`networkaddress.cache.ttl=5`)
2. Turn on pool connection validation and max lifetime (for example, HikariCP `maxLifetime`)
3. Restart the affected app instances to recover now

**Prevent →** Use RDS Proxy (it keeps client connections during failover) or the AWS Advanced JDBC Wrapper, and test failover regularly with `aws rds reboot-db-instance --force-failover`.

</details>

### Q108. Restoring a snapshot fails: `InvalidParameterCombination: Cannot find version 11.22 for postgres`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The engine version has reached end of standard support or is not offered in this Region any more, so you cannot create new instances on it without Extended Support or an upgrade.

**Diagnose →**

```bash
aws rds describe-db-engine-versions --engine postgres --query 'DBEngineVersions[].EngineVersion' --output text
aws rds describe-db-snapshots --db-snapshot-identifier snap-1 --query 'DBSnapshots[].EngineVersion'
```

**Fix →**

1. Restore with a supported version, specifying `--engine-version` if the restore path allows it
2. Or upgrade the snapshot first: `aws rds modify-db-snapshot --db-snapshot-identifier snap-1 --engine-version 12.22` (RDS PostgreSQL supports upgrading DB snapshots), then upgrade the restored instance further

**Prevent →** Track engine end-of-life dates, upgrade on a schedule, and run restore tests regularly.

</details>

### Q109. You changed `work_mem` in the parameter group, but `SHOW work_mem;` still shows the old value. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance uses a different parameter group, the parameter is static (it needs a reboot, status `pending-reboot`), or the session or role overrides it.

**Diagnose →**

```bash
aws rds describe-db-instances --db-instance-identifier mydb \
  --query 'DBInstances[].DBParameterGroups'
aws rds describe-db-parameters --db-parameter-group-name pg16-app \
  --query "Parameters[?ParameterName=='work_mem'].[ParameterValue,ApplyType,IsModifiable]"
psql -c "select name, setting, source from pg_settings where name='work_mem';"
```

**Fix →**

1. Attach the right custom group. The default group cannot be modified
2. For static parameters, reboot during a window
3. Remove role-level overrides (`ALTER ROLE app RESET work_mem`)

**Prevent →** Use one custom parameter group per engine and environment, managed in IaC, and check `pending-reboot` after every change.

</details>

### Q110. RDS IAM authentication fails: `FATAL: PAM authentication failed for user "app_iam"`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The token was generated for a different host, port, user or Region, or it is older than 15 minutes. The DB user is missing the `rds_iam` role (PostgreSQL) or `AWSAuthenticationPlugin` (MySQL). The IAM policy has no `rds-db:connect` on the right `dbuser` ARN (it uses the DbiResourceId, not the instance name). Or TLS is not used.

**Diagnose →**

```bash
aws rds describe-db-instances --db-instance-identifier mydb --query 'DBInstances[].[DbiResourceId,IAMDatabaseAuthenticationEnabled]'
TOKEN=$(aws rds generate-db-auth-token --hostname mydb.abc.us-east-1.rds.amazonaws.com --port 5432 --username app_iam --region us-east-1)
psql "host=mydb.abc.us-east-1.rds.amazonaws.com user=app_iam dbname=orders sslmode=require password=$TOKEN"
```

**Fix →**

1. Run `GRANT rds_iam TO app_iam;`
2. Policy resource: `arn:aws:rds-db:us-east-1:111111111111:dbuser:db-ABCDEFGH123/app_iam`
3. Generate the token right before you connect, and always use TLS

**Prevent →** Wrap token generation in the connection factory, and keep the IAM auth rate limits in mind (use RDS Proxy at high connection rates).

</details>

### Q111. `InvalidParameterCombination: Cannot delete protected DB Instance, please disable deletion protection and try again.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Deletion protection is on, which is the intended safeguard.

**Diagnose →**

```bash
aws rds describe-db-instances --db-instance-identifier old-db --query 'DBInstances[].DeletionProtection'
```

**Fix →**

```bash
aws rds modify-db-instance --db-instance-identifier old-db --no-deletion-protection --apply-immediately
aws rds delete-db-instance --db-instance-identifier old-db --final-db-snapshot-identifier old-db-final
```

**Prevent →** Keep deletion protection on in production. Require a final snapshot, and do the disable step through a change ticket, not by hand.

</details>

### Q112. Writes fail on Aurora MySQL with `ERROR 1290 (HY000): The MySQL server is running with the --read-only option so it cannot execute this statement`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app is connected to a reader: through the reader endpoint, an instance endpoint that became a reader after failover, or a pooled connection that lived through a failover.

**Diagnose →**

```bash
aws rds describe-db-clusters --db-cluster-identifier orders \
  --query 'DBClusters[].[Endpoint,ReaderEndpoint,DBClusterMembers[].[DBInstanceIdentifier,IsClusterWriter]]'
mysql -h <host> -e "select @@aurora_server_id, @@innodb_read_only;"
```

**Fix →**

1. Point write traffic at the **cluster (writer) endpoint**, never at an instance endpoint
2. Recycle pools after failover. The AWS Advanced JDBC Wrapper handles this
3. Retry the write after a reconnect

**Prevent →** Use separate writer and reader data sources in the app, and run failover drills (`aws rds failover-db-cluster`).

</details>

### Q113. RDS `CPUUtilization` is pinned at 100% and the API is timing out. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One or a few bad queries: a missing index after a deploy, a plan regression from stale statistics, a runaway report, or lock contention that shows up as CPU spin.

**Diagnose →**

```bash
aws pi get-resource-metrics --service-type RDS --identifier db-ABCDEFGH123 \
  --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.sql_tokenized","Limit":5}}]' \
  --start-time $(date -u -d '-1 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ) --period-in-seconds 300
psql -c "select pid, now()-query_start dur, state, left(query,80) from pg_stat_activity where state<>'idle' order by dur desc limit 10;"
```

**Fix →**

1. Kill the runaway query: `select pg_cancel_backend(<pid>);`
2. Add the missing index (`CREATE INDEX CONCURRENTLY`), and run `ANALYZE` on the affected tables
3. Scale up only as a stopgap

**Prevent →** Keep Performance Insights and `pg_stat_statements` on, review query plans in CI for hot paths, and alarm on DB load above the vCPU count.

</details>

### Q114. An RDS major version upgrade fails, and the event says `Database instance is in a state that cannot be upgraded: PreUpgrade checks failed`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pre-checks found blockers: unsupported extensions or extension versions (PostGIS, pg_repack), `reg*` data types, open prepared transactions, logical replication slots, or for MySQL, objects incompatible with 8.0/8.4 (reserved words, old temporal types).

**Diagnose →**

```bash
aws rds describe-events --source-identifier mydb --source-type db-instance --duration 60
aws rds describe-db-log-files --db-instance-identifier mydb --filename-contains upgrade
aws rds download-db-log-file-portion --db-instance-identifier mydb --log-file-name <precheck-log> --output text
aws rds describe-db-engine-versions --engine postgres --engine-version 15.8 --query 'DBEngineVersions[].ValidUpgradeTarget[].EngineVersion'
```

**Fix →**

1. Upgrade the extensions (`ALTER EXTENSION postgis UPDATE;`), drop replication slots, and commit or roll back prepared transactions
2. Retry with a matching custom parameter group for the target version
3. For minimal downtime, use RDS Blue/Green Deployments

**Prevent →** Rehearse upgrades on a restored snapshot, and use Blue/Green for major versions.

</details>

### Q115. Creating a DB fails: `InsufficientDBInstanceCapacity: Cannot create a db.r6g.4xlarge instance in us-east-1d`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance class has no capacity in that AZ right now, or the class is not offered there at all.

**Diagnose →**

```bash
aws rds describe-orderable-db-instance-options --engine postgres --db-instance-class db.r6g.4xlarge \
  --query 'OrderableDBInstanceOptions[].AvailabilityZones[].Name' --output text
```

**Fix →**

1. Retry in another AZ, or make sure the subnet group has subnets in AZs that offer the class
2. Pick a sibling class (`db.r7g`, `db.r6i`)

**Prevent →** Use DB subnet groups that cover at least three AZs, and avoid pinning `--availability-zone` unless you need to.

</details>

### Q116. `ProvisionedThroughputExceededException: The level of configured provisioned throughput for the table was exceeded.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The table is in provisioned mode and traffic is above its RCU/WCU. Auto scaling reacts only after a few minutes, so sudden spikes throttle first. Burst capacity covers only short spikes.

**Diagnose →**

```bash
aws dynamodb describe-table --table-name orders --query 'Table.[BillingModeSummary,ProvisionedThroughput]'
aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents \
  --dimensions Name=TableName,Value=orders --statistics Sum --period 60 \
  --start-time $(date -u -d '-1 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ)
aws application-autoscaling describe-scalable-targets --service-namespace dynamodb
```

**Fix →**

1. For spiky or unknown traffic, switch to on-demand: `aws dynamodb update-table --table-name orders --billing-mode PAY_PER_REQUEST`
2. Or raise the capacity and the auto scaling minimum before known peaks
3. Make sure the SDK retries with backoff

**Prevent →** Use on-demand by default, provisioned with auto scaling for steady, predictable load, and alarm on throttle events.

</details>

### Q117. One DynamoDB table throttles even though consumed capacity is far below provisioned (or it is on-demand). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A hot partition. One partition key, such as `status=PENDING`, today's date or one big tenant, takes most of the traffic. Each partition is limited to about 3,000 RCU and 1,000 WCU per second, no matter how much the table has.

**Diagnose →**

```bash
aws dynamodb describe-contributor-insights --table-name orders
aws dynamodb update-contributor-insights --table-name orders --contributor-insights-action ENABLE
# then view top partition keys in CloudWatch Contributor Insights
```

**Fix →**

1. Redesign the key for higher cardinality, for example write-sharding the hot key: `PENDING#<0-9>`
2. Cache hot reads with DAX or ElastiCache
3. Spread time-series writes across shard suffixes

**Prevent →** Model access patterns before you build the table, and keep Contributor Insights on for important tables.

</details>

### Q118. `ValidationException: One or more parameter values were invalid: Missing the key orderId in the item`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `PutItem` request does not include the partition key (or sort key) attribute, or uses a different name or case (`orderID` vs `orderId`), or a different type (S vs N).

**Diagnose →**

```bash
aws dynamodb describe-table --table-name orders --query 'Table.[KeySchema,AttributeDefinitions]'
```

**Fix →** Send every key attribute with the exact name and type:

```bash
aws dynamodb put-item --table-name orders \
  --item '{"orderId":{"S":"o-1001"},"createdAt":{"N":"1731139200"},"total":{"N":"42.5"}}'
```

**Prevent →** Use a typed data layer (for example, the DynamoDB Document Client or ElectroDB) with a schema, and add contract tests.

</details>

### Q119. Updates fail with `ConditionalCheckFailedException: The conditional request failed`. Is it a bug? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Usually it is working as designed: the `ConditionExpression` (for example, an optimistic lock on `version`, or `attribute_not_exists(pk)`) was false because another writer changed the item first, or the item already exists.

**Diagnose →**

```bash
aws dynamodb get-item --table-name orders --key '{"orderId":{"S":"o-1001"}}' --consistent-read
aws dynamodb update-item --table-name orders --key '{"orderId":{"S":"o-1001"}}' \
  --update-expression 'SET #s=:s, version=version+:one' --condition-expression 'version=:v' \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":s":{"S":"PAID"},":v":{"N":"3"},":one":{"N":"1"}}' \
  --return-values-on-condition-check-failure ALL_OLD
```

**Fix →**

1. Treat it as a conflict: re-read, merge, and retry with the new version
2. Use `ReturnValuesOnConditionCheckFailure=ALL_OLD` to see the current item without a second read

**Prevent →** Handle the exception explicitly in code, and count it as a business metric, not an error.

</details>

### Q120. `ValidationException: Item size has exceeded the maximum allowed size`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A DynamoDB item, including attribute names, is limited to 400 KB. A common cause is lists that keep growing (events, comments) or embedded blobs.

**Diagnose →**

```bash
aws dynamodb get-item --table-name sessions --key '{"id":{"S":"u-1"}}' --output json | wc -c
```

**Fix →**

1. Store large payloads in S3 and keep the S3 key in the item
2. Split unbounded lists into separate items under the same partition key (for example, `pk=USER#1, sk=EVENT#<ts>`)
3. Compress large attributes (gzip into a Binary attribute)

**Prevent →** Never model unbounded collections inside one item, and keep attribute names short in hot tables.

</details>

### Q121. `TransactionCanceledException: Transaction cancelled, please refer cancellation reasons for specific reasons [None, ConditionalCheckFailed]`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One item in `TransactWriteItems` failed, so the whole transaction rolled back. The reasons list matches the request order, so the second action's condition failed here. `TransactionConflict` in the list means a concurrent transaction touched the same item.

**Diagnose →**

```bash
aws dynamodb transact-write-items --transact-items file://tx.json 2>&1 | tail
# inspect CancellationReasons[].Code / Item in the SDK exception
```

**Fix →**

1. Map each reason back to its action, and handle `ConditionalCheckFailed` as a business conflict
2. Retry `TransactionConflict` with backoff
3. Use a `ClientRequestToken` for idempotent retries

**Prevent →** Keep transactions small (up to 100 items), avoid hot items inside transactions, and log cancellation reasons in a structured form.

</details>

### Q122. A user updates a profile and immediately reloads, but the GSI query still returns the old value. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Global secondary indexes are updated asynchronously and only support eventually consistent reads. `ConsistentRead=true` is rejected on a GSI.

**Diagnose →**

```bash
aws dynamodb query --table-name users --index-name email-index \
  --key-condition-expression 'email=:e' --expression-attribute-values '{":e":{"S":"a@b.com"}}'
aws dynamodb get-item --table-name users --key '{"userId":{"S":"u-1"}}' --consistent-read
```

**Fix →**

1. For read-after-write views, read from the base table by primary key with `--consistent-read`
2. Or use an LSI (it supports strong consistency but must be created with the table)
3. Or return the updated item to the client from the write response

**Prevent →** Decide consistency needs per access pattern during design, and watch `OnlineIndexPercentageProgress` and GSI throttles.

</details>

### Q123. A DynamoDB `Query` with a filter returns only 12 items, but you know there are hundreds. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Each Query or Scan page reads at most 1 MB **before** the `FilterExpression` is applied. The response has a `LastEvaluatedKey`, and the code does not paginate.

**Diagnose →**

```bash
aws dynamodb query --table-name orders --key-condition-expression 'customerId=:c' \
  --filter-expression '#s=:s' --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":c":{"S":"c-9"},":s":{"S":"OPEN"}}' --no-paginate \
  --query '[Count,ScannedCount,LastEvaluatedKey]'
```

**Fix →**

1. Loop until `LastEvaluatedKey` is empty (the CLI paginates by default; SDKs have paginators)
2. If the filter discards most items, add a GSI keyed on that attribute so you query instead of filter

**Prevent →** Always use paginators, and compare `ScannedCount` with `Count` in reviews; a big gap means the key design is wrong.

</details>

### Q124. Writes to the base table throttle, but the table itself has plenty of capacity. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A GSI is under-provisioned or hot. GSI writes happen asynchronously, but when a GSI cannot keep up, DynamoDB throttles writes on the **base table** (back-pressure). This often follows adding a GSI with a low-cardinality key.

**Diagnose →**

```bash
aws dynamodb describe-table --table-name orders \
  --query 'Table.GlobalSecondaryIndexes[].[IndexName,ProvisionedThroughput,IndexStatus]'
# CloudWatch WriteThrottleEvents with dimensions TableName + GlobalSecondaryIndexName
```

**Fix →**

1. Raise the GSI's WCU, or switch the table to on-demand
2. Redesign the GSI partition key so it has high cardinality
3. Project only the attributes you need, to lower write size

**Prevent →** Give each GSI the same auto scaling as the base table, and alarm on GSI throttles separately.

</details>

### Q125. TTL is enabled, but expired items are still in the table days later. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The TTL attribute is not a **Number** in epoch **seconds** (a string or milliseconds is ignored), the attribute name does not match, or deletion is simply lagging. TTL deletes in the background, usually within a few days.

**Diagnose →**

```bash
aws dynamodb describe-time-to-live --table-name sessions
aws dynamodb get-item --table-name sessions --key '{"id":{"S":"s-1"}}' --query 'Item.expiresAt'
date +%s
```

**Fix →**

1. Write `expiresAt` as a Number with `$(date -d '+1 day' +%s)`, not milliseconds
2. Filter out expired items in reads (`expiresAt > :now`) until they are deleted

**Prevent →** Use a unit test for TTL values, and use the `TimeToLiveDeletedItemCount` metric to confirm deletes are happening.

</details>

---

## ECS, EKS, ECR and Lambda

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. `docker push 111111111111.dkr.ecr.us-east-1.amazonaws.com/api:1.4` fails with `no basic auth credentials` (or `denied: Your authorization token has expired. Reauthenticate and try again.`). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker never logged in to that registry and Region, or the ECR token (valid for 12 hours) expired. Another cause: the repository does not exist (`name unknown: The repository with name 'api' does not exist in the registry`).

**Diagnose →**

```bash
aws sts get-caller-identity
aws ecr describe-repositories --repository-names api --region us-east-1
cat ~/.docker/config.json | grep -A2 dkr.ecr
```

**Fix →**

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 111111111111.dkr.ecr.us-east-1.amazonaws.com
aws ecr create-repository --repository-name api --image-scanning-configuration scanOnPush=true
```

**Prevent →** In CI, log in at the start of every job (for example, `aws-actions/amazon-ecr-login`), or use `amazon-ecr-credential-helper`.

</details>

### Q127. Fargate tasks stop with `CannotPullContainerError: pull image manifest has been retried 5 time(s): failed to resolve ref ... dial tcp 52.x.x.x:443: i/o timeout`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The task runs in a private subnet with no route to ECR: no NAT gateway, and no VPC endpoints for `ecr.api`, `ecr.dkr` and S3 (gateway endpoint, because image layers come from S3). Or it is in a public subnet without `assignPublicIp=ENABLED`.

**Diagnose →**

```bash
aws ecs describe-tasks --cluster prod --tasks <task-arn> --query 'tasks[].[stoppedReason,containers[].reason]'
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-0abc --query 'RouteTables[].Routes'
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-0abc --query 'VpcEndpoints[].ServiceName'
```

**Fix →**

1. Add a NAT route, or add the endpoints: `com.amazonaws.us-east-1.ecr.api`, `ecr.dkr`, `logs` (interface) and `s3` (gateway)
2. The endpoint SG must allow 443 from the task SG

**Prevent →** Ship a standard "private VPC endpoints" module, and test task launches in every new subnet.

</details>

### Q128. `ResourceInitializationError: unable to pull secrets or registry auth: ... AccessDeniedException: User: arn:aws:sts::111111111111:assumed-role/ecsTaskExecutionRole/... is not authorized to perform: secretsmanager:GetSecretValue`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The task definition refers to a secret in `secrets`, but the **execution role** (not the task role) cannot read it, or cannot decrypt the KMS key behind it. Network reachability to Secrets Manager or SSM can also cause this error.

**Diagnose →**

```bash
aws ecs describe-task-definition --task-definition api:42 \
  --query 'taskDefinition.[executionRoleArn,containerDefinitions[].secrets]'
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::111111111111:role/ecsTaskExecutionRole \
  --action-names secretsmanager:GetSecretValue --resource-arns <secret-arn>
```

**Fix →** Add `secretsmanager:GetSecretValue` (or `ssm:GetParameters`) on the specific secret ARNs, plus `kms:Decrypt` on the CMK, to the execution role. If the tasks are in private subnets, add the `secretsmanager` endpoint.

**Prevent →** Keep one execution role per service, scoped to its own secret paths, and remember: the execution role is for the agent, the task role is for your code.

</details>

### Q129. A task stops with `Essential container in task exited`, exit code 137, and reason `OutOfMemoryError: Container killed due to memory usage`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The container went over its hard `memory` limit, so the kernel OOM killer sent SIGKILL (137 = 128 + 9). A JVM heap that is not sized to the container limit, or a memory leak, are the usual reasons.

**Diagnose →**

```bash
aws ecs describe-tasks --cluster prod --tasks <task-arn> \
  --query 'tasks[].containers[].[name,exitCode,reason]'
# Container Insights: MemoryUtilized vs MemoryReserved per task
aws logs tail /ecs/api --since 30m
```

**Fix →**

1. Raise the task or container memory
2. Size runtimes to the container: `-XX:MaxRAMPercentage=75` for Java, `--max-old-space-size` for Node.js
3. Fix the leak (take a heap dump before you hit the limit)

**Prevent →** Turn on Container Insights, alarm on memory utilization above 85%, and load-test with production-like data.

</details>

### Q130. The ECS service event says `service api was unable to place a task because no container instance met all of its requirements. The closest matching container-instance ... has insufficient memory available.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** On the EC2 launch type, no container instance has enough free CPU, memory or ports (static host ports with `bridge` mode clash), or placement constraints such as `distinctInstance` rule them out. Capacity provider scaling may be off.

**Diagnose →**

```bash
aws ecs describe-services --cluster prod --services api --query 'services[].events[:5]'
aws ecs list-container-instances --cluster prod
aws ecs describe-container-instances --cluster prod --container-instances <arn> \
  --query 'containerInstances[].remainingResources[?name==`MEMORY` || name==`CPU`]'
```

**Fix →**

1. Use a capacity provider with managed scaling, so the ASG grows when tasks are pending
2. Use `awsvpc` or dynamic host ports (hostPort 0) instead of fixed ports
3. Right-size task reservations, or move to Fargate

**Prevent →** Use capacity providers with `targetCapacity` around 90%, and alarm on pending tasks.

</details>

### Q131. An ECS deployment rolls back automatically, and the event says `deployment failed: tasks failed to start` from the circuit breaker. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The new tasks crash at startup (bad config or secret, failed migration), fail the container health check, or fail ALB health checks before `healthCheckGracePeriodSeconds`. The deployment circuit breaker detected repeated failures and rolled back.

**Diagnose →**

```bash
aws ecs describe-services --cluster prod --services api \
  --query 'services[].[deployments,events[:10]]'
aws ecs list-tasks --cluster prod --service-name api --desired-status STOPPED
aws ecs describe-tasks --cluster prod --tasks <stopped-task> --query 'tasks[].[stoppedReason,containers[].[exitCode,reason]]'
aws logs tail /ecs/api --since 20m --filter-pattern ERROR
```

**Fix →**

1. Fix the root cause from the stopped-task reasons and logs
2. If the app is slow to start, raise `healthCheckGracePeriodSeconds` and the container health check `startPeriod`
3. Redeploy

**Prevent →** Keep the circuit breaker with rollback on, add CloudWatch alarm-based rollback, and smoke-test images in staging with the same task definition.

</details>

### Q132. `aws ecs execute-command` fails: `The execute command failed because execute command was not enabled when the task was run or the execute command agent isn't running.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The task was started without `enableExecuteCommand`. Other causes: the task role has no `ssmmessages:*` permissions, the private subnet cannot reach `ssmmessages`, or the Session Manager plugin is not installed locally.

**Diagnose →**

```bash
aws ecs describe-tasks --cluster prod --tasks <task-arn> \
  --query 'tasks[].[enableExecuteCommand,containers[].managedAgents]'
# AWS's open-source checker:
bash <(curl -s https://raw.githubusercontent.com/aws-containers/amazon-ecs-exec-checker/main/check-ecs-exec.sh) prod <task-id>
```

**Fix →**

1. `aws ecs update-service --cluster prod --service api --enable-execute-command --force-new-deployment` (existing tasks must be replaced)
2. Give the task role `ssmmessages:CreateControlChannel`, `CreateDataChannel`, `OpenControlChannel` and `OpenDataChannel`
3. Add the `ssmmessages` endpoint for private subnets

**Prevent →** Enable exec in non-production by default, log sessions to CloudWatch or S3, and restrict it with IAM conditions in production.

</details>

### Q133. `kubectl get pods` returns `error: You must be logged in to the server (Unauthorized)` on a new EKS cluster. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your IAM principal is not mapped into the cluster. Only the cluster creator gets admin automatically (and only if bootstrap admin was on). Other causes: the kubeconfig uses a different profile or role than the one that has an access entry.

**Diagnose →**

```bash
aws sts get-caller-identity
aws eks update-kubeconfig --name prod --region us-east-1
aws eks list-access-entries --cluster-name prod
aws eks describe-cluster --name prod --query 'cluster.accessConfig'
```

**Fix →**

```bash
aws eks create-access-entry --cluster-name prod --principal-arn arn:aws:iam::111111111111:role/platform-admin
aws eks associate-access-policy --cluster-name prod --principal-arn arn:aws:iam::111111111111:role/platform-admin \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy --access-scope type=cluster
```

**Prevent →** Use authentication mode `API` with access entries managed in IaC, not the `aws-auth` ConfigMap.

</details>

### Q134. New EKS worker nodes never join; `kubectl get nodes` does not list them, while the EC2 instances are running. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The node IAM role has no access entry (type `EC2_LINUX`) or `aws-auth` mapping. Other causes: the node SG cannot reach the API endpoint (private endpoint, missing 443 rule), the nodes cannot reach ECR or STS (no NAT or endpoints), or the bootstrap or `nodeadm` config has the wrong cluster name or CA.

**Diagnose →**

```bash
aws eks list-access-entries --cluster-name prod
aws eks describe-nodegroup --cluster-name prod --nodegroup-name ng-1 --query 'nodegroup.health'
# on the node via SSM:
sudo journalctl -u kubelet --no-pager | tail -50
sudo journalctl -u nodeadm-run --no-pager | tail -20   # AL2023
```

**Fix →**

1. Run `aws eks create-access-entry --cluster-name prod --principal-arn <node-role-arn> --type EC2_LINUX`
2. Allow 443 from the node SG to the cluster SG, and add NAT or endpoints (`ecr.api`, `ecr.dkr`, `s3`, `sts`, `ec2`)

**Prevent →** Use managed node groups or Karpenter, which create the access entries, and check node health issues in `describe-nodegroup`.

</details>

### Q135. Pods stay `Pending` with `0/6 nodes are available: 6 Too many pods`, but the nodes have plenty of CPU and memory. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With the VPC CNI, every pod gets a VPC IP, and max pods per node is limited by ENIs × IPs per ENI (for example, 29 on `m5.large`). Small instance types fill up quickly. The subnet may also be running out of IPs (`failed to assign an IP address to container` in aws-node logs).

**Diagnose →**

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,MAXPODS:.status.allocatable.pods
kubectl -n kube-system logs ds/aws-node -c aws-node | grep -i -E 'insufficient|failed to assign'
aws ec2 describe-subnets --subnet-ids subnet-0abc --query 'Subnets[].AvailableIpAddressCount'
```

**Fix →**

1. Turn on prefix delegation: `kubectl set env ds/aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true`, and raise max pods (managed node groups and Karpenter calculate it)
2. Use larger instances
3. If the subnet is out of IPs, add a secondary CIDR (for example 100.64.0.0/16) with custom networking

**Prevent →** Plan pod IP capacity at design time (/19 or larger node subnets), and alarm on free IPs per subnet.

</details>

### Q136. A pod's S3 calls fail with `AccessDenied ... User: arn:aws:sts::111111111111:assumed-role/eks-node-role/i-0abc is not authorized`. Why is it using the node role? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The pod has no working IRSA or Pod Identity setup, so the SDK falls back to the node's instance profile through IMDS. Common reasons: the service account has no `eks.amazonaws.com/role-arn` annotation (IRSA), or no Pod Identity association exists, or the Pod Identity Agent add-on is missing, or the pod was created before the association and was never restarted. An old SDK that does not support the credential source does the same.

**Diagnose →**

```bash
kubectl get sa app -n prod -o yaml | grep -A2 annotations
aws eks list-pod-identity-associations --cluster-name prod --namespace prod
kubectl exec -n prod deploy/app -- env | grep -E 'AWS_ROLE_ARN|AWS_WEB_IDENTITY|AWS_CONTAINER_CREDENTIALS_FULL_URI'
kubectl get ds -n kube-system eks-pod-identity-agent
```

**Fix →**

1. Pod Identity: install the add-on, then run `aws eks create-pod-identity-association --cluster-name prod --namespace prod --service-account app --role-arn <role>`. The role trust policy allows `pods.eks.amazonaws.com` with `sts:AssumeRole` and `sts:TagSession`
2. Restart the pods so the credentials get injected

**Prevent →** Block pod access to IMDS (set the hop limit to 1 on nodes) so a missing setup fails loudly instead of quietly using node permissions.

</details>

### Q137. An Ingress never gets an ALB, and the AWS Load Balancer Controller logs `Failed build model due to couldn't auto-discover subnets: unable to resolve at least one subnet`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The subnets do not have the discovery tags: `kubernetes.io/role/elb=1` for public subnets (internet-facing) or `kubernetes.io/role/internal-elb=1` for private ones. Another cause: fewer than two AZs qualify.

**Diagnose →**

```bash
kubectl -n kube-system logs deploy/aws-load-balancer-controller | tail -30
kubectl describe ingress web -n prod
aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-0abc \
  --query 'Subnets[].[SubnetId,AvailabilityZone,Tags[?starts_with(Key,`kubernetes.io/role`)]]'
```

**Fix →**

```bash
aws ec2 create-tags --resources subnet-0pub1 subnet-0pub2 --tags Key=kubernetes.io/role/elb,Value=1
```

Or list the subnets explicitly with the annotation `alb.ingress.kubernetes.io/subnets`. Also check that the controller's IAM role has the current policy, because new features need new permissions.

**Prevent →** Tag subnets in the VPC module, and pin the controller and its IAM policy version together.

</details>

### Q138. An EKS cluster upgrade to the next minor version is blocked, or workloads break right after it. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Manifests or Helm charts still use removed APIs. Add-ons (VPC CNI, CoreDNS, kube-proxy, EBS CSI) are at versions that do not support the new Kubernetes version. Or nodes are more than the allowed skew behind the control plane.

**Diagnose →**

```bash
aws eks list-insights --cluster-name prod --filter kubernetesVersions=1.33
aws eks describe-insight --cluster-name prod --id <insight-id>
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.33 \
  --query 'addons[].addonVersions[].addonVersion' --output text | head
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
```

**Fix →**

1. Migrate manifests (`kubectl-convert`, `pluto detect-helm`)
2. Upgrade the add-ons to compatible versions **before** the control plane, then upgrade the control plane, then the nodes
3. Update PDBs so node drains can finish

**Prevent →** Upgrade one minor version at a time, every quarter, rehearsed on a staging cluster. Track EKS upgrade insights in CI.

</details>

### Q139. A PVC stays `Pending` with `waiting for a volume to be created, either by external provisioner "ebs.csi.aws.com" or manually created by system administrator`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The EBS CSI driver add-on is not installed, or its controller has no IAM permissions (no IRSA or Pod Identity role with `AmazonEBSCSIDriverPolicy`). Or the StorageClass uses `WaitForFirstConsumer` and no pod has been scheduled yet, which is normal.

**Diagnose →**

```bash
kubectl describe pvc data-db-0 -n prod
aws eks describe-addon --cluster-name prod --addon-name aws-ebs-csi-driver --query 'addon.[status,serviceAccountRoleArn]'
kubectl -n kube-system logs deploy/ebs-csi-controller -c csi-provisioner | tail -20
kubectl get sc
```

**Fix →**

```bash
aws eks create-addon --cluster-name prod --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::111111111111:role/ebs-csi-role
```

If the volumes are KMS-encrypted, add KMS permissions for that key to the role.

**Prevent →** Install CSI drivers as managed add-ons with their roles in IaC, and set a default gp3 StorageClass.

</details>

### Q140. A deployment fails with `ErrImagePull ... manifest unknown` (or ECS `CannotPullContainerError: ... not found`) for an image that used to work. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The tag does not exist in that repository or Region. A typo, a pipeline that never pushed, or an ECR lifecycle policy that expired old tags.

**Diagnose →**

```bash
aws ecr describe-images --repository-name api --image-ids imageTag=1.4.2
aws ecr get-lifecycle-policy --repository-name api
aws ecr describe-images --repository-name api --query 'sort_by(imageDetails,&imagePushedAt)[-5:].imageTags'
```

**Fix →**

1. Push the missing tag again, or deploy an existing digest
2. Change the lifecycle policy so it keeps images that are still deployed (for example, keep the last 50, and expire only untagged images after 14 days)

**Prevent →** Deploy by immutable digest, turn on tag immutability, and make lifecycle rules generous enough for rollbacks.

</details>

### Q141. A container starts and immediately exits with `exec /usr/local/bin/app: exec format error` on the new Graviton nodes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The image was built only for `linux/amd64` and is running on an `arm64` (Graviton) node, or the other way round.

**Diagnose →**

```bash
docker buildx imagetools inspect 111111111111.dkr.ecr.us-east-1.amazonaws.com/api:1.4
kubectl get nodes -L kubernetes.io/arch
```

**Fix →**

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t 111111111111.dkr.ecr.us-east-1.amazonaws.com/api:1.4 --push .
```

For a quick fix, pin the workload with `nodeSelector: kubernetes.io/arch: amd64`, or set ECS `runtimePlatform.cpuArchitecture`.

**Prevent →** Build multi-arch images in CI, and run arm64 smoke tests before you add Graviton capacity.

</details>

### Q142. Lambda logs end with `Task timed out after 3.00 seconds`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The default timeout is 3 seconds. The function is slow, or it hangs on a network call: a VPC function with no route to the internet or endpoint, a DB connection without a timeout, or a downstream that is slow.

**Diagnose →**

```bash
aws lambda get-function-configuration --function-name orders-api --query '[Timeout,MemorySize,VpcConfig]'
aws logs tail /aws/lambda/orders-api --since 15m
# enable X-Ray active tracing to see which segment takes the time
```

**Fix →**

1. Set client timeouts on every network call below the function timeout
2. Raise the timeout to a realistic value: `aws lambda update-function-configuration --function-name orders-api --timeout 15`
3. Raising memory also gives more CPU, which speeds up CPU-bound code

**Prevent →** Alarm on the `Duration` p99 against the timeout, and turn on X-Ray tracing.

</details>

### Q143. A Lambda function was moved into a VPC, and now every AWS SDK call hangs until timeout. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Lambda ENIs in a VPC never get public IPs, even in public subnets. Without a NAT gateway or VPC endpoints, the function cannot reach AWS APIs or the internet.

**Diagnose →**

```bash
aws lambda get-function-configuration --function-name worker --query 'VpcConfig'
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-0priv --query 'RouteTables[].Routes'
```

**Fix →**

1. Put the function in **private** subnets with a route to a NAT gateway
2. Or add interface or gateway endpoints for the services it calls (S3, DynamoDB gateway endpoints are free)
3. Allow outbound 443 in the function's SG

**Prevent →** Only attach Lambda to a VPC when it needs private resources, and always pair that with endpoints or NAT.

</details>

### Q144. `Runtime.ImportModuleError: Unable to import module 'app': No module named 'requests'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Third-party dependencies were not packaged with the function code (the zip contains only `app.py`). Other causes: they were built for the wrong platform (native wheels built on macOS), or the handler path is wrong.

**Diagnose →**

```bash
aws lambda get-function-configuration --function-name api --query '[Handler,Runtime,Layers]'
unzip -l function.zip | head
```

**Fix →**

```bash
pip install -r requirements.txt -t package/ --platform manylinux2014_x86_64 --only-binary=:all:
cp app.py package/ && (cd package && zip -r ../function.zip .)
aws lambda update-function-code --function-name api --zip-file fileb://function.zip
```

Or move shared dependencies into a layer, or ship the function as a container image.

**Prevent →** Build packages in CI with SAM or a container matching the Lambda runtime, never from a laptop.

</details>

### Q145. Callers get `TooManyRequestsException: Rate Exceeded` from Lambda and the `Throttles` metric spikes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You hit concurrency limits: the account limit in the Region (1,000 by default), the function's reserved concurrency, or another function used up the shared pool.

**Diagnose →**

```bash
aws lambda get-account-settings --query 'AccountLimit.[ConcurrentExecutions,UnreservedConcurrentExecutions]'
aws lambda get-function-concurrency --function-name orders-api
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name ConcurrentExecutions \
  --statistics Maximum --period 60 --start-time $(date -u -d '-1 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ)
```

**Fix →**

1. Request an account concurrency increase through Service Quotas
2. Give critical functions reserved concurrency, and cap noisy ones
3. Put SQS in front of bursty async producers

**Prevent →** Alarm on `ConcurrentExecutions` near the limit, and on `Throttles` above 0 for synchronous APIs.

</details>

### Q146. A Java Lambda behind API Gateway has p99 latency of 6–8 seconds on the first request after idle periods. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cold starts: JVM startup, heavy framework init (Spring) and large packages. VPC attachment adds little now (Hyperplane ENIs), so init code is usually the cause.

**Diagnose →**

```bash
aws logs filter-log-events --log-group-name /aws/lambda/orders-api \
  --filter-pattern '"Init Duration"' --max-items 5 --query 'events[].message'
```

**Fix →**

1. Turn on **SnapStart** (Java, Python, .NET): `aws lambda update-function-configuration --function-name orders-api --snap-start ApplyOn=PublishedVersions`, then publish a version and point the alias at it
2. Or use provisioned concurrency on an alias for predictable traffic
3. Trim dependencies, initialize lazily, and raise memory (more CPU during init)

**Prevent →** Track `Init Duration` as an SLO input, and choose lighter runtimes or frameworks for latency-critical paths.

</details>

### Q147. Deploying fails: `InvalidParameterValueException: Unzipped size must be smaller than 262144000 bytes`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The function code plus all layers is larger than 250 MB unzipped. Large ML libraries (numpy, pandas, torch) or bundled test files and caches usually cause it.

**Diagnose →**

```bash
unzip -l function.zip | tail -1
aws lambda get-function --function-name ml-infer --query 'Configuration.Layers[].CodeSize'
du -sh package/* | sort -h | tail
```

**Fix →**

1. Package the function as a **container image** (up to 10 GB)
2. Or remove tests, `__pycache__` and docs from the package, and use AWS-provided layers (for example AWS SDK for pandas)

**Prevent →** Use container images for heavy dependencies, and fail CI builds when package size grows past a set budget.

</details>

### Q148. An SQS-triggered Lambda processes the same messages again and again, and some end up in the DLQ even though the code succeeded. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The queue's visibility timeout is shorter than the function timeout, so messages become visible again while they are still being processed. Or one failing record makes the whole batch retry, because partial batch responses are not enabled.

**Diagnose →**

```bash
aws sqs get-queue-attributes --queue-url <url> --attribute-names VisibilityTimeout RedrivePolicy
aws lambda get-function-configuration --function-name consumer --query Timeout
aws lambda list-event-source-mappings --function-name consumer --query 'EventSourceMappings[].[BatchSize,FunctionResponseTypes]'
```

**Fix →**

1. Set the visibility timeout to at least 6× the function timeout (AWS guidance)
2. Turn on `ReportBatchItemFailures` and return only the failed message IDs: `aws lambda update-event-source-mapping --uuid <id> --function-response-types ReportBatchItemFailures`
3. Make handlers idempotent

**Prevent →** Use a standard SQS-Lambda module with these defaults, and alarm on DLQ depth above 0.

</details>

### Q149. A Lambda dies with `Runtime exited with error: signal: killed` and `Runtime.ExitError`, and no stack trace. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The function ran out of memory. The REPORT line shows `Max Memory Used` equal to `Memory Size`. Large files loaded into memory, or a leak across warm invocations from global caches, are common reasons.

**Diagnose →**

```bash
aws logs filter-log-events --log-group-name /aws/lambda/etl \
  --filter-pattern 'REPORT' --max-items 5 --query 'events[].message'
```

**Fix →**

1. Raise `MemorySize`. Use Lambda Power Tuning to find the cheapest setting that works
2. Stream large objects instead of loading them whole, and use `/tmp` (configurable up to 10 GB) for scratch space
3. Clear global caches between invocations

**Prevent →** Alarm when `Max Memory Used` goes above 80% of the configured memory (through Lambda Insights).

</details>

### Q150. An S3-triggered Lambda bill exploded, and you get a notification that `Lambda stopped a recursive invocation loop`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The function writes its output back into the **same bucket and prefix** that triggers it, so every write triggers the function again. Lambda recursive loop detection (for SQS, SNS, S3 and Lambda) stops it after about 16 iterations, but only for supported patterns.

**Diagnose →**

```bash
aws s3api get-bucket-notification-configuration --bucket images
aws lambda get-function-recursion-config --function-name thumbnailer
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name RecursiveInvocationsDropped \
  --dimensions Name=FunctionName,Value=thumbnailer --statistics Sum --period 300 \
  --start-time $(date -u -d '-1 day' +%FT%TZ) --end-time $(date -u +%FT%TZ)
```

**Fix →**

1. Set reserved concurrency to 0 right away to stop it: `aws lambda put-function-concurrency --function-name thumbnailer --reserved-concurrent-executions 0`
2. Write output to a different bucket, or scope the trigger to the `uploads/` prefix and write to `thumbs/`

**Prevent →** Keep recursion detection set to `Terminate`, use prefix or suffix filters on triggers, and set billing alarms on Lambda spend.

</details>

---

## Route 53, CloudFront, API Gateway and Messaging

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. You created a public hosted zone and records for `example.com`, but `dig example.com` returns `NXDOMAIN` or old records. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The registrar still points to the old name servers, not to the four NS records of the new hosted zone. Another cause: two hosted zones exist for the same name and you edited the one that is not delegated.

**Diagnose →**

```bash
aws route53 list-hosted-zones-by-name --dns-name example.com
aws route53 get-hosted-zone --id Z0123ABC --query 'DelegationSet.NameServers'
dig NS example.com +short
dig @a.gtld-servers.net NS example.com +norecurse   # what the parent zone delegates to
```

**Fix →**

1. Update the NS at the registrar to match the zone's delegation set. For Route 53 Domains, run `aws route53domains update-domain-nameservers`
2. Delete the duplicate zone

**Prevent →** Use one hosted zone per domain in IaC, with an output that checks delegation, and lower NS TTLs before migrations.

</details>

### Q152. `InvalidChangeBatch: RRSet of type CNAME with DNS name example.com. is not permitted at apex in zone example.com.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DNS does not allow a CNAME at the zone apex, because it would clash with the SOA and NS records there.

**Diagnose →**

```bash
aws route53 list-resource-record-sets --hosted-zone-id Z0123ABC --query "ResourceRecordSets[?Name=='example.com.']"
```

**Fix →** Use a Route 53 **alias A/AAAA** record to the ALB, CloudFront or S3 target:

```bash
aws route53 change-resource-record-sets --hosted-zone-id Z0123ABC --change-batch '{
 "Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"example.com","Type":"A",
 "AliasTarget":{"HostedZoneId":"Z35SXDOTRQ7X7K","DNSName":"dualstack.web-123.us-east-1.elb.amazonaws.com","EvaluateTargetHealth":true}}}]}'
```

The alias `HostedZoneId` is the **load balancer's** canonical zone ID, not your zone's ID.

**Prevent →** Use alias records for AWS targets everywhere. They are free to query and follow IP changes automatically.

</details>

### Q153. The primary Region is down, but Route 53 failover records still send traffic to it. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The primary record has no health check associated, or `EvaluateTargetHealth` is false on the alias. The health check tests something that is still up (for example, the ALB answers `/` even though the app is broken). Or clients cache DNS beyond the TTL.

**Diagnose →**

```bash
aws route53 list-resource-record-sets --hosted-zone-id Z0123ABC \
  --query "ResourceRecordSets[?Failover!=null].[Name,Failover,HealthCheckId,AliasTarget.EvaluateTargetHealth]"
aws route53 get-health-check-status --health-check-id <id>
```

**Fix →**

1. Attach a health check that tests a deep `/healthz` endpoint (checking the app and its dependencies) to the PRIMARY record
2. Set `EvaluateTargetHealth: true` on alias records
3. Use a 60s TTL

**Prevent →** Run regular game days that fail the primary, and consider Route 53 ARC routing controls for manual, deterministic failover.

</details>

### Q154. A Route 53 health check reports `Failure: Connection timed out` for an endpoint that works fine from your laptop. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SG, NACL or WAF does not allow the Route 53 health checker IP ranges. Other causes: the check uses HTTP while only HTTPS is open, or the host header or SNI is wrong.

**Diagnose →**

```bash
aws route53 get-health-check --health-check-id <id> --query 'HealthCheck.HealthCheckConfig'
aws route53 get-health-check-last-failure-reason --health-check-id <id>
curl -s https://ip-ranges.amazonaws.com/ip-ranges.json | jq -r '.prefixes[]|select(.service=="ROUTE53_HEALTHCHECKS").ip_prefix' | head
```

**Fix →**

1. Allow the `ROUTE53_HEALTHCHECKS` ranges (or the managed prefix list `com.amazonaws.route53-healthchecks` where available) on the port
2. Set `FullyQualifiedDomainName` and `EnableSNI` for HTTPS checks

**Prevent →** For private resources, use CloudWatch alarm-based (calculated) health checks, because Route 53 checkers cannot reach private IPs.

</details>

### Q155. You changed an A record an hour ago, but some users still reach the old server, and a newly created name returned `NXDOMAIN` for a while. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Resolvers cache the old answer for the record's TTL (it was 86400). For the new name, the earlier `NXDOMAIN` was **negatively cached** for the SOA minimum TTL (default 900s in Route 53, but it can be set higher). Some clients and JVMs cache longer still.

**Diagnose →**

```bash
dig app.example.com +noall +answer          # remaining TTL from your resolver
dig @ns-123.awsdns-45.com app.example.com   # authoritative answer
aws route53 list-resource-record-sets --hosted-zone-id Z0123ABC --query "ResourceRecordSets[?Type=='SOA']"
```

**Fix →**

1. Wait it out, or keep the old server forwarding or redirecting until the TTL passes
2. Flush local caches (`resolvectl flush-caches`)

**Prevent →** Lower TTLs to 60s at least one old-TTL period **before** a migration, avoid querying names before you create them, and keep the SOA negative TTL reasonable.

</details>

### Q156. CloudFront returns `403 AccessDenied` (S3 XML) for every object after you switched the S3 origin to Origin Access Control. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bucket policy does not allow the `cloudfront.amazonaws.com` service principal with `AWS:SourceArn` set to this distribution. Other causes: objects are SSE-KMS and the key policy does not allow CloudFront, the request has no default root object, or the key is missing (S3 returns 403 instead of 404 without `s3:ListBucket`).

**Diagnose →**

```bash
aws cloudfront get-distribution-config --id E123ABC --query 'DistributionConfig.[DefaultRootObject,Origins.Items[].[DomainName,OriginAccessControlId]]'
aws s3api get-bucket-policy --bucket site-assets --output text
aws s3api head-object --bucket site-assets --key index.html
```

**Fix →**

```json
{"Effect":"Allow","Principal":{"Service":"cloudfront.amazonaws.com"},
 "Action":"s3:GetObject","Resource":"arn:aws:s3:::site-assets/*",
 "Condition":{"StringEquals":{"AWS:SourceArn":"arn:aws:cloudfront::111111111111:distribution/E123ABC"}}}
```

Add a `kms:Decrypt` statement for CloudFront to the key policy if the objects use SSE-KMS, and set `DefaultRootObject` to `index.html`.

**Prevent →** Create the OAC, the bucket policy and the distribution together in one IaC module.

</details>

### Q157. CloudFront shows `502 ERROR. The request could not be satisfied. CloudFront wasn't able to connect to the origin.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CloudFront → origin TLS handshake fails: the origin certificate does not match the origin domain name, it is self-signed or expired, or the chain is incomplete. Or the origin protocol policy is wrong (HTTPS-only to an origin that only speaks HTTP), or the SG blocks CloudFront.

**Diagnose →**

```bash
aws cloudfront get-distribution-config --id E123ABC \
  --query 'DistributionConfig.Origins.Items[].[DomainName,CustomOriginConfig]'
openssl s_client -connect origin.example.com:443 -servername origin.example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -dates
```

**Fix →**

1. Use an origin domain covered by a valid public certificate (for example, an ACM certificate on the ALB for `origin.example.com`), and forward the matching Host or SNI
2. Match the origin protocol policy and ports to what the origin serves
3. Allow the `com.amazonaws.global.cloudfront.origin-facing` prefix list on the ALB SG

**Prevent →** Monitor origin certificate expiry, and use CloudFront VPC origins or origin-facing prefix lists for ALB origins.

</details>

### Q158. After deploying a new frontend build, users still see the old JavaScript for hours. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CloudFront (and browsers) cache the objects for the TTL set by the cache policy or by `Cache-Control` headers. Files with the same name, such as `app.js` and `index.html`, are not refetched.

**Diagnose →**

```bash
curl -sI https://www.example.com/app.js | grep -iE 'x-cache|age|cache-control|etag'
aws cloudfront list-invalidations --distribution-id E123ABC --max-items 3
```

**Fix →**

```bash
aws cloudfront create-invalidation --distribution-id E123ABC --paths "/index.html" "/app.js"
```

**Prevent →** Use content-hashed asset names (`app.3f9c2.js`) with long TTLs and `immutable`. Give `index.html` `Cache-Control: no-cache` (or a short TTL), and invalidate only `index.html` on deploy.

</details>

### Q159. When editing a CloudFront distribution, your ACM certificate does not appear in the dropdown. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CloudFront only uses ACM certificates from **us-east-1 (N. Virginia)**. Other causes: the certificate is not yet `ISSUED`, or it does not cover the alternate domain name you added.

**Diagnose →**

```bash
aws acm list-certificates --region us-east-1 --query 'CertificateSummaryList[].[DomainName,Status]'
aws acm describe-certificate --region us-east-1 --certificate-arn <arn> --query 'Certificate.SubjectAlternativeNames'
```

**Fix →** Request the certificate in us-east-1 with every CNAME (or a wildcard), validate it by DNS, then attach it.

**Prevent →** In Terraform, use a provider alias for us-east-1 just for CloudFront certificates.

</details>

### Q160. CloudFront cache hit ratio is under 20% and origin load is high, even for static content. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cache key includes high-cardinality values: all query strings (tracking parameters like `utm_*`), all cookies, or headers like `User-Agent` or `Authorization`. Legacy "forward all" settings split the cache. Short TTLs or `Cache-Control: no-store` from the origin also prevent caching.

**Diagnose →**

```bash
aws cloudfront get-distribution-config --id E123ABC \
  --query 'DistributionConfig.DefaultCacheBehavior.[CachePolicyId,OriginRequestPolicyId,ForwardedValues]'
aws cloudfront get-cache-policy --id <policy-id>
# CloudFront standard logs / real-time logs: x-edge-result-type Miss vs Hit, by URI
```

**Fix →**

1. Use a cache policy with a minimal key (for example, `CachingOptimized` for static paths)
2. Forward the values the origin needs through an **origin request policy**, which does not affect the cache key
3. Normalize or strip query strings, and set proper `Cache-Control` at the origin

**Prevent →** Use separate behaviors for `/static/*` and `/api/*`, and alarm on `CacheHitRate` (with additional metrics turned on).

</details>

### Q161. A React SPA on S3 + CloudFront: `/` works, but refreshing `/dashboard/settings` returns `403 AccessDenied`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** There is no S3 object at `dashboard/settings`. With OAC and no `ListBucket`, S3 returns 403 for missing keys, and the client-side router never loads.

**Diagnose →**

```bash
curl -sI https://app.example.com/dashboard/settings | head -3
aws cloudfront get-distribution-config --id E123ABC --query 'DistributionConfig.CustomErrorResponses'
```

**Fix →**

1. Add custom error responses that map 403 and 404 to `/index.html` with response code 200
2. Or (cleaner) use a CloudFront Function on viewer-request that rewrites paths with no file extension to `/index.html`, so real API 404s are not masked

**Prevent →** Make it part of your SPA hosting module, and keep APIs on a separate behavior or origin.

</details>

### Q162. Adding an alternate domain name fails: `CNAMEAlreadyExists: One or more of the CNAMEs you provided are already associated with a different resource.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another distribution, possibly in another account (an old environment or a vendor), already has that alternate domain name.

**Diagnose →**

```bash
aws cloudfront list-conflicting-aliases --distribution-id E123ABC --alias www.example.com
aws cloudfront list-distributions --query "DistributionList.Items[?contains(Aliases.Items,'www.example.com')].Id"
```

**Fix →**

1. Same account: remove the alias from the old distribution, then add it to the new one
2. Different account: prove ownership with a TXT record (`_www.example.com` pointing to the new distribution's domain) and run `aws cloudfront associate-alias --target-distribution-id E123ABC --alias www.example.com`

**Prevent →** Track aliases in IaC, and remove them when you decommission distributions.

</details>

### Q163. Calling your API returns `{"message":"Missing Authentication Token"}`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Despite the name, this almost always means **the route does not exist**: a wrong path or method, a missing stage in the URL (`/prod/orders`), or the API was not deployed after the change (REST APIs need a deployment).

**Diagnose →**

```bash
aws apigateway get-resources --rest-api-id abc123 --query 'items[].[path,resourceMethods]'
aws apigateway get-stages --rest-api-id abc123 --query 'item[].[stageName,deploymentId,lastUpdatedDate]'
curl -i https://abc123.execute-api.us-east-1.amazonaws.com/prod/orders
```

**Fix →**

1. Call the correct stage, path and method
2. Redeploy: `aws apigateway create-deployment --rest-api-id abc123 --stage-name prod`

**Prevent →** Deploy APIs through IaC, which creates deployments automatically, and add a smoke test per route after each deploy.

</details>

### Q164. API Gateway returns `502` with `{"message": "Internal server error"}`, and the execution log shows `Execution failed due to configuration error: Malformed Lambda proxy response`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With Lambda proxy integration, the function must return `{"statusCode": int, "headers": {...}, "body": "<string>"}`. A dict or object as `body`, a missing `statusCode`, or an unhandled exception all produce this error.

**Diagnose →**

```bash
aws apigateway update-stage --rest-api-id abc123 --stage-name prod \
  --patch-operations op=replace,path=/*/*/logging/loglevel,value=INFO
aws logs tail API-Gateway-Execution-Logs_abc123/prod --since 10m
aws lambda invoke --function-name orders-api --payload fileb://event.json out.json && cat out.json
```

**Fix →** Return the exact proxy shape and serialize the body:

```python
return {"statusCode": 200, "headers": {"Content-Type": "application/json"}, "body": json.dumps(result)}
```

Catch exceptions and return a 500 response yourself.

**Prevent →** Use a framework (Powertools for AWS Lambda event handler) that builds responses, and add contract tests with a sample API Gateway event.

</details>

### Q165. API Gateway returns `504` with `{"message": "Endpoint request timed out"}` on long requests. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The integration took longer than the integration timeout. The default is 29 seconds; for Regional and private REST APIs it can be raised with a quota increase, which may lower throttle limits.

**Diagnose →**

```bash
aws apigateway get-integration --rest-api-id abc123 --resource-id res1 --http-method POST --query timeoutInMillis
aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name IntegrationLatency \
  --dimensions Name=ApiName,Value=orders --extended-statistics p99 --period 60 \
  --start-time $(date -u -d '-1 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ)
```

**Fix →**

1. Make the long work asynchronous: the API puts a job on SQS or Step Functions and returns `202` with a job ID, and the client polls or gets a callback
2. Optimize the backend call

**Prevent →** Design APIs so every synchronous call finishes in a few seconds.

</details>

### Q166. The browser shows a CORS error only when the API returns 401 or 4XX, while successful calls work. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Lambda adds `Access-Control-Allow-Origin` to its own responses, but errors produced by **API Gateway itself** (authorizer denies, throttling, missing routes) are gateway responses without CORS headers. The browser then hides the real 401 behind a CORS error.

**Diagnose →**

```bash
curl -i -X GET https://abc123.execute-api.us-east-1.amazonaws.com/prod/me -H 'Origin: https://app.example.com'
aws apigateway get-gateway-responses --rest-api-id abc123 --query 'items[].[responseType,responseParameters]'
```

**Fix →**

```bash
aws apigateway put-gateway-response --rest-api-id abc123 --response-type DEFAULT_4XX \
  --response-parameters '{"gatewayresponse.header.Access-Control-Allow-Origin":"'"'"'https://app.example.com'"'"'"}'
aws apigateway create-deployment --rest-api-id abc123 --stage-name prod
```

Do the same for `DEFAULT_5XX`. On HTTP APIs, use the built-in CORS configuration, which covers this.

**Prevent →** Define CORS once at the API level in IaC, and test error paths from a browser origin.

</details>

### Q167. A REST API returns `{"message":"Forbidden"}` (403) to a client that sends the right path. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The method requires an API key and the `x-api-key` header is missing, or the key is not in a usage plan tied to this stage. For a **private** API, the call does not come through the allowed VPC endpoint, or the resource policy denies it. A WAF rule may also block it.

**Diagnose →**

```bash
aws apigateway get-method --rest-api-id abc123 --resource-id res1 --http-method GET --query apiKeyRequired
aws apigateway get-usage-plans --query 'items[].[name,apiStages]'
aws apigateway get-rest-api --rest-api-id abc123 --query '[endpointConfiguration,policy]'
```

**Fix →**

1. Send `x-api-key`, and link the key to a usage plan with this stage: `aws apigateway create-usage-plan-key --usage-plan-id up1 --key-id k1 --key-type API_KEY`
2. For private APIs, call through the execute-api VPC endpoint and allow `aws:SourceVpce` in the resource policy

**Prevent →** Do not use API keys as authentication (use IAM, Cognito or JWT authorizers), and turn on access logging with `$context.error.responseType`.

</details>

### Q168. Clients get `429 Too Many Requests` with `{"message":"Too Many Requests"}` from API Gateway at modest traffic. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Throttling at one of the levels: the usage plan or API key limit, a stage or method throttle set low, or the account-level Regional limit (10,000 rps steady with a 5,000 burst by default) shared by all APIs.

**Diagnose →**

```bash
aws apigateway get-stage --rest-api-id abc123 --stage-name prod --query methodSettings
aws apigateway get-usage-plan --usage-plan-id up1 --query '[throttle,quota]'
aws apigateway get-account --query throttleSettings
```

**Fix →**

1. Raise the stage, method or usage plan throttles to match the expected load
2. Request an account limit increase if needed
3. Make clients retry with backoff and jitter

**Prevent →** Load-test with real limits, alarm on the `4XXError` rate, and document per-client quotas.

</details>

### Q169. Messages keep landing in the SQS dead-letter queue even though consumers look healthy. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Each message was received `maxReceiveCount` times without being deleted. Reasons: consumers crash or time out on certain payloads ("poison messages"), the visibility timeout is too short, or the code never calls `DeleteMessage`.

**Diagnose →**

```bash
aws sqs get-queue-attributes --queue-url <main-url> --attribute-names RedrivePolicy VisibilityTimeout
aws sqs receive-message --queue-url <dlq-url> --max-number-of-messages 5 \
  --attribute-names ApproximateReceiveCount --message-attribute-names All
```

**Fix →**

1. Inspect the DLQ payloads, and fix the parser or the code path
2. Make sure consumers delete messages only after success
3. Once fixed, redrive: `aws sqs start-message-move-task --source-arn <dlq-arn> --destination-arn <main-arn>`

**Prevent →** Alarm on `ApproximateNumberOfMessagesVisible` above 0 on every DLQ, and log the message ID on failures.

</details>

### Q170. You subscribed an email address to an SNS topic, but nothing arrives when you publish. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The subscription is still `PendingConfirmation`; the recipient never clicked the confirmation link (check spam). Another cause: a filter policy on the subscription does not match the message attributes.

**Diagnose →**

```bash
aws sns list-subscriptions-by-topic --topic-arn arn:aws:sns:us-east-1:111111111111:alerts \
  --query 'Subscriptions[].[Protocol,Endpoint,SubscriptionArn]'
aws sns get-subscription-attributes --subscription-arn <arn> --query 'Attributes.FilterPolicy'
```

**Fix →**

1. Confirm the subscription, or delete it and subscribe again to resend the email
2. Fix or remove the filter policy
3. Publish a test message: `aws sns publish --topic-arn <arn> --message test`

**Prevent →** Prefer chat integrations (Amazon Q Developer in chat applications, formerly AWS Chatbot) or an SQS/Lambda fan-out over personal emails for operations alerts.

</details>

### Q171. SNS in account A publishes successfully, but the SQS queue in account B never receives anything. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SQS queue policy does not allow `sqs:SendMessage` from `sns.amazonaws.com` with `aws:SourceArn` set to the topic. The queue uses SSE with the AWS managed key `aws/sqs`, which SNS cannot use. Or the cross-account subscription was never confirmed.

**Diagnose →**

```bash
aws sqs get-queue-attributes --queue-url <url> --attribute-names Policy KmsMasterKeyId
aws sns list-subscriptions-by-topic --topic-arn <topic-arn>
# SNS delivery status logging / NumberOfNotificationsFailed metric on the topic
```

**Fix →**

1. Add a queue policy statement that allows `sns.amazonaws.com` with `"ArnEquals":{"aws:SourceArn":"<topic-arn>"}`
2. Switch the queue to a CMK whose key policy allows `sns.amazonaws.com` to use `kms:GenerateDataKey*` and `kms:Decrypt`, or to SSE-SQS
3. Confirm the subscription from account B

**Prevent →** Use a reusable cross-account fan-out module, and alarm on `NumberOfNotificationsFailed`.

</details>

### Q172. An SQS FIFO queue has thousands of messages, but consumers receive nothing for minutes at a time. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** FIFO guarantees order **per message group**. While a message from a group is in flight (received but not deleted), no other message in that group is delivered. If most messages share one `MessageGroupId`, a single slow or failing message blocks everything behind it until the visibility timeout ends.

**Diagnose →**

```bash
aws sqs get-queue-attributes --queue-url <fifo-url> \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible VisibilityTimeout RedrivePolicy
# check producer code: what is MessageGroupId set to?
```

**Fix →**

1. Use a finer-grained group ID (per customer or order), so groups process in parallel
2. Add a DLQ so poison messages leave the group
3. Make sure consumers delete or extend visibility promptly

**Prevent →** Choose the group key for the ordering you really need. Use high-throughput FIFO mode only with many groups.

</details>

### Q173. An EventBridge rule should trigger a Lambda on EC2 state changes, but the Lambda never runs. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The event pattern does not match: a typo, wrong case, or values not wrapped in arrays. The rule is on the wrong event bus or Region. The Lambda resource policy does not allow `events.amazonaws.com`, which shows up as `FailedInvocations`. Or the rule is disabled.

**Diagnose →**

```bash
aws events describe-rule --name ec2-state --query '[State,EventBusName,EventPattern]'
aws events test-event-pattern --event-pattern file://pattern.json --event file://sample-event.json
aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name FailedInvocations \
  --dimensions Name=RuleName,Value=ec2-state --statistics Sum --period 300 \
  --start-time $(date -u -d '-1 day' +%FT%TZ) --end-time $(date -u +%FT%TZ)
aws lambda get-policy --function-name on-ec2-change
```

**Fix →**

1. Correct the pattern, for example `{"source":["aws.ec2"],"detail-type":["EC2 Instance State-change Notification"],"detail":{"state":["stopped"]}}`
2. Add the permission: `aws lambda add-permission --function-name on-ec2-change --statement-id evb --action lambda:InvokeFunction --principal events.amazonaws.com --source-arn <rule-arn>`
3. Add a DLQ on the target

**Prevent →** Unit-test patterns with `test-event-pattern` in CI, and alarm on `FailedInvocations`.

</details>

### Q174. Kinesis consumers fall behind: `GetRecords.IteratorAgeMilliseconds` keeps rising, and producers see `ProvisionedThroughputExceededException`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Shards are saturated (1 MB/s or 1,000 records/s in, 2 MB/s out per shard), often because of a hot partition key. Too many consumers share the 5 reads/s per shard limit. Or the consumer is slow (a Lambda batch that keeps failing and retrying blocks the shard).

**Diagnose →**

```bash
aws kinesis describe-stream-summary --stream-name clicks --query 'StreamDescriptionSummary.[StreamModeDetails,OpenShardCount]'
aws cloudwatch get-metric-statistics --namespace AWS/Kinesis --metric-name WriteProvisionedThroughputExceeded \
  --dimensions Name=StreamName,Value=clicks --statistics Sum --period 60 \
  --start-time $(date -u -d '-1 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ)
aws lambda list-event-source-mappings --function-name clicks-consumer --query 'EventSourceMappings[].[BatchSize,ParallelizationFactor,BisectBatchOnFunctionError]'
```

**Fix →**

1. Switch to on-demand mode, or reshard: `aws kinesis update-shard-count --stream-name clicks --target-shard-count 8 --scaling-type UNIFORM_SCALING`
2. Use a high-cardinality partition key
3. Use enhanced fan-out for multiple consumers
4. On the Lambda mapping, set `ParallelizationFactor`, `BisectBatchOnFunctionError`, `MaximumRetryAttempts` and an on-failure destination

**Prevent →** Alarm on iterator age, and turn on shard-level metrics to spot hot shards.

</details>

### Q175. The SDK fails with `AWS.SimpleQueueService.NonExistentQueue: The specified queue does not exist.` but the queue is in the console. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client is in a different Region or account from the queue, the queue URL or name has a typo (FIFO queues need the `.fifo` suffix), or the queue was deleted and re-created within 60 seconds.

**Diagnose →**

```bash
aws sts get-caller-identity
aws sqs get-queue-url --queue-name orders.fifo --region eu-west-1
aws sqs list-queues --queue-name-prefix orders --region eu-west-1
```

**Fix →** Use the full queue URL from `get-queue-url` with the matching Region, and add `--queue-owner-aws-account-id` for cross-account queues.

**Prevent →** Inject queue URLs from IaC outputs through environment variables or SSM, never hand-typed names.

</details>

---

## CloudFormation, CloudWatch, KMS, Secrets and Cost

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. Updating a stack fails: `Stack:arn:aws:cloudformation:...:stack/app/... is in ROLLBACK_COMPLETE state and can not be updated.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The **first** create of the stack failed and rolled back. A stack in `ROLLBACK_COMPLETE` cannot be updated; it can only be deleted.

**Diagnose →**

```bash
aws cloudformation describe-stack-events --stack-name app \
  --query "StackEvents[?ResourceStatus=='CREATE_FAILED'].[LogicalResourceId,ResourceStatusReason]" --output table
```

**Fix →**

1. Read the first `CREATE_FAILED` reason and fix the template
2. Run `aws cloudformation delete-stack --stack-name app && aws cloudformation wait stack-delete-complete --stack-name app`
3. Create the stack again

**Prevent →** Use `--disable-rollback` (or `--on-failure DO_NOTHING`) in dev to debug failed creates, and run `cfn-lint` in CI.

</details>

### Q177. A stack is stuck in `UPDATE_ROLLBACK_FAILED`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** During rollback, CloudFormation could not return a resource to its old state. Typical reasons: the resource was changed or deleted outside CloudFormation, a dependency is gone, or permissions were removed.

**Diagnose →**

```bash
aws cloudformation describe-stack-events --stack-name app \
  --query "StackEvents[?contains(ResourceStatus,'FAILED')].[Timestamp,LogicalResourceId,ResourceStatusReason]" --max-items 10
```

**Fix →**

1. Fix the underlying issue if you can (re-create the deleted dependency, restore permissions)
2. Run `aws cloudformation continue-update-rollback --stack-name app`
3. If a resource cannot be recovered, skip it: `aws cloudformation continue-update-rollback --stack-name app --resources-to-skip MyBrokenResource`. Then reconcile that resource manually, because CloudFormation now treats it as matching the old template

**Prevent →** Never change stack resources by hand. Use stack policies and drift detection, and deploy through change sets.

</details>

### Q178. A pipeline fails with `An error occurred (ValidationError) when calling the UpdateStack operation: No updates are to be performed.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The template and parameters are identical to what is deployed. CloudFormation treats this as an error for `update-stack`.

**Diagnose →**

```bash
aws cloudformation get-template --stack-name app --query TemplateBody > deployed.yaml
diff deployed.yaml template.yaml
```

**Fix →** Use `aws cloudformation deploy --no-fail-on-empty-changeset`, which handles this case. Or catch that specific message and treat it as success.

**Prevent →** Deploy with `aws cloudformation deploy` or change sets in pipelines, not raw `update-stack`.

</details>

### Q179. `InsufficientCapabilitiesException: Requires capabilities : [CAPABILITY_NAMED_IAM]`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The template creates IAM resources with custom names (`RoleName`, `PolicyName`). CloudFormation makes you acknowledge that explicitly. Templates that use `Transform` macros (SAM) also need `CAPABILITY_AUTO_EXPAND`.

**Diagnose →**

```bash
grep -nE 'AWS::IAM::|RoleName|Transform' template.yaml
```

**Fix →**

```bash
aws cloudformation deploy --template-file template.yaml --stack-name app \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

**Prevent →** Avoid hard-coded IAM names (let CloudFormation generate them) so you only need `CAPABILITY_IAM`, and let only reviewed pipelines pass capabilities.

</details>

### Q180. Stack deletion ends in `DELETE_FAILED` with `The bucket you tried to delete is not empty` on an S3 bucket resource. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CloudFormation does not empty buckets. A non-empty or versioned bucket blocks deletion. The same thing happens with ECR repositories that contain images and security groups still attached to ENIs.

**Diagnose →**

```bash
aws cloudformation describe-stack-events --stack-name app \
  --query "StackEvents[?ResourceStatus=='DELETE_FAILED'].[LogicalResourceId,PhysicalResourceId,ResourceStatusReason]"
```

**Fix →**

1. Empty the bucket, including all versions (see Q82), then delete the stack again
2. Or keep the bucket: `aws cloudformation delete-stack --stack-name app --retain-resources LogsBucket` (only allowed when the stack is in `DELETE_FAILED`)

**Prevent →** Set `DeletionPolicy: Retain` on data buckets, and use auto-delete custom resources (CDK `autoDeleteObjects`) only for ephemeral stacks.

</details>

### Q181. Drift detection shows `MODIFIED` on a security group and an RDS instance that someone changed in the console. How do you reconcile? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Changes made outside CloudFormation (console hotfix during an incident). The next stack update may silently undo them, or fail.

**Diagnose →**

```bash
ID=$(aws cloudformation detect-stack-drift --stack-name app --query StackDriftDetectionId --output text)
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id $ID
aws cloudformation describe-stack-resource-drifts --stack-name app --stack-resource-drift-status-filters MODIFIED DELETED \
  --query 'StackResourceDrifts[].[LogicalResourceId,PropertyDifferences]'
```

**Fix →**

1. Decide which version is correct
2. If the console change should stay, put it into the template and deploy a change set (it should show no replacement)
3. If not, deploy the template to overwrite it
4. For resources deleted out of band, remove them from the template, or re-create and **import** them (`create-change-set --change-set-type IMPORT`)

**Prevent →** Run scheduled drift detection (AWS Config rule `cloudformation-stack-drift-detection-check`), make production console access read-only, and have a break-glass process that requires a follow-up PR.

</details>

### Q182. Deleting or updating a stack fails: `Export network-VpcId cannot be deleted as it is in use by app-stack`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another stack imports the output through `Fn::ImportValue`. CloudFormation will not let you change or remove an export while it is imported.

**Diagnose →**

```bash
aws cloudformation list-imports --export-name network-VpcId
aws cloudformation list-exports --query "Exports[?Name=='network-VpcId']"
```

**Fix →**

1. In the importing stacks, replace `ImportValue` with a hard-coded value or an SSM parameter and deploy them
2. Then change or remove the export in the producer stack

**Prevent →** For loose coupling, share values between stacks through SSM Parameter Store (`{{resolve:ssm:/network/vpc-id}}`) rather than exports.

</details>

### Q183. A stack has been in `CREATE_IN_PROGRESS` for an hour on a `Custom::DbSeeder` resource. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The custom resource Lambda never sent a response to the pre-signed `ResponseURL`. It crashed, timed out, could not reach S3 (in a VPC without NAT or an endpoint), or forgot to answer on `Delete`. CloudFormation waits up to the service timeout, which is one hour by default.

**Diagnose →**

```bash
aws cloudformation describe-stack-resource --stack-name app --logical-resource-id DbSeeder
aws logs tail /aws/lambda/app-DbSeeder --since 2h
```

**Fix →**

1. To unblock now, send a response yourself with `curl -X PUT -H 'Content-Type:' --data-binary '{"Status":"FAILURE","Reason":"manual","PhysicalResourceId":"seed","StackId":"...","RequestId":"...","LogicalResourceId":"DbSeeder"}' "<ResponseURL from logs>"`
2. Fix the handler so it always responds in `try/finally`, including on Delete

**Prevent →** Use the `cfn-response` module or CDK Provider framework, and set `ServiceTimeout` on the custom resource to something short (for example, 300 seconds).

</details>

### Q184. A CloudWatch alarm on a custom metric stays in `INSUFFICIENT_DATA` forever. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The alarm's namespace, metric name or dimensions do not match exactly what is published (an alarm must specify **all** dimensions). Other causes: the period is shorter than the publish interval, or the metric stopped being sent.

**Diagnose →**

```bash
aws cloudwatch describe-alarms --alarm-names api-errors \
  --query 'MetricAlarms[].[Namespace,MetricName,Dimensions,Period,StateReason]'
aws cloudwatch list-metrics --namespace MyApp --metric-name Errors
```

**Fix →**

1. Copy the exact dimension set from `list-metrics` into the alarm
2. Make the period match the publishing rate
3. For sparse metrics, set `--treat-missing-data notBreaching`

**Prevent →** Create alarms in the same IaC module that defines the metric names, and test alarms with `set-alarm-state`.

</details>

### Q185. The CloudWatch agent is running, but no `mem_used_percent` or disk metrics appear in the `CWAgent` namespace. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The agent was started with the default or no config, so the `metrics` section is missing. Other causes: the instance role has no `CloudWatchAgentServerPolicy`, or the agent cannot reach the CloudWatch endpoint (private subnet with no `monitoring` endpoint).

**Diagnose →**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
sudo tail -50 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
aws cloudwatch list-metrics --namespace CWAgent --dimensions Name=InstanceId,Value=i-0abc
```

**Fix →**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s -c ssm:AmazonCloudWatch-linux
```

Attach `CloudWatchAgentServerPolicy`, and add the `monitoring` and `logs` endpoints for private subnets.

**Prevent →** Store the agent config in SSM Parameter Store, and push it with State Manager to every instance.

</details>

### Q186. An app writes logs, but its CloudWatch log group stays empty. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role cannot write logs (`logs:CreateLogStream`, `logs:PutLogEvents`, or `logs:CreateLogGroup` if the group does not exist). The log group is KMS-encrypted and the key policy does not allow the `logs.<region>.amazonaws.com` principal. Or you are looking in the wrong Region or group name.

**Diagnose →**

```bash
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/orders --query 'logGroups[].[logGroupName,kmsKeyId]'
aws logs describe-log-streams --log-group-name /aws/lambda/orders --order-by LastEventTime --descending --max-items 3
aws iam list-attached-role-policies --role-name orders-lambda-role
```

**Fix →**

1. For Lambda, attach `AWSLambdaBasicExecutionRole`. For ECS, give the execution role `logs:*` on the group
2. Create the group in IaC with retention set
3. Fix the KMS key policy for CloudWatch Logs

**Prevent →** Pre-create every log group with retention and permissions, rather than relying on auto-create, which also means infinite retention.

</details>

### Q187. A CPU alarm flaps between `OK` and `ALARM` every few minutes and pages the on-call engineer all night. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The alarm uses 1 datapoint of 1 over a short period on a noisy metric. Or missing data is treated as breaching while the metric is published irregularly.

**Diagnose →**

```bash
aws cloudwatch describe-alarm-history --alarm-name web-cpu --history-item-type StateUpdate --max-items 20 \
  --query 'AlarmHistoryItems[].[Timestamp,HistorySummary]'
aws cloudwatch describe-alarms --alarm-names web-cpu \
  --query 'MetricAlarms[].[Period,EvaluationPeriods,DatapointsToAlarm,TreatMissingData,Threshold]'
```

**Fix →**

```bash
aws cloudwatch put-metric-alarm --alarm-name web-cpu --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=web-asg --statistic Average --period 60 \
  --evaluation-periods 5 --datapoints-to-alarm 4 --threshold 85 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching --alarm-actions <sns-arn>
```

Use composite alarms (for example, high CPU **and** high latency) for paging.

**Prevent →** Page on symptoms (SLO burn, errors, latency) and send resource alarms to tickets.

</details>

### Q188. Decrypt calls fail with `KMSInvalidStateException: arn:aws:kms:...:key/... is pending deletion.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone scheduled the key for deletion (7–30 day waiting period). While pending, the key cannot be used, and everything encrypted with it stops working.

**Diagnose →**

```bash
aws kms describe-key --key-id <key-id> --query 'KeyMetadata.[KeyState,DeletionDate]'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ScheduleKeyDeletion --max-results 5
```

**Fix →**

```bash
aws kms cancel-key-deletion --key-id <key-id>
aws kms enable-key --key-id <key-id>      # cancel leaves it Disabled
```

**Prevent →** Deny `kms:ScheduleKeyDeletion` through SCP except for a break-glass role, alarm on that CloudTrail event through EventBridge, and use the 30-day window.

</details>

### Q189. A cross-account decrypt fails: `AccessDeniedException: The ciphertext refers to a customer master key that does not exist, does not exist in this region, or you are not allowed to access it.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One of three: (1) the key is in another Region, because KMS keys are Regional unless multi-Region, (2) the key policy in the owning account does not allow the caller's account or role, or (3) the caller's IAM policy has no `kms:Decrypt` on the key ARN. Cross-account access needs **both** the key policy and IAM. The error deliberately does not say which one failed.

**Diagnose →**

```bash
aws kms describe-key --key-id arn:aws:kms:us-east-1:222222222222:key/abcd... --query 'KeyMetadata.[Arn,KeyState,MultiRegion]'
aws kms get-key-policy --key-id <arn> --policy-name default --output text   # run in owning account
aws iam simulate-principal-policy --policy-source-arn <caller-role-arn> --action-names kms:Decrypt --resource-arns <key-arn>
```

**Fix →**

1. Add a key policy statement allowing `arn:aws:iam::111111111111:role/consumer` to use `kms:Decrypt` and `kms:DescribeKey`
2. Add a matching IAM allow on the consumer role, referencing the **key ARN** (not an alias)
3. Call KMS in the key's Region, or use a multi-Region replica

**Prevent →** Template cross-account key grants, and use encryption context conditions to scope them.

</details>

### Q190. Under load, S3 and SSM calls start failing with `ThrottlingException: You have exceeded the rate at which you may call KMS. Reduce the frequency of your calls.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every SSE-KMS object read or write, every SecureString parameter read and every envelope encryption call hits KMS, and they share a per-account, per-Region request quota for cryptographic operations (thousands of requests per second, depending on the Region).

**Diagnose →**

```bash
aws service-quotas list-service-quotas --service-code kms \
  --query "Quotas[?contains(QuotaName,'Cryptographic operations')].[QuotaName,Value]"
# CloudWatch AWS/Usage: Service=KMS, Type=API, Resource=Decrypt/GenerateDataKey
aws s3api get-bucket-encryption --bucket data --query 'ServerSideEncryptionConfiguration.Rules[].BucketKeyEnabled'
```

**Fix →**

1. Turn on **S3 Bucket Keys**, which cut KMS calls from S3 by up to 99%: `aws s3api put-bucket-encryption --bucket data --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms","KMSMasterKeyID":"<key>"},"BucketKeyEnabled":true}]}'`
2. Cache decrypted secrets and parameters in the app (Secrets Manager caching client, Parameters and Secrets Lambda extension)
3. Use data key caching (AWS Encryption SDK)
4. Request a quota increase

**Prevent →** Alarm on KMS usage at 70% of the quota, and design for caching from the start.

</details>

### Q191. `ResourceNotFoundException: Secrets Manager can't find the specified secret.` but the secret is in the console. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The call goes to the wrong Region or account. Or the code uses a partial ARN: Secrets Manager adds a random 6-character suffix to ARNs, so `arn:...:secret:prod/db` without the suffix fails, while the name `prod/db` works. The secret may also be scheduled for deletion.

**Diagnose →**

```bash
aws secretsmanager describe-secret --secret-id prod/db --region us-east-1 --query '[ARN,DeletedDate]'
aws secretsmanager list-secrets --filters Key=name,Values=prod/ --region us-east-1 --query 'SecretList[].Name'
```

**Fix →**

1. Reference the secret by name, or by full ARN including the suffix
2. Set the Region explicitly
3. If the secret is pending deletion: `aws secretsmanager restore-secret --secret-id prod/db`

**Prevent →** Pass secret ARNs from IaC outputs, and in IAM policies use `secret:prod/db-??????` to match the suffix.

</details>

### Q192. Secrets Manager rotation fails, and the secret shows `AWSPENDING` stuck on a version. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The rotation Lambda failed in one of its four steps (`createSecret`, `setSecret`, `testSecret`, `finishSecret`). Common reasons: it cannot reach the DB (VPC or SG), it cannot reach the Secrets Manager endpoint from a private subnet, the DB user has no rights to change its password, or a previous failed rotation left `AWSPENDING` behind, which blocks the next try.

**Diagnose →**

```bash
aws secretsmanager describe-secret --secret-id prod/db --query '[RotationEnabled,LastRotatedDate,VersionIdsToStages]'
aws logs tail /aws/lambda/SecretsManagerRDSPostgreSQLRotationSingleUser --since 2h
```

**Fix →**

1. Fix the network path: the Lambda must reach both the DB and the `secretsmanager` endpoint
2. Remove the stale stage: `aws secretsmanager update-secret-version-stage --secret-id prod/db --version-stage AWSPENDING --remove-from-version-id <id>`
3. Retry: `aws secretsmanager rotate-secret --secret-id prod/db`

**Prevent →** Prefer RDS managed master passwords, use the alternating-users strategy for apps, and alarm on rotation failures through EventBridge (`RotationFailed`).

</details>

### Q193. `aws ssm get-parameter --name /prod/db/password` returns an encrypted blob instead of the password. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The parameter is a `SecureString`, and the call did not ask for decryption. If you add the flag and get `AccessDeniedException`, the caller has no `kms:Decrypt` on the key.

**Diagnose →**

```bash
aws ssm describe-parameters --parameter-filters Key=Name,Values=/prod/db/password --query 'Parameters[].[Type,KeyId]'
```

**Fix →**

```bash
aws ssm get-parameter --name /prod/db/password --with-decryption --query Parameter.Value --output text
aws ssm get-parameters-by-path --path /prod/db --recursive --with-decryption
```

Grant `ssm:GetParameter*` on `arn:aws:ssm:...:parameter/prod/db/*` and `kms:Decrypt` on the key.

**Prevent →** Scope parameter access by path per service, and use `ParameterNotFound` checks in startup code so missing config fails fast.

</details>

### Q194. Re-creating a secret fails: `InvalidRequestException: You can't create this secret because a secret with this name is already scheduled for deletion.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A secret with that name was deleted with a recovery window (7–30 days) and is still recoverable. This often happens in IaC teardown and re-create cycles.

**Diagnose →**

```bash
aws secretsmanager describe-secret --secret-id dev/api-key --query '[Name,DeletedDate]'
```

**Fix →**

1. To reuse it, restore it: `aws secretsmanager restore-secret --secret-id dev/api-key`
2. For dev only, force deletion now: `aws secretsmanager delete-secret --secret-id dev/api-key --force-delete-without-recovery`

**Prevent →** For ephemeral environments, set `recovery_window_in_days = 0` in Terraform, and keep a 30-day window in production.

</details>

### Q195. A production security group rule disappeared overnight. How do you find out who removed it? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A manual console change, an automation job, or an IaC apply that removed the rule as drift.

**Diagnose →**

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=sg-0abc \
  --start-time $(date -u -d '-2 day' +%FT%TZ) \
  --query 'Events[].[EventTime,EventName,Username]' --output table
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RevokeSecurityGroupIngress --max-results 10
aws configservice get-resource-config-history --resource-type AWS::EC2::SecurityGroup --resource-id sg-0abc --limit 5
```

**Fix →**

1. Restore the rule through IaC
2. Find the principal (`userIdentity.arn`, `sourceIPAddress`, user agent such as Terraform) and close the gap

**Prevent →** Keep an organization trail with Athena queries, AWS Config rules for SG baselines, and EventBridge alerts on SG changes in production. Note that `lookup-events` covers only 90 days of management events.

</details>

### Q196. The NAT Gateway line is the second-biggest item on the AWS bill. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NAT charges per GB processed as well as per hour. Big traffic through NAT, such as pulling container images, S3 or DynamoDB transfers, or backups, adds up quickly. Cross-AZ NAT usage adds data transfer charges on top.

**Diagnose →**

```bash
aws ce get-cost-and-usage --time-period Start=2026-10-01,End=2026-11-01 --granularity MONTHLY --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"USAGE_TYPE_GROUP","Values":["EC2: NAT Gateway - Data Processed"]}}'
# CloudWatch AWS/NATGateway BytesOutToDestination per NAT
# VPC Flow Logs on the NAT ENI -> top destinations by bytes (Athena)
```

**Fix →**

1. Add free **gateway endpoints** for S3 and DynamoDB
2. Add interface endpoints for ECR, Logs and STS where the volume justifies their hourly cost
3. Use one NAT per AZ with AZ-local routes
4. Cache images (ECR pull-through cache)

**Prevent →** Tag and dashboard NAT bytes, and review the top talkers through flow logs every month.

</details>

### Q197. Data transfer costs jumped after you spread services across three AZs. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Inter-AZ traffic is charged in both directions (about $0.01/GB each way). Chatty microservices, Kafka or Elasticsearch replication, and cross-AZ load balancing to targets (on ALB it is free inside the LB; on NLB with cross-zone on, you pay inter-AZ) add up. Calling through public IPs or Elastic IPs inside the Region costs even more.

**Diagnose →**

```bash
aws ce get-cost-and-usage --time-period Start=2026-10-01,End=2026-11-01 --granularity DAILY \
  --metrics UnblendedCost --group-by Type=DIMENSION,Key=USAGE_TYPE \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute","EC2 - Other"]}}' \
  --query 'ResultsByTime[-1].Groups[?contains(Keys[0],`DataTransfer-Regional`)]'
# Flow logs with az-id fields to find top cross-AZ pairs
```

**Fix →**

1. Use topology-aware routing (Kubernetes `trafficDistribution: PreferClose`, or Istio locality LB)
2. Use rack- or AZ-aware consumers (Kafka `client.rack`)
3. Use private IPs, not public ones, for in-Region calls

**Prevent →** Make data transfer a design review item, and put cross-AZ bytes on a dashboard per service.

</details>

### Q198. A budget alert fired at 120% of forecast mid-month. How do you find what is spending money? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A new or forgotten resource (a big instance, a GPU node pool, a NAT), runaway usage (Lambda loop, logs ingestion), or a missing commitment discount.

**Diagnose →**

```bash
aws ce get-cost-and-usage --time-period Start=2026-11-01,End=2026-11-09 --granularity DAILY \
  --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[-1].Groups | sort_by(@,&to_number(Metrics.UnblendedCost.Amount))[-5:]'
aws ce get-cost-and-usage --time-period Start=2026-11-01,End=2026-11-09 --granularity DAILY \
  --metrics UnblendedCost --group-by Type=DIMENSION,Key=LINKED_ACCOUNT
aws ce get-anomalies --date-interval StartDate=2026-11-01,EndDate=2026-11-09
```

**Fix →**

1. Drill down from service to usage type, then account, then tag (`Type=TAG,Key=team`)
2. Stop or right-size the offender
3. Fix the runaway trigger

**Prevent →** Turn on Cost Anomaly Detection with alerts to a channel, use per-team budgets with actions, and enforce cost allocation tags.

</details>

### Q199. All dev instances are stopped every night, yet the account still costs money each day. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Stopped instances still have charged **EBS volumes**. Other charges continue too: snapshots, Elastic IPs and all public IPv4 addresses (charged hourly since February 2024), NAT gateways, load balancers, RDS instances (which also restart automatically after 7 days stopped) and CloudWatch Logs storage.

**Diagnose →**

```bash
aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].[VolumeId,Size,CreateTime]'
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]'
aws ec2 describe-nat-gateways --filter Name=state,Values=available --query 'NatGateways[].NatGatewayId'
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'
```

**Fix →**

1. Delete unattached volumes and unused EIPs
2. Expire old snapshots with Data Lifecycle Manager
3. Remove idle NATs and ALBs in dev (recreate them with IaC when needed)

**Prevent →** Run ephemeral dev environments from IaC that are destroyed nightly, and use Trusted Advisor or Compute Optimizer idle-resource checks.

</details>

### Q200. You bought a Compute Savings Plan last month, but Cost Explorer shows only 40% coverage and costs barely moved. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The commitment covers less spend than you thought (the commitment is $/hour of **discounted** usage). Other reasons: most spend is on services the plan does not cover (RDS, OpenSearch, ElastiCache and Redshift are not covered by Compute Savings Plans and need their own Reserved Instances), usage is in member accounts with discount sharing turned off, or you bought an **EC2 Instance** Savings Plan locked to a family and Region you then moved away from.

**Diagnose →**

```bash
aws ce get-savings-plans-coverage --time-period Start=2026-10-01,End=2026-11-01 --granularity MONTHLY
aws ce get-savings-plans-utilization --time-period Start=2026-10-01,End=2026-11-01
aws savingsplans describe-savings-plans --query 'savingsPlans[].[savingsPlanType,commitment,region,ec2InstanceFamily,state]'
aws ce get-savings-plans-purchase-recommendation --savings-plans-type COMPUTE_SP --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT --lookback-period-in-days THIRTY_DAYS
```

**Fix →**

1. If utilization is low, move workloads onto the covered family or Region, or switch to Compute SP for new purchases
2. If coverage is low, layer extra commitments based on the recommendation, plus RIs for databases
3. Turn on RI/SP discount sharing in the payer account

**Prevent →** Buy in small tranches every quarter based on a steady-state baseline, prefer Compute SP for flexibility, and alarm on utilization below 95%.

</details>

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
