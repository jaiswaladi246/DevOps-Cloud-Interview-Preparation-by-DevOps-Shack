# AWS for DevOps — Assignments

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

> 25 hands-on assignments in three levels: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Do them in a sandbox account, tag everything, and delete what you create.

[⬅ Back to AWS for DevOps](./README.md) · [🏠 Home](../README.md)

## How to use these

Use the AWS CLI for every task, even where the console is easier; the CLI is what you will script and automate. Set a billing budget before you begin, work in one Region (`ap-south-1` in our examples), and tag every resource with `Project`, `Environment` and `Owner` so cleanup is a single query. Hints are collapsed so you can try first.

## Table of contents

- 🟢 [Beginner](#-beginner) — Assignments 1–10
- 🟡 [Intermediate](#-intermediate) — Assignments 11–20
- 🔴 [Advanced](#-advanced) — Assignments 21–25

---

## 🟢 Beginner

### Assignment 1. Secure a new account 🟢

**Objective:** Put the basic guardrails on a fresh AWS account before building anything.

**Tasks**
1. Enable MFA on the root user and stop using root for daily work.
2. Create an admin identity through IAM Identity Center, or an IAM user with MFA if Identity Center is unavailable.
3. Configure the CLI with a named profile and confirm with `aws sts get-caller-identity`.
4. Create a monthly cost budget of 10 USD that emails you at 80 percent.

**Expected outcome:** Root has MFA and no access keys, you work as a non-root identity, and a budget exists.

<details>
<summary>Hints</summary>

`aws configure sso` sets up an SSO profile. `aws iam list-access-keys` for root is checked in the console security credentials page. `aws budgets create-budget` takes a `--budget` JSON and `--notifications-with-subscribers`.

</details>

**Bonus:** Turn on a multi-Region CloudTrail trail delivering to an S3 bucket.

### Assignment 2. Build a VPC by hand 🟢

**Objective:** Understand what makes a subnet public or private.

**Tasks**
1. Create a VPC `10.20.0.0/16` with two public and two private subnets across two AZs.
2. Attach an internet gateway and create a public route table with a default route to it.
3. Associate only the public subnets with that route table.
4. Enable auto-assign public IP on the public subnets only.

**Expected outcome:** `describe-route-tables` shows exactly the two public subnets routed to `igw-`, and the private subnets have only the local route.

<details>
<summary>Hints</summary>

`aws ec2 create-vpc`, `create-subnet --availability-zone`, `create-internet-gateway`, `attach-internet-gateway`, `create-route --destination-cidr-block 0.0.0.0/0`, `modify-subnet-attribute --map-public-ip-on-launch`.

</details>

**Bonus:** Add a NAT gateway and a private route table, then explain its hourly cost.

### Assignment 3. EC2 without SSH 🟢

**Objective:** Manage an instance with Session Manager and an instance profile instead of key pairs.

**Tasks**
1. Create an IAM role trusted by `ec2.amazonaws.com` with `AmazonSSMManagedInstanceCore` attached, and an instance profile.
2. Launch a `t3.micro` Amazon Linux 2023 instance with that profile and no key pair, with IMDSv2 required.
3. Open a session with `aws ssm start-session`.
4. From inside the instance, query metadata using IMDSv2 and print the instance ID.

**Expected outcome:** A working shell with no inbound ports open and no key pair on the instance.

<details>
<summary>Hints</summary>

Get the AMI from the SSM parameter `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`. The instance needs an outbound path (public IP or NAT) or VPC endpoints to register. IMDSv2 requires a PUT to `/latest/api/token` first. Use `--metadata-options HttpTokens=required`.

</details>

**Bonus:** Add a CloudWatch Logs session log destination in Session Manager preferences.

### Assignment 4. Chained security groups 🟢

**Objective:** Build tier isolation with security group references.

**Tasks**
1. Create `web-sg` (80 from anywhere), `app-sg` (8080 from `web-sg`) and `db-sg` (5432 from `app-sg`).
2. Launch one instance in each of web and app tiers.
3. Prove web can reach app on 8080 and that the internet cannot.
4. Prove web cannot reach the database port directly.

**Expected outcome:** Reachability matches the diagram and you can show it with `curl` or `nc -zv` from the right places.

<details>
<summary>Hints</summary>

Use `--ip-permissions "IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId=sg-xxxx}]"`. Start a test listener with `python3 -m http.server 8080`.

</details>

**Bonus:** Explain why a stateless NACL would need extra ephemeral port rules for the same design.

### Assignment 5. A properly locked-down S3 bucket 🟢

**Objective:** Create a private, versioned, encrypted bucket with a lifecycle policy.

**Tasks**
1. Create a bucket with a globally unique name and turn on all four Block Public Access settings.
2. Enable versioning and default encryption.
3. Add a lifecycle rule moving objects to Standard-IA after 30 days and expiring noncurrent versions after 90 days.
4. Upload a file twice with different content and list both versions.

**Expected outcome:** `get-public-access-block`, `get-bucket-versioning` and `get-bucket-lifecycle-configuration` show the expected settings, and `list-object-versions` shows two versions.

<details>
<summary>Hints</summary>

`aws s3api put-bucket-versioning --versioning-configuration Status=Enabled`. Lifecycle JSON uses `Transitions` with `StorageClass: STANDARD_IA` and `NoncurrentVersionExpiration` with `NoncurrentDays`. A bucket outside `us-east-1` needs `--create-bucket-configuration LocationConstraint=<region>`.

</details>

**Bonus:** Add a bucket policy that denies any request not using TLS (`aws:SecureTransport` is false).

### Assignment 6. Static site on S3 and CloudFront 🟢

**Objective:** Serve a private bucket through CloudFront with Origin Access Control.

**Tasks**
1. Upload an `index.html` to a private bucket.
2. Create an OAC and a distribution with `DefaultRootObject` set to `index.html`.
3. Add a bucket policy that allows only your distribution's ARN to `s3:GetObject`.
4. Confirm the CloudFront URL returns 200 and the direct S3 URL returns 403.

**Expected outcome:** Content is reachable only through CloudFront over HTTPS.

<details>
<summary>Hints</summary>

The policy principal is `{"Service":"cloudfront.amazonaws.com"}` with a condition `StringEquals` on `AWS:SourceArn`. Use `aws cloudfront create-distribution --distribution-config file://...`. Distribution deployment takes several minutes.

</details>

**Bonus:** Update the file, observe the stale cache, and fix it with `aws cloudfront create-invalidation --paths "/index.html"`.

### Assignment 7. Least-privilege S3 policy 🟢

**Objective:** Write a policy that grants one action on one prefix and prove it.

**Tasks**
1. Create an IAM role that can `s3:GetObject` only on `arn:aws:s3:::<bucket>/reports/*` and `s3:ListBucket` only with prefix `reports/`.
2. Use `aws iam simulate-principal-policy` to test allowed and denied actions.
3. Assume the role and try to read an object outside the prefix.
4. Record the exact `AccessDenied` message.

**Expected outcome:** Reads under `reports/` succeed, everything else is denied.

<details>
<summary>Hints</summary>

`s3:ListBucket` applies to the bucket ARN and needs a `Condition` with `StringLike` on `s3:prefix`. `GetObject` applies to the object ARN with `/*`. Use `aws sts assume-role` and export the three temporary credential variables.

</details>

**Bonus:** Generate a policy from CloudTrail activity with IAM Access Analyzer policy generation.

### Assignment 8. CPU alarm with SNS 🟢

**Objective:** Get notified when an instance is hot.

**Tasks**
1. Create an SNS topic and subscribe your email, then confirm the subscription.
2. Create a CloudWatch alarm on `CPUUtilization` for one instance, threshold 70 percent for 2 of 3 periods of 60 seconds.
3. Generate load with `stress-ng` or `yes > /dev/null`.
4. Receive the email and clear the alarm by stopping the load.

**Expected outcome:** An `ALARM` email followed by an `OK` email.

<details>
<summary>Hints</summary>

`aws cloudwatch put-metric-alarm --datapoints-to-alarm 2 --evaluation-periods 3`. Basic monitoring is 5 minute resolution, so enable detailed monitoring with `aws ec2 monitor-instances` for 1 minute data. Test the path without load using `set-alarm-state`.

</details>

**Bonus:** Add an EC2 recover action for status check failures: `arn:aws:automate:<region>:ec2:recover`.

### Assignment 9. ALB in front of two instances 🟢

**Objective:** Load balance across AZs and watch health checks work.

**Tasks**
1. Launch two instances in different AZs running nginx that prints their hostname.
2. Create an ALB, a target group with a `/health` check, and an HTTP listener.
3. Register both instances and loop `curl` to see responses alternate.
4. Stop nginx on one instance and watch it leave rotation.

**Expected outcome:** Traffic alternates, then goes only to the healthy instance, and `describe-target-health` shows the failure reason.

<details>
<summary>Hints</summary>

`aws elbv2 create-target-group`, `register-targets`, `create-listener`. The ALB needs two subnets in two AZs. The instance SG must allow port 80 from the ALB SG. Health transitions follow interval times threshold.

</details>

**Bonus:** Add a path-based rule that sends `/api/*` to a second target group.

### Assignment 10. Tagging audit 🟢

**Objective:** Define a tag standard and find what violates it.

**Tasks**
1. Define required tags: `Project`, `Environment`, `Owner`, `CostCenter`.
2. Create five resources of different types with and without the tags.
3. Use the Resource Groups Tagging API to list resources missing a required tag.
4. Write a short script that prints each untagged resource ARN.

**Expected outcome:** A repeatable command or script that lists noncompliant resources.

<details>
<summary>Hints</summary>

`aws resourcegroupstaggingapi get-resources` returns tags per resource; filter with `--query` or `jq` for entries lacking a key. The API only returns resources that have, or once had, tags, so completely untagged resources can be missed; cross-check with service calls such as `aws ec2 describe-instances`.

</details>

**Bonus:** Create an AWS Config managed rule `required-tags` to flag violations automatically.

---

## 🟡 Intermediate

### Assignment 11. Auto Scaling with load test 🟡

**Objective:** Watch a target tracking policy scale out and in.

**Tasks**
1. Create a launch template with user data that installs nginx.
2. Create an ASG (min 1, max 4) across two subnets behind an ALB.
3. Add target tracking on `ALBRequestCountPerTarget` with a target of 50.
4. Drive load with `hey -z 5m -c 50 http://<alb>/` and record scaling activities.

**Expected outcome:** Instance count rises under load, then falls after the cooldown, with timestamps captured from `describe-scaling-activities`.

<details>
<summary>Hints</summary>

For `ALBRequestCountPerTarget` the `PredefinedMetricSpecification` needs a `ResourceLabel` of the form `app/<alb-name>/<id>/targetgroup/<tg-name>/<id>`. Scale-in is deliberately slower than scale-out.

</details>

**Bonus:** Add a scheduled action that raises the minimum to 3 during business hours.

### Assignment 12. Private RDS with a tunnel 🟡

**Objective:** Reach a private database without a public IP or a bastion host.

**Tasks**
1. Create an RDS PostgreSQL instance in private subnets with `--no-publicly-accessible` and a managed master password.
2. Launch an SSM-managed instance in the app subnet.
3. Open a port-forward from your laptop with `AWS-StartPortForwardingSessionToRemoteHost`.
4. Connect with `psql` to `localhost` and run a query.

**Expected outcome:** A working local `psql` session through the tunnel and no inbound SG rule from the internet.

<details>
<summary>Hints</summary>

`aws ssm start-session --target i-... --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters '{"host":["<rds-endpoint>"],"portNumber":["5432"],"localPortNumber":["5433"]}'`. Retrieve the password from the secret ARN shown in `describe-db-instances`.

</details>

**Bonus:** Enforce `rds.force_ssl` through a custom parameter group and prove non-TLS connections are rejected.

### Assignment 13. Multi-AZ failover drill 🟡

**Objective:** Measure what a failover does to a client.

**Tasks**
1. Use a Multi-AZ instance from Assignment 12.
2. Run a loop that executes `select 1` every second and logs success or failure with timestamps.
3. Trigger `reboot-db-instance --force-failover`.
4. Compute downtime from the log and compare the primary AZ before and after.

**Expected outcome:** A downtime figure (typically 60 to 120 seconds), a swapped AZ, and a note on what your client should do.

<details>
<summary>Hints</summary>

Use `psql -c` inside `while true` with `date -u` prefixed output. The endpoint hostname does not change; the DNS record behind it does. Set short connection timeouts such as `connect_timeout=3`.

</details>

**Bonus:** Repeat with Aurora and compare failover time.

### Assignment 14. Cross-account role assumption 🟡

**Objective:** Grant access across accounts without sharing keys.

**Tasks**
1. In account B, create a role with a trust policy for account A's principal and an `sts:ExternalId` condition.
2. In account A, allow a role or user to call `sts:AssumeRole` on it.
3. Assume it with the CLI and verify with `get-caller-identity`.
4. Add `aws:MultiFactorAuthPresent` to the trust policy and observe the failure without MFA.

**Expected outcome:** Role assumption succeeds only with the correct external ID and, for step 4, MFA.

<details>
<summary>Hints</summary>

`aws sts assume-role --role-arn ... --role-session-name lab --external-id ...`. A named profile with `role_arn`, `source_profile` and `external_id` automates it. Both sides must allow it: trust policy in B and identity policy in A.

</details>

**Bonus:** Restrict the trust to your AWS Organization with `aws:PrincipalOrgID`.

### Assignment 15. VPC endpoints instead of NAT 🟡

**Objective:** Remove internet dependence for AWS API traffic.

**Tasks**
1. Start with private instances that reach S3 and SSM through a NAT gateway.
2. Add an S3 gateway endpoint and attach it to the private route table.
3. Add interface endpoints for `ssm`, `ssmmessages` and `ec2messages`.
4. Delete the NAT default route and confirm Session Manager and S3 still work.

**Expected outcome:** Private instances operate with no internet path, and you can explain the cost of interface endpoints per AZ per hour against NAT.

<details>
<summary>Hints</summary>

Interface endpoints need `--private-dns-enabled` and a security group allowing 443 from the instances. Gateway endpoints add a prefix-list route to the route table. `aws ec2 describe-vpc-endpoints` shows state.

</details>

**Bonus:** Add an endpoint policy that restricts the S3 endpoint to your own buckets.

### Assignment 16. Route 53 failover routing 🟡

**Objective:** Fail DNS over to a backup when the primary is unhealthy.

**Tasks**
1. Create a health check against your primary endpoint `/health`.
2. Create PRIMARY and SECONDARY failover records, the secondary pointing to a static S3 maintenance site or a second ALB.
3. Break the primary and watch DNS answers change.
4. Restore the primary and confirm it fails back.

**Expected outcome:** `dig` returns the secondary after the health check fails, with the TTL you chose bounding recovery time.

<details>
<summary>Hints</summary>

`aws route53 create-health-check --caller-reference ... --health-check-config Type=HTTP,FullyQualifiedDomainName=...,ResourcePath=/health,RequestInterval=10,FailureThreshold=2`. Records take `Failover=PRIMARY`, `SetIdentifier` and `HealthCheckId`. Use a low TTL for the lab.

</details>

**Bonus:** Replace failover with weighted records and do a 90/10 canary.

### Assignment 17. HTTPS with ACM 🟡

**Objective:** Terminate TLS on the ALB and redirect HTTP.

**Tasks**
1. Request an ACM certificate for your domain with DNS validation.
2. Create the validation CNAME in Route 53 and wait for `ISSUED`.
3. Add an HTTPS listener on 443 with `ELBSecurityPolicy-TLS13-1-2-2021-06`.
4. Change the port 80 listener to a permanent redirect to 443.

**Expected outcome:** `curl -I http://domain` returns 301 to HTTPS and the HTTPS site serves with a valid certificate.

<details>
<summary>Hints</summary>

`aws acm request-certificate --domain-name ... --validation-method DNS`, then `describe-certificate` for the validation record. Redirect action: `Type=redirect,RedirectConfig={Protocol=HTTPS,Port=443,StatusCode=HTTP_301}`. CloudFront certificates must be in `us-east-1`; ALB certificates must be in the ALB's Region.

</details>

**Bonus:** Add an ALB rule that returns a fixed 403 for requests missing a secret header, and have CloudFront add it.

### Assignment 18. Investigate with CloudTrail 🟡

**Objective:** Answer "who did this and when" from audit logs.

**Tasks**
1. Ensure a trail logs management events to S3.
2. Delete a security group rule or a tagged resource on purpose.
3. Find the event with `aws cloudtrail lookup-events` and extract user, source IP and request parameters.
4. Create an EventBridge rule that notifies SNS on `DeleteSecurityGroup`.

**Expected outcome:** You can name the principal, time and parameters of the change, and a new deletion triggers a notification.

<details>
<summary>Hints</summary>

`aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RevokeSecurityGroupIngress`. Lookup covers 90 days of management events. EventBridge pattern: `{"source":["aws.ec2"],"detail-type":["AWS API Call via CloudTrail"],"detail":{"eventName":["DeleteSecurityGroup"]}}`.

</details>

**Bonus:** Query the S3 trail logs with Athena for all `ConsoleLogin` events without MFA.

### Assignment 19. Logs to metrics to alarm 🟡

**Objective:** Turn application log lines into an actionable alert.

**Tasks**
1. Install the CloudWatch agent on an instance and ship `/var/log/nginx/access.log` and `error.log` to log groups with 14-day retention.
2. Create a metric filter that counts `5xx` responses.
3. Create an alarm on that metric that publishes to SNS.
4. Generate 500s and receive the alert.

**Expected outcome:** Logs in CloudWatch, a custom metric on a graph, and an email when errors spike.

<details>
<summary>Hints</summary>

`dnf install -y amazon-cloudwatch-agent`, then `amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:<parameter-name> -s`. The instance role needs `CloudWatchAgentServerPolicy`. Metric filter pattern for a combined access log can use `[ip, id, user, ts, req, status=5*, size]`. `aws logs put-retention-policy` sets retention.

</details>

**Bonus:** Query the same logs with CloudWatch Logs Insights for the top 10 failing URLs.

### Assignment 20. Secrets without baking them into the image 🟡

**Objective:** Deliver configuration and secrets to instances securely.

**Tasks**
1. Store a non-secret setting in SSM Parameter Store and a secret in Secrets Manager.
2. Give an instance role access to only those two ARNs.
3. Read both at boot in user data or a systemd unit.
4. Rotate the secret and show the app picking up the new value without a redeploy.

**Expected outcome:** No secret in the AMI, user data or environment files at rest, and a documented rotation procedure.

<details>
<summary>Hints</summary>

`aws ssm get-parameter --name /app/prod/log-level --with-decryption`, `aws secretsmanager get-secret-value --secret-id`. Anything in user data is visible through `describe-instance-attribute`, so never place secrets there. For rotation use `aws secretsmanager rotate-secret` with a rotation Lambda, or `put-secret-value` manually for the lab.

</details>

**Bonus:** Compare monthly cost and features of Parameter Store SecureString and Secrets Manager.

---

## 🔴 Advanced

### Assignment 21. Rebuild the lab as Terraform 🔴

**Objective:** Express the whole HA stack as code with safe state handling.

**Tasks**
1. Recreate the VPC, ALB, ASG, RDS Multi-AZ, S3, CloudFront and IAM from [the lab](./hands-on-lab.md) in Terraform modules.
2. Use an S3 backend with state locking and encryption.
3. Introduce drift by changing a security group rule in the console, then detect it with `terraform plan`.
4. Add a CI job that runs `fmt`, `validate`, `plan` on pull requests and `apply` on merge after approval.

**Expected outcome:** `terraform apply` builds the stack from zero, `plan` is clean afterwards, and drift is caught.

<details>
<summary>Hints</summary>

Use `default_tags` in the AWS provider for cost tags. Use `use_lockfile = true` in the S3 backend on current Terraform versions, or a DynamoDB table on older ones. Use `manage_master_user_password = true` on `aws_db_instance`. Put `lifecycle { create_before_destroy = true }` on launch-template driven resources.

</details>

**Bonus:** Add `checkov` or `tfsec` to the pipeline and fix the findings.

### Assignment 22. Multi-Region pilot light 🔴

**Objective:** Build and test a disaster recovery plan with a measured RTO and RPO.

**Tasks**
1. Replicate the assets bucket to a second Region with Cross-Region Replication.
2. Create a cross-Region RDS read replica.
3. Keep the secondary stack scaled to zero (ASG desired 0, launch template ready).
4. Script the failover: promote the replica, scale the ASG, flip Route 53 failover records.
5. Run the drill, record the time for each phase and the replication lag at cutover.

**Expected outcome:** A runbook with measured RTO and RPO, and evidence it works.

<details>
<summary>Hints</summary>

CRR needs versioning on both buckets and a replication role. `aws rds create-db-instance-read-replica --source-db-instance-identifier <arn> --region <dr-region>` (encrypted sources need `--kms-key-id` in the target Region). `aws rds promote-read-replica` is irreversible. AMIs must be copied with `aws ec2 copy-image`. Watch `ReplicaLag` in CloudWatch.

</details>

**Bonus:** Automate the failover with a Step Functions workflow triggered by a manual approval.

### Assignment 23. Blue/green with automatic rollback 🔴

**Objective:** Shift traffic between two versions with an alarm as the safety net.

**Tasks**
1. Create two target groups (blue and green) with separate ASGs.
2. Use a listener rule with weighted forwarding to move traffic 10, 50, 100 percent.
3. Watch the 5xx and latency alarms between each step.
4. Write a script that rolls back to 100 percent blue if any alarm enters `ALARM`.

**Expected outcome:** A promotion that completes cleanly for a good build and rolls back automatically for a bad one.

<details>
<summary>Hints</summary>

`aws elbv2 modify-listener --default-actions` with `ForwardConfig` and `TargetGroups=[{TargetGroupArn=...,Weight=...}]`. Use `aws cloudwatch describe-alarms --state-value ALARM --alarm-names ...` in the loop. Enable target group stickiness only if you need it, as it skews weights.

</details>

**Bonus:** Implement the same flow with AWS CodeDeploy and its automatic rollback on alarms.

### Assignment 24. Organization guardrails 🔴

**Objective:** Enforce security rules that cannot be bypassed from inside a member account.

**Tasks**
1. Create an Organizational Unit and a member account (or use a sandbox OU).
2. Write an SCP that denies actions outside two approved Regions, except for global services.
3. Write an SCP that denies disabling CloudTrail and deleting the S3 Block Public Access settings.
4. Apply a permission boundary to a developer role so it cannot create IAM roles with more rights than itself.
5. Prove each control by attempting the forbidden action as an administrator in the member account.

**Expected outcome:** Admin users in the member account receive explicit denies, with the exact error messages recorded.

<details>
<summary>Hints</summary>

SCPs never grant permissions, they cap them, and they do not affect the management account. Region denial uses `aws:RequestedRegion` with `NotAction` for global services such as `iam:*`, `sts:*`, `cloudfront:*`, `route53:*`, `support:*`. For privilege escalation protection, require `iam:PermissionsBoundary` on `iam:CreateRole` and `iam:PutRolePermissionsBoundary`. Test with `aws iam simulate-principal-policy` where possible.

</details>

**Bonus:** Turn on IAM Access Analyzer for the organization and review the external access findings.

### Assignment 25. Cost optimization sprint 🔴

**Objective:** Find savings in a running workload and prove them with numbers.

**Tasks**
1. Run the lab stack for a day with a synthetic load, then pull cost by service and by tag with `aws ce get-cost-and-usage --group-by`.
2. Identify the top three cost drivers and a cheaper design for each (NAT per AZ versus endpoints, Graviton, gp3, right-sized RDS, scheduled scaling).
3. Apply at least three changes and measure again.
4. Evaluate Savings Plans or Reserved Instances for the steady portion and compute the break-even.
5. Write a one-page summary: before, after, percentage saved, risks.

**Expected outcome:** A documented reduction of at least 30 percent on the workload with the reasoning for each change.

<details>
<summary>Hints</summary>

`aws ce get-cost-and-usage --time-period Start=...,End=... --granularity DAILY --metrics UnblendedCost --group-by Type=TAG,Key=Project`. Graviton means `t4g` or `m7g` instances with an arm64 AMI. Compute Optimizer gives rightsizing recommendations via `aws compute-optimizer get-ec2-instance-recommendations`. Cost Explorer API calls are billed per request, so keep queries small.

</details>

**Bonus:** Create an anomaly monitor with `aws ce create-anomaly-monitor` and an alert subscription.

---

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
