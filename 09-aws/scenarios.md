# AWS — Scenario-Based Questions

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

> Real-world AWS situations and how to reason through them in an interview · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to AWS](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [IAM, Organizations and Multi-Account](#iam-organizations-and-multi-account) — Q1–Q25
- [VPC and Networking](#vpc-and-networking) — Q26–Q51
- [EC2, Auto Scaling and Load Balancing](#ec2-auto-scaling-and-load-balancing) — Q52–Q75
- [S3, RDS and DynamoDB](#s3-rds-and-dynamodb) — Q76–Q99
- [Route 53, CloudFront and Messaging](#route-53-cloudfront-and-messaging) — Q100–Q124
- [ECS, EKS, Lambda and ECR](#ecs-eks-lambda-and-ecr) — Q125–Q150
- [Observability, Secrets, KMS and CloudFormation](#observability-secrets-kms-and-cloudformation) — Q151–Q173
- [Cost Optimization and Disaster Recovery](#cost-optimization-and-disaster-recovery) — Q174–Q200

---

## IAM, Organizations and Multi-Account

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. A developer needs read-only access to one S3 bucket from their laptop. How do you grant it without creating long-lived access keys? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Human access to AWS should be federated and short-lived; an IAM user with static keys is the thing auditors flag first.

**What I'd do →**

1. Put the person in IAM Identity Center (SSO) and create a permission set with a scoped policy for that bucket
2. Assign the permission set to the user or group for the target account
3. Developer runs `aws configure sso` then `aws sso login --profile dev`
4. Verify with `aws s3 ls s3://app-reports --profile dev` and `aws sts get-caller-identity`

**Why →** SSO credentials expire in hours, access is revoked centrally when the person leaves, and CloudTrail shows the SSO role session name.

**Interview tip →** Say "no IAM users for humans" early; it signals you know the modern baseline.

</details>

### Q2. An EC2 application needs to write to DynamoDB. Where do the credentials come from? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Never bake keys into the AMI or env vars; the instance should assume a role through its instance profile.

**What I'd do →**

1. Create a role trusting `ec2.amazonaws.com` with a policy limited to `dynamodb:PutItem` on the table ARN
2. Create an instance profile and attach it: `aws ec2 associate-iam-instance-profile --instance-id i-0abc --iam-instance-profile Name=orders-app`
3. Use the default SDK credential chain in code, with no keys configured
4. Enforce IMDSv2 so SSRF cannot trivially steal the role credentials

**Why →** The SDK fetches rotating credentials from IMDS automatically, so there is nothing to leak or rotate by hand.

**Interview tip →** Mention IMDSv2 `HttpTokens=required`; it shows security awareness beyond "use a role".

</details>

### Q3. Security wants to know who deleted a production S3 bucket yesterday. How do you find out? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is a CloudTrail question; bucket deletion is a management event, recorded by default for 90 days.

**What I'd do →**

1. Open CloudTrail Event history and filter on Event name `DeleteBucket`
2. Or CLI: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket`
3. Read `userIdentity`, `sourceIPAddress` and `userAgent` to identify the principal
4. For longer history query the trail's S3 logs with Athena

**Why →** Event history only covers 90 days of management events in one region view, so an organization trail to S3 is what actually keeps the evidence.

**Interview tip →** Distinguish management events from data events (object-level), which are off by default.

</details>

### Q4. You must stop anyone in a member account from disabling CloudTrail, even account admins. What do you use? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** IAM policies inside the account cannot restrain the account admin, so the control belongs above the account, in Organizations.

**What I'd do →**

1. Create an SCP that denies `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail` and `cloudtrail:UpdateTrail`
2. Attach it to the OU holding workload accounts
3. Test from a member account admin: the call should fail with an explicit deny from the SCP
4. Keep break-glass in the management account, which SCPs do not affect

**Why →** SCPs set the maximum permissions boundary for every principal in the account except the management account.

**Interview tip →** Remember SCPs never grant anything; they only filter.

</details>

### Q5. A cross-account role assumption fails with AccessDenied even though the target role trusts the source account. What do you check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both sides must allow it: the target trust policy and an identity policy on the caller granting `sts:AssumeRole`.

**What I'd do →**

1. Check the role's trust policy principal and any `Condition` such as `sts:ExternalId` or `aws:PrincipalOrgID`
2. Check the caller's identity policy allows `sts:AssumeRole` on the exact role ARN
3. Check no SCP or permissions boundary blocks it
4. Retry: `aws sts assume-role --role-arn arn:aws:iam::222222222222:role/deploy --role-session-name test` and read the error text

**Why →** Cross-account needs an allow on both sides, whereas same-account trust can be satisfied by one.

**Interview tip →** Mention that a trust principal of the account root only delegates; the caller still needs its own allow.

</details>

### Q6. Design access for a CI pipeline in GitHub Actions to deploy to AWS without storing AWS keys in GitHub. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** OIDC federation lets GitHub present a signed token that AWS exchanges for temporary role credentials.

**What I'd do →**

1. Create an IAM OIDC provider for `token.actions.githubusercontent.com` with audience `sts.amazonaws.com`
2. Create a deploy role whose trust policy uses `sts:AssumeRoleWithWebIdentity` and a condition `token.actions.githubusercontent.com:sub` equal to `repo:acme/shop:ref:refs/heads/main`
3. In the workflow set `permissions: id-token: write` and use `aws-actions/configure-aws-credentials` with `role-to-assume`
4. Scope the role policy to only the resources it deploys

**Why →** No secret exists to steal, tokens last about an hour, and the `sub` condition ties the role to one repo and branch.

**Interview tip →** Call out that a missing `sub` condition lets any repo on GitHub assume the role.

</details>

### Q7. How would you let developers create IAM roles for their apps without letting them escalate to admin? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is the classic delegated-administration problem; solve it with permissions boundaries.

**What I'd do →**

1. Define a boundary policy that caps what any app role can ever do (for example no IAM, no Organizations)
2. Give developers `iam:CreateRole` and `iam:AttachRolePolicy` only with condition `iam:PermissionsBoundary` equal to the boundary ARN
3. Deny `iam:DeleteRolePermissionsBoundary`
4. Prefix-scope with a path such as `role/app/*`

**Why →** Effective permissions are the intersection of identity policy and boundary, so any role they create stays inside the cage.

**Interview tip →** Name the escalation path you are blocking: create a role with AdministratorAccess and assume it.

</details>

### Q8. Your company has 40 AWS accounts. How do you structure them and why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One account is a blast-radius and billing boundary; a landing zone separates concerns from day one.

**What I'd do →**

1. Use AWS Organizations with OUs: Security (log archive, audit), Infrastructure (network, shared services), Workloads split into prod and non-prod, Sandbox
2. Deploy with Control Tower or Landing Zone Accelerator for guardrails
3. Centralize CloudTrail, Config and GuardDuty into the security accounts
4. Attach SCPs per OU such as region restriction and deny leaving the org

**Why →** Separate prod and non-prod accounts give strong IAM and quota isolation that tags and policies in one account cannot.

**Interview tip →** Say "account per workload per environment" and be ready to defend the cost of more accounts.

</details>

### Q9. An access key for an IAM user was committed to a public GitHub repo 20 minutes ago. Walk through your response. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Assume it is already scraped; speed matters more than forensics at first, but preserve evidence.

**What I'd do →**

1. Deactivate the key immediately: `aws iam update-access-key --user-name ci-bot --access-key-id AKIA... --status Inactive`
2. Attach an explicit deny-all policy or use the AWSCompromisedKeyQuarantine policy AWS may have applied
3. Query CloudTrail for the key id over the exposure window for unusual calls like `RunInstances` in odd regions
4. Terminate rogue resources, check for new IAM users, roles and access keys
5. Create a replacement via a role or OIDC, then delete the old key and purge it from git history

**Why →** Deactivate before delete so you can still audit the key and roll back if a legitimate system breaks.

**Interview tip →** Mention enabling secret scanning push protection to prevent a repeat.

</details>

### Q10. A Lambda function cannot read an S3 bucket in another account even though the bucket policy allows the role. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-account S3 access requires the identity policy in the caller account and the bucket policy to both allow, and KMS if the objects are encrypted.

**What I'd do →**

1. Confirm the Lambda execution role has `s3:GetObject` on the bucket ARN and object path
2. Confirm the bucket policy principal is the role ARN, not just the account root
3. If SSE-KMS, check the key policy and the role's `kms:Decrypt` on that key
4. Check the bucket for Block Public Access or a VPC endpoint policy denying the call

**Why →** Missing KMS permission produces an AccessDenied that looks identical to a missing S3 permission, which wastes hours.

**Interview tip →** Always ask "is the object KMS-encrypted?" first in cross-account S3 problems.

</details>

### Q11. Design break-glass access for production that is audited, time-bound, and not a shared password. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Break-glass must exist and be loud; the failure modes are shared credentials and no alarm.

**What I'd do →**

1. Create a dedicated `BreakGlassAdmin` permission set in Identity Center, assigned to a small group, with session duration 1 hour
2. Require MFA with phishing-resistant keys
3. EventBridge rule on the `AssumeRole` or `CreateCredential` events for that role pages the on-call and posts to the security channel
4. Keep two root account credentials sealed offline with hardware MFA for the case SSO itself is down
5. Quarterly drill, then review the CloudTrail trail of every use

**Why →** Auditability plus alerting turns emergency access into an accountable action rather than a backdoor.

**Interview tip →** Mention you also test it, since untested break-glass fails when needed.

</details>

### Q12. You need to guarantee no resources are ever created outside eu-west-1 and eu-central-1 across the org. What is the design and the pitfalls? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Region restriction is an SCP job, but global services make a naive deny break things.

**What I'd do →**

1. Write an SCP with `Deny` and `NotAction` listing global services (IAM, Organizations, Route 53, CloudFront, Support, STS)
2. Condition `StringNotEquals` on `aws:RequestedRegion` for the two allowed regions
3. Add exception for control-plane roles such as the Control Tower execution role via `ArnNotLike` on `aws:PrincipalARN`
4. Apply to a test OU first and watch CloudTrail for denied calls

**Why →** Using `Action: *` without carving out global endpoints in us-east-1 breaks IAM, billing and CloudFront calls.

**Interview tip →** Mention opt-in regions and that the CloudFront and ACM-for-CloudFront certificate must live in us-east-1.

</details>

### Q13. Two accounts need private access to each other's APIs, with overlapping CIDRs. What do you build? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Overlapping CIDRs rule out VPC peering and Transit Gateway routing, so expose services rather than networks.

**What I'd do →**

1. In the provider account put the service behind an NLB and create a VPC endpoint service with `aws ec2 create-vpc-endpoint-service-configuration --network-load-balancer-arns ...`
2. Whitelist the consumer account principal and require acceptance
3. In the consumer account create an interface endpoint to the service name
4. Consumer calls the endpoint's private DNS name, unaffected by CIDR overlap

**Why →** PrivateLink is one-directional and NAT-less at the packet level, so address overlap never matters.

**Interview tip →** Contrast: peering and TGW share routes, PrivateLink shares a service.

</details>

### Q14. The audit team requires that every S3 bucket in the org is encrypted and not public. How do you enforce and detect? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Combine preventive controls (SCP, account settings) and detective controls (Config).

**What I'd do →**

1. Turn on S3 Block Public Access at the account level via `aws s3control put-public-access-block` and enforce through an SCP denying its removal
2. Enable default SSE-S3 or SSE-KMS (default on since 2023) and add a bucket-policy or SCP that denies `s3:PutObject` without the expected encryption header when KMS is required
3. Deploy Config managed rules `s3-bucket-public-read-prohibited` and `s3-bucket-server-side-encryption-enabled` through an org conformance pack
4. Auto-remediate with SSM Automation

**Why →** Prevent what you can, detect the rest; either alone leaves gaps.

**Interview tip →** Mention Security Hub CSPM standards for an aggregated score.

</details>

### Q15. What is the difference between an IAM role and an IAM user, and when do you pick each? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A user is a long-lived identity with static credentials; a role is an assumable identity with temporary credentials.

**What I'd do →**

1. Use roles for EC2, Lambda, ECS tasks, cross-account access and CI
2. Use SSO-federated roles for humans
3. Reserve IAM users for rare cases like a third-party tool that cannot federate, and then rotate keys and restrict by condition
4. Audit with the credential report: `aws iam generate-credential-report`

**Why →** Temporary credentials shrink the window of a leak from forever to hours.

**Interview tip →** One-line answer: "roles for everything, users almost never".

</details>

### Q16. A teammate says "just give it AdministratorAccess to get unblocked". How do you handle it properly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least privilege can be reached fast by letting AWS tell you what the app actually used.

**What I'd do →**

1. Allow the broad permission only in a dev account for a short window
2. Run the workload, then use IAM Access Analyzer policy generation from CloudTrail: it builds a policy from observed calls
3. Review the draft, replace wildcards with ARNs, and apply it
4. Remove the admin policy and check `aws iam get-role` for attached policies

**Why →** Generated policies come from real activity so they are tighter than guesses.

**Interview tip →** Mention Access Analyzer unused-access findings for cleanup later.

</details>

### Q17. A policy has an Allow for `s3:*` but a user still gets AccessDenied on `PutObject`. List the places a deny can hide. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Evaluation is explicit deny first, so I hunt for any deny across the layers.

**What I'd do →**

1. SCP in Organizations
2. Permissions boundary on the role or user
3. Session policy passed during assume-role
4. Bucket policy with `Deny` conditions such as `aws:SecureTransport` or required encryption
5. VPC endpoint policy
6. Use the IAM policy simulator or `aws iam simulate-principal-policy` and CloudTrail `errorMessage`, which often names the deny type

**Why →** AWS error messages now say "explicit deny in a service control policy" or similar, which pinpoints the layer.

**Interview tip →** Recite the evaluation order: explicit deny, SCP, boundary, session, identity or resource allow.

</details>

### Q18. How do you give an on-premises server AWS access without IAM users? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** IAM Roles Anywhere issues temporary credentials against X.509 certificates, or SSM hybrid activation covers management tasks.

**What I'd do →**

1. Create a trust anchor from your private CA or ACM Private CA
2. Create a profile mapping to an IAM role
3. Install `aws_signing_helper` on the server and configure `credential_process` in the AWS profile
4. Restrict the trust policy with a condition on the certificate subject

**Why →** The server proves identity by certificate rather than a stored secret, and credentials rotate automatically.

**Interview tip →** Mention certificate lifecycle as the new operational burden.

</details>

### Q19. A product team wants to give an external vendor limited access to one account. Design it safely. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Third-party access must prevent the confused deputy problem and be revocable.

**What I'd do →**

1. Create a role in your account trusting the vendor's account with `sts:ExternalId` condition set to a unique value you generate
2. Attach a narrowly scoped policy plus a permissions boundary
3. Require `aws:MultiFactorAuthPresent` only if their human flow supports it, otherwise limit by `aws:SourceIp`
4. Alert on the role's use via CloudTrail and set `MaxSessionDuration` to 1 hour

**Why →** ExternalId stops another customer of the same vendor from tricking the vendor into using your role.

**Interview tip →** Say "confused deputy" by name.

</details>

### Q20. The finance team asks for monthly cost by team. How do you structure IAM and tagging to make that possible? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Attribution needs enforcement at creation time, not a spreadsheet after the fact.

**What I'd do →**

1. Define tag keys (`team`, `env`, `cost-center`) and activate them as cost allocation tags in Billing
2. Enforce with an SCP or tag policy and IAM condition `aws:RequestTag/team` on create actions
3. Use an AWS Config rule `required-tags` to catch drift
4. Use account-per-team where possible, since account boundary is the cleanest chargeback

**Why →** Untagged resources become unallocatable spend that nobody owns.

**Interview tip →** Mention that some services (data transfer, shared EKS) need split allocation tooling.

</details>

### Q21. Your SCP change accidentally locked everyone out of a production OU, including the pipeline. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** SCPs do not apply to the management account, which is the recovery path.

**What I'd do →**

1. Sign in to the management account with an admin who has Organizations permissions
2. Detach or fix the SCP: `aws organizations detach-policy --policy-id p-xxxx --target-id ou-abcd-12345678`
3. Verify member-account calls work again
4. Add a post-mortem action to roll out SCPs to a canary OU first with CloudTrail review

**Why →** Management account is exempt by design, which is also why you must keep workloads out of it.

**Interview tip →** Always keep `FullAWSAccess` attached alongside deny lists unless you intentionally use allow-list mode.

</details>

### Q22. How do you rotate IAM access keys used by a legacy app you cannot refactor quickly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two keys per user permit zero-downtime rotation; automation prevents it from being forgotten.

**What I'd do →**

1. Create the second key: `aws iam create-access-key --user-name legacy-app`
2. Store it in Secrets Manager and update the app configuration
3. Confirm via `aws iam get-access-key-last-used` that the new key is in use
4. Deactivate then delete the old key
5. Schedule it with a Lambda or Secrets Manager rotation template, and alert via Config rule `access-keys-rotated`

**Why →** Overlap eliminates the outage window that makes teams avoid rotation.

**Interview tip →** Offer the long-term fix: move the app to a role.

</details>

### Q23. What does MFA enforcement for the console look like in a shared account, and how do you require it for API calls too? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Console MFA is easy; API MFA is enforced via a policy condition.

**What I'd do →**

1. Use SSO with MFA enforced in Identity Center settings
2. For remaining IAM users attach a policy denying all except MFA-device management when `aws:MultiFactorAuthPresent` is false
3. For API use, users call `aws sts get-session-token --serial-number arn:aws:iam::111111111111:mfa/alice --token-code 123456`
4. Prefer hardware or passkey MFA for admins

**Why →** The deny-unless-MFA pattern gives enforcement even when the user has valid keys.

**Interview tip →** Mention root user MFA and no root access keys as the first hygiene items.

</details>

### Q24. An engineer left the company. How do you make sure nothing of theirs still works? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Offboarding is easy if identity is centralized, hard if they created IAM users or keys.

**What I'd do →**

1. Disable the user in the IdP, which kills SSO sessions into every account
2. Run the credential report to find any IAM users or keys: `aws iam get-credential-report`
3. Search for roles they created with trust to personal accounts
4. Rotate any shared secrets they knew
5. Check CloudTrail for the user's actions the final week

**Why →** Central IdP removal handles 95 percent of access; the remainder is the shadow credentials.

**Interview tip →** Mention periodic IAM Access Analyzer external-access findings.

</details>

### Q25. Walk me through how you would implement least privilege at scale for 200 roles that currently have wide permissions. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You cannot hand-review 200 roles, so use data to shrink them in waves.

**What I'd do →**

1. Enable Access Analyzer unused-access analyzer at the org level
2. Rank roles by findings of unused permissions and unused roles
3. Generate policies from CloudTrail for the top risks, replace with scoped ones in lower environments first
4. Roll to prod behind a monitored change window with the old policy kept for rollback
5. Add policy-as-code checks (IAM Access Analyzer `validate-policy` or cfn-guard) in CI

**Why →** Iterative tightening with rollback beats a big-bang rewrite that breaks production.

**Interview tip →** Mention that you measure progress by unused-permission count dropping.

</details>

## VPC and Networking

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. You are asked to design a VPC for a three-tier web app. What does the layout look like? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Public only for what must face the internet; everything else private across at least two AZs.

**What I'd do →**

1. Pick a /16 such as `10.20.0.0/16` and carve /20 or /24 subnets per tier per AZ
2. Public subnets hold ALB and NAT gateways, private app subnets hold compute, isolated data subnets hold RDS with no internet route
3. Route tables: public to IGW, private to NAT, data with local only
4. Security groups chained: ALB to app to DB by SG reference

**Why →** Tier isolation and SG referencing limit lateral movement and keep rules readable.

**Interview tip →** Leave spare CIDR space; resizing a VPC later hurts.

</details>

### Q27. Instances in a private subnet cannot reach the internet to download packages. What do you add? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Private subnets need an outbound path, typically a NAT gateway in a public subnet.

**What I'd do →**

1. Create a NAT gateway in a public subnet with an Elastic IP
2. Add `0.0.0.0/0 -> nat-0abc` to the private route table
3. Test from the instance: `curl -sI https://example.com`
4. For one NAT per AZ, add one route table per AZ

**Why →** NAT gives outbound-only access; inbound connections from the internet still cannot be initiated.

**Interview tip →** Mention NAT cost and the gateway endpoint alternative for S3 and DynamoDB.

</details>

### Q28. Your NAT gateway bill is $4,000 a month. How do you cut it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** NAT charges per hour and per GB; most heavy traffic is usually to AWS services or ECR and S3.

**What I'd do →**

1. Use VPC Flow Logs or CloudWatch metric `BytesOutToDestination` to find top talkers
2. Add free gateway endpoints for S3 and DynamoDB
3. Add interface endpoints for ECR, STS, CloudWatch Logs where traffic is high, since endpoint per-GB cost is lower than NAT
4. Keep chatty cross-AZ traffic in the same AZ as its NAT
5. Consider NAT instances only for dev environments

**Why →** Most NAT spend is avoidable traffic to AWS public endpoints.

**Interview tip →** Mention that cross-AZ data transfer through NAT adds a second charge.

</details>

### Q29. What is the difference between a security group and a network ACL, and when would you touch NACLs? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** SGs are stateful and attach to ENIs; NACLs are stateless and attach to subnets.

**What I'd do →**

1. Default to SGs for all allow rules
2. Use NACLs for coarse subnet-wide denies such as blocking a hostile CIDR
3. Remember NACL rules need both inbound and outbound with ephemeral ports `1024-65535`
4. Inspect: `aws ec2 describe-network-acls --filters Name=vpc-id,Values=vpc-0abc`

**Why →** SG rules are allow-only, NACLs support explicit deny and are evaluated in rule-number order.

**Interview tip →** State the "stateful vs stateless" contrast first.

</details>

### Q30. Connect 30 VPCs across 10 accounts with shared on-prem connectivity. VPC peering or Transit Gateway? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Peering is non-transitive and creates a mesh; TGW is a hub with route tables.

**What I'd do →**

1. Create a TGW in the network account and share it with RAM across the org
2. Attach each VPC and the VPN or Direct Connect gateway
3. Use separate TGW route tables to segment prod from dev
4. Add VPC route `10.0.0.0/8 -> tgw-0abc` and test with Reachability Analyzer

**Why →** Peering scales as n squared; TGW scales linearly and supports central egress and inspection.

**Interview tip →** Know the TGW cost: per attachment per hour plus per GB.

</details>

### Q31. Design hybrid connectivity to on-prem that survives a link failure. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Redundancy must exist at the device, circuit and path level.

**What I'd do →**

1. Primary: Direct Connect, ideally two connections at different locations
2. Backup: Site-to-Site VPN over the internet attached to the same Transit Gateway or Virtual Private Gateway
3. Use BGP on both with local preference or AS-path prepending favoring DX
4. Enable BFD and test failover by shutting one path
5. Monitor `TunnelState` and DX `ConnectionState` metrics

**Why →** DX gives predictable latency; VPN gives fast, cheap insurance with automatic BGP failover.

**Interview tip →** A single DX is a single point of failure; say so.

</details>

### Q32. Traffic from your VPC to S3 goes through the NAT gateway. How do you keep it on the AWS network and add control? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A gateway endpoint routes S3 traffic privately and allows policy control at no charge.

**What I'd do →**

1. Create the endpoint: `aws ec2 create-vpc-endpoint --vpc-id vpc-0abc --service-name com.amazonaws.eu-west-1.s3 --route-table-ids rtb-0abc`
2. Attach an endpoint policy restricting access to your buckets
3. Add a bucket policy condition `aws:SourceVpce` so the bucket is only reachable through the endpoint
4. Verify with `traceroute` or route table entry for the prefix list

**Why →** The endpoint adds a prefix-list route that wins over the default route, so NAT is bypassed.

**Interview tip →** Gateway endpoints exist only for S3 and DynamoDB; everything else uses interface endpoints.

</details>

### Q33. You must inspect all egress traffic from 50 VPCs centrally. What is the architecture? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Centralized egress with an inspection VPC behind Transit Gateway.

**What I'd do →**

1. Spoke VPCs send `0.0.0.0/0` to the TGW
2. TGW route table for spokes points to an inspection VPC attachment with appliance mode enabled
3. Inspection VPC runs AWS Network Firewall or third-party firewalls behind a Gateway Load Balancer
4. Firewall subnets forward to NAT gateways in an egress VPC
5. Return path via TGW; enable flow logs on all attachments

**Why →** Appliance mode keeps both directions of a flow on the same AZ so stateful inspection works.

**Interview tip →** Mention the cost and single-choke-point trade-off versus per-VPC firewalls.

</details>

### Q34. Two services in different AZs talk constantly and the cross-AZ data transfer charge is rising. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-AZ traffic costs about a cent per GB each direction, which adds up for chatty tiers.

**What I'd do →**

1. Identify with Cost Explorer usage type `DataTransfer-Regional-Bytes` and flow logs
2. Prefer zone-aware routing: Kubernetes topology-aware routing or ALB zonal affinity
3. Keep caches and replicas per AZ
4. Accept cross-AZ for HA-critical paths such as DB replication
5. Re-measure after the change

**Why →** Resilience needs multi-AZ, but balancing locality and failover captures savings.

**Interview tip →** Don't suggest single-AZ for production just to save money.

</details>

### Q35. A new subnet's instances cannot be reached via SSH from the internet. Walk through the checks. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Connectivity needs an IGW route, a public IP, an SG rule and a NACL that allows it.

**What I'd do →**

1. Subnet route table has `0.0.0.0/0 -> igw-...`
2. Instance has a public or Elastic IP
3. SG allows TCP 22 from your IP
4. NACL allows inbound 22 and outbound ephemeral ports
5. Better, remove public SSH entirely and use SSM Session Manager: `aws ssm start-session --target i-0abc`

**Why →** Each layer can independently block; checking in packet order finds the break quickly.

**Interview tip →** Suggest Session Manager as the modern replacement.

</details>

### Q36. How would you give private instances access to AWS Systems Manager without NAT or public IPs? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SSM needs three interface endpoints in the VPC.

**What I'd do →**

1. Create endpoints `ssm`, `ssmmessages` and `ec2messages` with private DNS enabled
2. Attach an SG allowing 443 from the instance subnets
3. Ensure the instance role has `AmazonSSMManagedInstanceCore`
4. Test: instance appears in `aws ssm describe-instance-information`

**Why →** Instances reach SSM purely on the AWS backbone, shrinking attack surface.

**Interview tip →** Forgetting `ssmmessages` is the usual reason Session Manager fails.

</details>

### Q37. Two VPCs peered, but the app in VPC A cannot reach VPC B's database. What is the routine? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Peering needs routes on both sides plus SG and DNS settings.

**What I'd do →**

1. Check peering is `active`: `aws ec2 describe-vpc-peering-connections`
2. Add routes in both VPC route tables to the peer CIDR
3. Check the DB SG allows the source CIDR or the peer SG reference (cross-account SG references work only in the same region)
4. Check NACLs
5. Enable DNS resolution over peering if using private hostnames

**Why →** Peering creates a path only; routing and filtering are still your job.

**Interview tip →** Peering never routes through a third VPC.

</details>

### Q38. Design IPv6 adoption for a dual-stack app to reduce NAT costs. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** IPv6 addresses are globally routable, so private egress uses an egress-only internet gateway instead of NAT.

**What I'd do →**

1. Associate an Amazon-provided /56 with the VPC and /64 with subnets
2. Add `::/0 -> eigw-0abc` in private route tables
3. Enable dual-stack on the ALB (`--ip-address-type dualstack`)
4. Update SGs with IPv6 rules
5. Keep NAT64 and DNS64 for IPv4-only destinations

**Why →** IPv6 egress via EIGW has no hourly or per-GB NAT fee, and public IPv4 addresses now cost money too.

**Interview tip →** Remember that public IPv4 charges began in 2024; it drives IPv6 interest.

</details>

### Q39. An application needs a stable outbound IP that a partner can whitelist. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Outbound IPs come from NAT gateway Elastic IPs, which are stable.

**What I'd do →**

1. Create a NAT gateway with an allocated EIP in each AZ
2. Route private subnets through it
3. Give the partner all EIPs
4. For Lambda, place it in the VPC private subnets so egress uses the NAT
5. Verify `curl https://checkip.amazonaws.com`

**Why →** Auto-assigned public IPs change on stop/start, Elastic IPs and NAT EIPs do not.

**Interview tip →** Mention you need all AZ EIPs on the allowlist, not just one.

</details>

### Q40. You must expose an internal service to 200 customer VPCs privately without peering. What do you build? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** PrivateLink endpoint service scales to many consumers with no CIDR coordination.

**What I'd do →**

1. Put the service behind an NLB
2. Create the endpoint service and allow principals by account or org
3. Enable acceptance if you want control
4. Consumers create interface endpoints, optionally with private DNS names verified on your domain
5. Monitor `ActiveConnections` and use cross-zone settings

**Why →** Consumers get a local ENI in their subnets, so there is no routing or overlap issue.

**Interview tip →** Mention that NLB is required (or GWLB for appliances).

</details>

### Q41. What is a VPC flow log and how would you use it when a connection is mysteriously dropped? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Flow logs record accepted and rejected flows per ENI, subnet or VPC.

**What I'd do →**

1. Enable: `aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-0abc --traffic-type ALL --log-destination-type s3 --log-destination arn:aws:s3:::flowlogs-bucket`
2. Query with Athena for `action = 'REJECT'` on the destination IP and port
3. Distinguish SG vs NACL: stateless NACL rejects often show up on return traffic
4. Fix the rule that shows the REJECT

**Why →** Flow logs show whether traffic reached the ENI and was refused, narrowing blame to the network.

**Interview tip →** Flow logs exclude DNS to Route 53 resolver and IMDS traffic.

</details>

### Q42. The ALB health checks fail only for targets in one subnet. What could differ? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One subnet differs in routing, NACL, or SG source.

**What I'd do →**

1. Compare NACLs on the failing subnet for the health check port
2. Confirm the target SG allows the ALB SG on the health check port
3. Confirm the subnet is enabled for the ALB's AZ
4. Check the instance's local firewall and app listening address with `ss -ltnp`

**Why →** Health checks originate from ALB ENIs in the enabled subnets, so subnet-level rules matter.

**Interview tip →** Mention app binding to `127.0.0.1` as the classic cause.

</details>

### Q43. Your company is acquiring another with the same 10.0.0.0/16. How do you integrate networks? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Overlap blocks direct routing; options are renumbering, NAT or service-level exposure.

**What I'd do →**

1. Short term: expose only needed services with PrivateLink
2. Or use a private NAT gateway with a non-overlapping translation range via Transit Gateway
3. Long term: renumber one side to a new range during migrations, and plan with IPAM
4. Adopt Amazon VPC IPAM to prevent new overlaps

**Why →** Private NAT lets overlapping networks talk with translated addresses without renumbering first.

**Interview tip →** Mention secondary CIDR associations for new workloads.

</details>

### Q44. How would you secure a VPC against data exfiltration to the internet? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default-deny egress with allow-listed destinations and visibility.

**What I'd do →**

1. Remove public IPs, route egress through NAT and AWS Network Firewall using domain-list rules
2. Tighten SG egress rules
3. Use Route 53 Resolver DNS Firewall to block unknown domains
4. Add endpoint policies for S3 to restrict to your buckets
5. Alert on GuardDuty exfiltration findings

**Why →** DNS and S3 are the typical exfiltration paths, so control both.

**Interview tip →** SG default egress allows all; mention that you tighten it.

</details>

### Q45. What happens when you delete the default VPC and why might you? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default VPCs have public subnets and permissive defaults that bypass your network design.

**What I'd do →**

1. Delete in every region via CLI, including subnets, IGW and the VPC
2. Prevent recreation of workloads in them using an SCP and detect with Config
3. If needed, `aws ec2 create-default-vpc` restores it

**Why →** Removing it prevents people launching instances into an unmanaged, internet-routable network.

**Interview tip →** Mention doing it for all regions, not just the one you use.

</details>

### Q46. Latency between two regions is high for a replication job. What network options exist? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Public internet paths vary; AWS backbone options offer more consistency.

**What I'd do →**

1. Use inter-region VPC peering or Transit Gateway peering, which stay on the AWS backbone
2. Check throughput limits and use multiple flows because single-flow bandwidth is limited
3. For S3 use Cross-Region Replication, for DBs use native replication
4. Test with `iperf3` across two instances

**Why →** Backbone traffic is encrypted and avoids congestion hotspots on the public internet.

**Interview tip →** Know that single TCP flow caps at about 5 Gbps within a placement group and less across regions.

</details>

### Q47. Design DNS for a hybrid environment where on-prem and AWS must resolve each other's private names. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Route 53 Resolver endpoints forward in both directions.

**What I'd do →**

1. Create an inbound endpoint so on-prem DNS can forward `*.aws.internal` to it
2. Create an outbound endpoint and forwarding rule for `corp.example.com` to on-prem DNS servers
3. Share rules via RAM across accounts
4. Associate private hosted zones with all VPCs
5. Test with `dig @10.0.1.10 db.aws.internal` from on-prem

**Why →** Resolver endpoints bridge the VPC's `.2` resolver and external servers without running your own DNS fleet.

**Interview tip →** Put endpoints in at least two AZs.

</details>

### Q48. An EKS cluster ran out of IP addresses in its subnets. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The VPC CNI assigns a VPC IP per pod, so small subnets exhaust quickly.

**What I'd do →**

1. Associate a secondary CIDR such as `100.64.0.0/16` with the VPC
2. Enable custom networking so pods use the new subnets
3. Or enable prefix delegation: `kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true`
4. Long term, size subnets larger from the start

**Why →** Secondary CIDRs expand capacity without re-IPing nodes.

**Interview tip →** Mention IPv6 EKS clusters avoid the problem entirely.

</details>

### Q49. How do you restrict an S3 bucket so it is reachable only from your VPC? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Combine a gateway endpoint with a bucket policy condition.

**What I'd do →**

1. Add a Deny statement with `StringNotEquals` on `aws:SourceVpce` equal to `vpce-0abc`
2. Keep an exception for admin roles or the console flow
3. Test from inside and outside the VPC
4. Don't lock out CloudFormation or other AWS service principals

**Why →** A deny-unless-endpoint policy blocks leaked credentials used from outside.

**Interview tip →** Mention the risk of locking out yourself; test with a break-glass role.

</details>

### Q50. Design a multi-AZ NAT strategy and justify cost versus resilience. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One NAT gateway per AZ avoids cross-AZ charges and an AZ-failure blast radius.

**What I'd do →**

1. Deploy a NAT gateway in each public subnet
2. Per-AZ private route tables point to the NAT in the same AZ
3. For dev, use one NAT to save about $32 per month per gateway
4. Alarm on `ErrorPortAllocation` and `PacketsDropCount`
5. Use IPv6 or endpoints to reduce the load

**Why →** An AZ outage with a single NAT takes all private-subnet egress with it.

**Interview tip →** Prod: one per AZ. Non-prod: one shared.

</details>

### Q51. What is the purpose of an Elastic IP and what is the downside of leaving them unattached? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** EIP is a static public IPv4 owned by the account.

**What I'd do →**

1. Allocate: `aws ec2 allocate-address --domain vpc`
2. Associate with an instance or NAT gateway
3. List unassociated: `aws ec2 describe-addresses --query 'Addresses[?AssociationId==null]'`
4. Release those you do not need

**Why →** Public IPv4 addresses are billed hourly whether or not attached.

**Interview tip →** Prefer DNS and load balancers over EIPs for services.

</details>

## EC2, Auto Scaling and Load Balancing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q52. Traffic doubles every Friday evening. How do you make the web tier handle it automatically? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Predictable spikes call for scheduled scaling plus a reactive policy as a safety net.

**What I'd do →**

1. Put instances in an Auto Scaling group across at least two AZs behind an ALB
2. Add a target tracking policy on `ALBRequestCountPerTarget` or CPU at 50 to 60 percent
3. Add a scheduled action to raise `MinSize` Friday at 16:00
4. Use a launch template with a baked AMI so instances are ready in under two minutes

**Why →** Scheduled capacity removes cold-start lag, target tracking handles the surprise.

**Interview tip →** Mention predictive scaling for repeating patterns.

</details>

### Q53. What is the difference between an ALB, an NLB and a Gateway Load Balancer? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** They operate at different layers for different jobs.

**What I'd do →**

1. ALB is layer 7: host and path routing, redirects, OIDC auth, WebSocket and gRPC
2. NLB is layer 4: TCP, UDP, TLS, static IPs per AZ, extreme throughput, PrivateLink backend
3. GWLB is layer 3: transparent insertion of firewalls and inspection appliances over GENEVE
4. Choose ALB for HTTP microservices, NLB for non-HTTP or fixed IP, GWLB for appliances

**Why →** Matching layer to need avoids paying for features you do not use.

**Interview tip →** NLB preserves client IP by default for instance targets.

</details>

### Q54. You deploy a new version to an ASG without downtime. Which approach do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Instance refresh with health-gated batches or blue/green with two target groups.

**What I'd do →**

1. Update the launch template version
2. Start: `aws autoscaling start-instance-refresh --auto-scaling-group-name web --preferences '{"MinHealthyPercentage":90,"InstanceWarmup":120}'`
3. Watch with `describe-instance-refresh`
4. For safer cutovers use a second ASG and weighted ALB target groups, then shift 10, 50, 100 percent

**Why →** Refresh replaces instances gradually, honoring health so capacity never drops below the floor.

**Interview tip →** Mention checkpoints and auto-rollback preferences.

</details>

### Q55. Instances in an ASG keep getting terminated and replaced in a loop. What is going on? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Health checks are failing before the app is ready, or user data is crashing.

**What I'd do →**

1. Look at the activity history: `aws autoscaling describe-scaling-activities --auto-scaling-group-name web`
2. Check cause text such as "Terminating EC2 instance: health check type ELB"
3. Increase `HealthCheckGracePeriod` beyond boot plus app start time
4. Read `/var/log/cloud-init-output.log` via a suspended instance
5. Use `aws autoscaling suspend-processes --scaling-processes ReplaceUnhealthy` to debug

**Why →** ELB health checks start immediately after grace period; a slow boot keeps tripping them.

**Interview tip →** Suspending ReplaceUnhealthy is the debug trick that proves you have done this.

</details>

### Q56. You want to save up to 70 percent on a stateless batch fleet. What do you design? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Spot Instances with diversification and interruption handling.

**What I'd do →**

1. Use an ASG mixed instances policy across 8 to 10 instance types and all AZs with allocation strategy `price-capacity-optimized`
2. Keep a small On-Demand base
3. Handle the two-minute interruption notice via EventBridge or IMDS to drain and checkpoint
4. Use SQS so work is retried on interruption
5. Enable capacity rebalancing

**Why →** Diversified pools lower the chance that one capacity pool dries up.

**Interview tip →** Never run stateful singletons on Spot.

</details>

### Q57. Choose between Savings Plans and Reserved Instances for a steady fleet that is changing instance families. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compute Savings Plans keep flexibility across family, region and even Fargate and Lambda.

**What I'd do →**

1. Review usage in Cost Explorer Savings Plans recommendations
2. Commit to roughly 70 to 80 percent of the steady baseline, not peak
3. Choose Compute SP for flexibility, EC2 Instance SP for a higher discount when family is fixed
4. Use RIs only for specific cases such as RDS or capacity reservations
5. Revisit quarterly

**Why →** Over-commitment wastes money; under-commitment leaves discount on the table.

**Interview tip →** Savings Plans do not reserve capacity; RIs with zonal scope do.

</details>

### Q58. A stopped-and-started EC2 instance changed its public IP and broke the DNS record. How do you prevent that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Auto-assigned public IPs are released on stop.

**What I'd do →**

1. Put a load balancer in front and point DNS at it
2. Or associate an Elastic IP
3. Or use a Route 53 alias to the ALB
4. Avoid pointing DNS at instance IPs

**Why →** Load balancer DNS names are stable endpoints that survive instance replacement.

**Interview tip →** Treat instances as cattle.

</details>

### Q59. An ALB returns 502 intermittently during deployments. How do you stop that? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Targets are being deregistered with in-flight requests, or the app closes keep-alive connections early.

**What I'd do →**

1. Set deregistration delay to match the longest request, such as 30 to 60 seconds
2. Make the app handle SIGTERM and finish requests
3. Set app keep-alive timeout greater than the ALB idle timeout (60 s default)
4. Use slow start for new targets
5. Check `HTTPCode_ELB_502_Count` and target response time metrics

**Why →** A backend closing idle connections sooner than the ALB expects causes 502s on reused connections.

**Interview tip →** State the rule: backend keep-alive greater than load balancer idle timeout.

</details>

### Q60. Design a zero-downtime blue/green for a monolith on EC2 behind an ALB. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two environments, one listener, weighted shift with fast rollback.

**What I'd do →**

1. Create blue and green ASGs, each with its own target group
2. Deploy the new AMI to green and run smoke tests against a test listener on port 8443
3. Shift listener weights 10, 50, 100 percent with alarms on 5xx and latency
4. Keep blue running for 30 minutes then scale to zero
5. Roll back by flipping weights back: `aws elbv2 modify-listener --listener-arn ... --default-actions ...`

**Why →** Rollback is a weight change, not a redeploy, so recovery takes seconds.

**Interview tip →** Mention database migrations must be backward compatible for this to work.

</details>

### Q61. An EC2 instance is running at 100 percent CPU with CPU credits at zero. What happened? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A burstable T-family instance exhausted its credits and is throttled to baseline.

**What I'd do →**

1. Confirm `CPUCreditBalance` in CloudWatch
2. Switch to unlimited mode: `aws ec2 modify-instance-credit-specification`
3. Or right-size to M or C family
4. Or scale out with an ASG
5. Compare costs; unlimited bills surplus credits

**Why →** T instances are cheap only for low average CPU; sustained load needs fixed-performance instances.

**Interview tip →** Never run steady production CPU on T-class without unlimited.

</details>

### Q62. Your ASG launched instances in only one AZ and an AZ outage took down the service. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The ASG was configured with a single subnet, or the AZs were imbalanced.

**What I'd do →**

1. Check `aws autoscaling describe-auto-scaling-groups` for `AvailabilityZones`
2. Add subnets across 3 AZs
3. Make sure the ALB has those AZs enabled
4. Enable AZ rebalancing
5. Test with an AZ failure drill using ARC zonal shift

**Why →** Multi-AZ is only real if both the ALB and the ASG span it with enough capacity per AZ.

**Interview tip →** Mention over-provisioning N+1 across AZs.

</details>

### Q63. How would you connect to a private EC2 instance for troubleshooting without a bastion? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use Systems Manager Session Manager.

**What I'd do →**

1. Ensure the instance has `AmazonSSMManagedInstanceCore`
2. Start: `aws ssm start-session --target i-0abc`
3. For port forwarding use `--document-name AWS-StartPortForwardingSession --parameters portNumber=5432,localPortNumber=5432`
4. Session logs go to S3 or CloudWatch for audit

**Why →** No inbound ports, no SSH keys, and every session is audited.

**Interview tip →** Mention EC2 Instance Connect Endpoint as another keyless option.

</details>

### Q64. The business requires 99.99 percent for a customer API. How do you compose EC2 components? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-AZ everything, no single points, and automated recovery.

**What I'd do →**

1. ALB across three AZs, ASG with min 3
2. Stateless instances, state in DynamoDB or Multi-AZ Aurora
3. Health checks at ELB level, deep health endpoints
4. Route 53 health checks with failover to a second region if needed
5. Error budget alarms and game-day tests

**Why →** Availability multiplies; each tier must itself be redundant.

**Interview tip →** Be honest about regional dependencies like IAM and Route 53 control planes.

</details>

### Q65. Right-size a fleet of 300 instances where nobody knows actual usage. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data-driven right-sizing with guardrails, not guessing.

**What I'd do →**

1. Enable Compute Optimizer across the org and install CloudWatch agent for memory metrics
2. Pull recommendations: `aws compute-optimizer get-ec2-instance-recommendations`
3. Sort by savings and risk, start with over-provisioned dev and staging
4. Change one size down at a time and watch p99 latency
5. Move to Graviton (m7g, c7g) where the stack supports ARM

**Why →** CPU-only data misses memory pressure; the agent metric makes the call accurate.

**Interview tip →** Graviton typically gives 20 percent better price-performance.

</details>

### Q66. User data script ran but the app is not installed. How do you debug a new launch template? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** User data runs once at first boot as root through cloud-init, and failures are silent unless you look.

**What I'd do →**

1. SSM into the instance and read `/var/log/cloud-init-output.log`
2. Run `cloud-init status --long`
3. Check the script has a shebang and `set -euxo pipefail`
4. Verify the instance has outbound access for package repositories
5. Fix the template and refresh

**Why →** Most user data failures are no network path or a missing shebang.

**Interview tip →** Better: bake an AMI with EC2 Image Builder so boot does less.

</details>

### Q67. What is the difference between an AMI, a snapshot and an instance store? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** AMI is a template, a snapshot is a block-level backup in S3, instance store is ephemeral local disk.

**What I'd do →**

1. Create an AMI from an instance: `aws ec2 create-image --instance-id i-0abc --name web-v12`
2. Snapshots back EBS volumes and are incremental
3. Instance store data is lost on stop or termination
4. Use it only for caches or scratch

**Why →** Knowing which storage persists is basic to not losing data.

**Interview tip →** EBS persists after stop; instance store does not.

</details>

### Q68. Your fleet needs OS patching monthly with no manual SSH. How do you automate it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Systems Manager Patch Manager with maintenance windows.

**What I'd do →**

1. Tag instances by `PatchGroup`
2. Create a patch baseline and a maintenance window
3. Register the `AWS-RunPatchBaseline` task with rate control (10 percent concurrency, 5 percent error threshold)
4. Combine with ASG instance refresh to bake new AMIs instead of in-place patching for immutable fleets
5. Report compliance in the console

**Why →** Immutable AMI rotation beats patching in place for repeatability.

**Interview tip →** Mention rate control to avoid patching the whole fleet at once.

</details>

### Q69. An ALB must route `/api` to one service and `/` to another, and also redirect HTTP to HTTPS. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Listener rules handle path routing and redirects natively.

**What I'd do →**

1. Listener 80: default action redirect to HTTPS 443 with status 301
2. Listener 443 with ACM certificate and a rule `path-pattern /api/*` forwarding to the api target group
3. Default action forward to the web target group
4. Check rule priority order
5. `aws elbv2 describe-rules --listener-arn ...`

**Why →** Rules evaluate in priority order, so specific paths must come before the catch-all.

**Interview tip →** Mention host-header rules for multi-tenant routing.

</details>

### Q70. You need zero-downtime AZ failover and a way to rehearse it. What tool do you use? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Application Recovery Controller zonal shift moves traffic away from an impaired AZ.

**What I'd do →**

1. Enable zonal shift on the ALB or NLB
2. Run `aws arc-zonal-shift start-zonal-shift --resource-identifier <alb-arn> --away-from eu-west-1a --expires-in 1h --comment drill`
3. Watch capacity and latency in the other AZs
4. Pre-scale so remaining AZs can absorb the load
5. Practice quarterly

**Why →** Shifting traffic is reversible and quick, and rehearsal exposes under-provisioned AZs.

**Interview tip →** Static stability: do not depend on control plane actions during a failure.

</details>

### Q71. An EC2 instance cannot be launched with "InsufficientInstanceCapacity". What are your options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The AZ is out of that instance type right now.

**What I'd do →**

1. Retry in another AZ or subnet
2. Use a different instance type or size, or `price-capacity-optimized` flexibility
3. For critical capacity use On-Demand Capacity Reservations
4. Use attribute-based instance type selection in the ASG
5. Don't hardcode one type

**Why →** Flexibility across types and AZs is the best protection against capacity errors.

**Interview tip →** Reserve capacity ahead of events like a launch.

</details>

### Q72. Choose between EBS volume types for a database needing 10,000 IOPS consistently. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** gp3 for most; io2 when sustained high IOPS and durability matter.

**What I'd do →**

1. gp3 lets you provision IOPS (up to 16,000) and throughput independent of size
2. io2 Block Express goes to 256,000 IOPS with 99.999 percent durability
3. Check the instance limit too, since EBS-optimized bandwidth caps
4. Modify live: `aws ec2 modify-volume --volume-id vol-0abc --volume-type gp3 --iops 10000`

**Why →** gp3 is about 20 percent cheaper than gp2 and decouples performance from size.

**Interview tip →** Mention instance-level EBS throughput as a hidden cap.

</details>

### Q73. Design a self-healing stateful service on EC2, such as a leader-based cluster, with EBS volumes. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** EBS is AZ-bound, so replacement logic must re-attach or rebuild in the same AZ.

**What I'd do →**

1. One ASG per AZ with min and max 1, or lifecycle hooks that attach the volume
2. Tag volumes and attach with a user data script using the instance's AZ
3. Prefer replicated data (rebuild from peers) over reattachment
4. Use EBS snapshots through Data Lifecycle Manager
5. Alarm on replication lag

**Why →** Rebuild-from-peer avoids long reattach waits and corruption risk from forced detach.

**Interview tip →** If possible, push state to a managed service.

</details>

### Q74. What is a placement group and when would you use one? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Placement groups control how instances are placed on hardware.

**What I'd do →**

1. Cluster: low latency, high throughput, one AZ (HPC, tightly coupled)
2. Spread: each instance on distinct hardware, max 7 per AZ, for critical small groups
3. Partition: groups of instances on distinct racks, for Kafka, Cassandra
4. Create: `aws ec2 create-placement-group --group-name hpc --strategy cluster`

**Why →** The strategy trades off latency against blast-radius.

**Interview tip →** Cluster is a single-AZ risk.

</details>

### Q75. Pick the target type for an ALB serving Lambda, instances and IPs in the same system. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Target groups can be instance, IP or Lambda type.

**What I'd do →**

1. Instance for ASG-managed hosts
2. IP for containers on awsvpc networking, peered or on-prem addresses
3. Lambda for serverless endpoints, invoked synchronously
4. Weighted forwarding lets you migrate gradually between types

**Why →** One ALB can front mixed backends during migrations.

**Interview tip →** ECS with awsvpc must use IP targets.

</details>

## S3, RDS and DynamoDB

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. You need to host a static website with HTTPS and low latency worldwide. Which services do you use? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** S3 as private origin, CloudFront as the public face.

**What I'd do →**

1. Create a private bucket, no static website hosting endpoint required
2. Create a CloudFront distribution with Origin Access Control to the bucket
3. Attach an ACM certificate from us-east-1 and add the domain as alternate name
4. Route 53 alias record to the distribution
5. Default root object `index.html`, plus custom error response for SPA routes

**Why →** OAC keeps the bucket private so content is only reachable through the CDN.

**Interview tip →** Note OAC replaced the older Origin Access Identity.

</details>

### Q77. Uploaded files in your bucket must be deleted after 90 days and old versions after 30. How do you do that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** S3 lifecycle rules.

**What I'd do →**

1. Enable versioning
2. Add a rule: expire current objects after 90 days, `NoncurrentVersionExpiration` 30 days, and abort incomplete multipart uploads after 7 days
3. Apply: `aws s3api put-bucket-lifecycle-configuration --bucket app-uploads --lifecycle-configuration file://lifecycle.json`
4. Verify with `get-bucket-lifecycle-configuration`

**Why →** Lifecycle is free automation that stops storage costs creeping up.

**Interview tip →** Mention incomplete multipart uploads as hidden cost.

</details>

### Q78. Someone deleted objects from a versioned bucket. How do you recover and prevent repeats? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** With versioning on, a delete adds a delete marker; the data is still there.

**What I'd do →**

1. List versions: `aws s3api list-object-versions --bucket b --prefix path/`
2. Remove the delete marker: `aws s3api delete-object --bucket b --key path/file --version-id <marker-id>`
3. Turn on MFA delete or Object Lock for critical data
4. Add a bucket policy denying `s3:DeleteObjectVersion` to most principals
5. Consider replication to a second account

**Why →** Deleting the marker restores the previous version instantly.

**Interview tip →** Without versioning, deletion is permanent; ask whether it was enabled.

</details>

### Q79. Millions of small objects are being uploaded and S3 returns 503 Slow Down. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-prefix request rates (3,500 PUT and 5,500 GET per second) were exceeded.

**What I'd do →**

1. Spread keys across multiple prefixes
2. Add retries with exponential backoff and jitter in the SDK
3. Batch small objects into larger ones
4. Request-rate scale-up is automatic over minutes, so ramp traffic gradually
5. Check CloudWatch `5xxErrors` request metrics

**Why →** S3 scales per prefix, so more prefixes means more throughput.

**Interview tip →** Backoff plus jitter is the universal AWS client answer.

</details>

### Q80. Large 50 GB files upload slowly from remote offices to S3. How do you speed it up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use multipart parallel uploads and edge acceleration.

**What I'd do →**

1. Multipart is automatic in the CLI: tune `aws configure set default.s3.multipart_chunksize 64MB` and `max_concurrent_requests 20`
2. Enable Transfer Acceleration: `aws s3api put-bucket-accelerate-configuration --bucket b --accelerate-configuration Status=Enabled`
3. Use the accelerate endpoint `b.s3-accelerate.amazonaws.com`
4. For bulk offline use Snowball

**Why →** Parallel parts use multiple connections, and acceleration shortens the public hop via edge locations.

**Interview tip →** Acceleration has per-GB cost; test with the speed comparison tool first.

</details>

### Q81. Design a cross-region DR strategy for a critical S3 bucket with a 15-minute RPO. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** S3 Replication Time Control is the only option with an SLA on replication time.

**What I'd do →**

1. Enable versioning on both buckets
2. Configure CRR with S3 RTC (99.99 percent of objects within 15 minutes)
3. Replicate delete markers if desired and KMS-encrypted objects with the destination key
4. Monitor `ReplicationLatency` and failed replication events
5. Use Multi-Region Access Points for failover routing

**Why →** Normal replication is best-effort, RTC makes it contractual and monitorable.

**Interview tip →** Mention replication does not cover pre-existing objects without batch replication.

</details>

### Q82. An analytics team needs query access to 10 TB of logs in S3 without loading a warehouse. What do you propose? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Athena over partitioned columnar data.

**What I'd do →**

1. Convert JSON logs to Parquet with Glue or Athena CTAS
2. Partition by `year/month/day` and use Glue Data Catalog
3. Use partition projection to avoid crawlers
4. Query: `SELECT count(*) FROM logs WHERE dt = '2026-11-01'`
5. Control cost with workgroup data-scanned limits

**Why →** Athena charges per TB scanned, so columnar format and partitions cut cost by 90 percent or more.

**Interview tip →** Always mention Parquet plus partitioning.

</details>

### Q83. Choose between RDS and running a database on EC2. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** RDS manages backups, patching and failover; EC2 gives control at the cost of operations.

**What I'd do →**

1. Use RDS or Aurora unless you need unsupported extensions or OS access
2. Enable Multi-AZ for production
3. Automated backups with a 7 to 35 day retention
4. Use parameter groups instead of editing files
5. Go EC2 only with a specific need and a team to run it

**Why →** Operational toil is the real cost of self-managed databases.

**Interview tip →** RDS Custom exists for the in-between case.

</details>

### Q84. Your RDS primary failed over to the standby. What happened and what must the app do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** In Multi-AZ, RDS flips the DNS endpoint CNAME to the standby, usually in 60 to 120 seconds.

**What I'd do →**

1. App must reconnect: use the endpoint DNS name, never an IP
2. Set low DNS TTL awareness and connection retry
3. Check events: `aws rds describe-events --source-type db-instance --source-identifier mydb`
4. Test failover: `aws rds reboot-db-instance --db-instance-identifier mydb --force-failover`
5. Use RDS Proxy to keep connections stable

**Why →** The standby is synchronously replicated, so no data is lost but connections drop.

**Interview tip →** Multi-AZ is for HA, not for read scaling (except the Multi-AZ cluster option).

</details>

### Q85. The database CPU is high due to read traffic. How do you scale reads? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Offload reads to replicas and cache.

**What I'd do →**

1. Create read replicas: `aws rds create-db-instance-read-replica --db-instance-identifier mydb-r1 --source-db-instance-identifier mydb`
2. Route read queries with a reader endpoint (Aurora) or app-level routing
3. Add ElastiCache for hot keys
4. Check slow queries with Performance Insights
5. Watch `ReplicaLag`

**Why →** Replicas are asynchronous, so reads can be stale; choose consistency needs deliberately.

**Interview tip →** Fix bad queries before adding hardware.

</details>

### Q86. Plan a major-version RDS PostgreSQL upgrade with minimal downtime. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Blue/green deployments make a staged copy that you switch over.

**What I'd do →**

1. Create: `aws rds create-blue-green-deployment --blue-green-deployment-name pg-upgrade --source <arn> --target-engine-version 17.2`
2. Test the green environment with real queries and extension checks
3. Pause writes or accept a brief switchover, then `switchover-blue-green-deployment`
4. Keep blue for rollback until verified
5. Run `ANALYZE` after upgrade

**Why →** Switchover typically takes under a minute and avoids in-place upgrade risk.

**Interview tip →** Mention testing extensions and parameter compatibility first.

</details>

### Q87. Lambda functions exhaust database connections on RDS under load. Fix it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each concurrent Lambda opens its own connection.

**What I'd do →**

1. Put RDS Proxy in front and connect to the proxy endpoint
2. Create it with a Secrets Manager secret and the DB SG
3. Reuse connections outside the handler
4. Limit reserved concurrency
5. Monitor `DatabaseConnections`

**Why →** The proxy pools and multiplexes connections so bursts do not hit `max_connections`.

**Interview tip →** Proxy needs IAM auth or secrets and runs in the VPC.

</details>

### Q88. DynamoDB throws ProvisionedThroughputExceededException for one user's activity. Why, and what's the fix? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A hot partition: one partition key takes disproportionate traffic.

**What I'd do →**

1. Look at CloudWatch Contributor Insights for the hot key
2. Redesign the key to spread load, such as adding a shard suffix `user#17`
3. Switch to on-demand mode if the pattern is unpredictable
4. Add DAX for read-heavy hotspots
5. Add SDK backoff

**Why →** Throughput is divided among partitions, so even keys matter more than total capacity.

**Interview tip →** Adaptive capacity helps but does not fix a skewed design.

</details>

### Q89. When do you choose DynamoDB over RDS? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Known access patterns, massive scale, single-digit-millisecond latency, little ops.

**What I'd do →**

1. Use DynamoDB for key-value or single-table designs like sessions, carts, IoT
2. Use RDS or Aurora for relational joins, ad hoc queries and transactions across entities
3. Model DynamoDB around queries, using GSIs for alternate access
4. Prove the access patterns before committing

**Why →** DynamoDB is hard to retrofit with new query patterns, so choose it on certainty.

**Interview tip →** Say "access patterns first".

</details>

### Q90. Design a globally available DynamoDB application with active-active writes. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Global tables replicate across regions with last-writer-wins conflict resolution.

**What I'd do →**

1. Create the table with streams and add replicas: `aws dynamodb update-table --table-name orders --replica-updates 'Create={RegionName=eu-west-1}'`
2. Route users with Route 53 latency routing
3. Design writes to avoid conflicts, such as region-owned items
4. Monitor `ReplicationLatency`
5. Use multi-Region strong consistency option only when needed

**Why →** Replication is asynchronous and conflicts resolve by timestamp, so data modeling prevents lost updates.

**Interview tip →** Mention idempotent writes.

</details>

### Q91. You must restore a DynamoDB table to 14:30 yesterday. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Point-in-time recovery restores to a new table.

**What I'd do →**

1. Make sure PITR is enabled: `aws dynamodb update-continuous-backups --table-name t --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true`
2. Restore: `aws dynamodb restore-table-to-point-in-time --source-table-name t --target-table-name t-restore --restore-date-time 2026-11-08T14:30:00Z`
3. Copy indexes, TTL, alarms and IAM manually
4. Swap application to the new table

**Why →** PITR keeps 35 days of continuous backups and restores never overwrite in place.

**Interview tip →** Remember PITR is off by default.

</details>

### Q92. The RDS storage is 95 percent full on a Friday. What is the safe response? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Storage autoscaling or manual modify can extend online.

**What I'd do →**

1. Modify: `aws rds modify-db-instance --db-instance-identifier mydb --allocated-storage 500 --apply-immediately`
2. Enable storage autoscaling with `--max-allocated-storage 2000`
3. Find bloat or giant logs and purge
4. Note the six-hour cooldown between modifications
5. Alert on `FreeStorageSpace` at 20 percent

**Why →** Storage can grow online but never shrink, so size growth with intent.

**Interview tip →** Mention the cooldown trap.

</details>

### Q93. Aurora versus RDS PostgreSQL: how do you decide? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Aurora gives faster failover, shared storage and up to 15 replicas, at a higher price.

**What I'd do →**

1. Pick Aurora for heavy read scale, quick failover (about 30 seconds), serverless v2 elasticity or global database
2. Pick RDS PostgreSQL for cost and plain engine behavior
3. Compare IOPS charges against Aurora I/O-Optimized
4. Benchmark with production-like load

**Why →** Aurora decouples compute and storage, replicas read the same volume with tiny lag.

**Interview tip →** Don't claim Aurora is always better; cost matters.

</details>

### Q94. An S3 bucket receiving PUTs must trigger processing for each file. What is the design? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** S3 event notifications to SQS or EventBridge, then workers.

**What I'd do →**

1. Configure notifications on `s3:ObjectCreated:*` to an SQS queue
2. Lambda or ECS workers poll the queue
3. Add a DLQ with `maxReceiveCount` 5
4. Use EventBridge instead for multiple consumers and filtering
5. Make processing idempotent since events are at-least-once

**Why →** A queue buffers bursts and decouples upload rate from processing rate.

**Interview tip →** S3 to Lambda directly is fine for small flows but lacks buffering.

</details>

### Q95. Share a private S3 object with an external customer for one hour. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use a presigned URL.

**What I'd do →**

1. Generate: `aws s3 presign s3://bucket/report.pdf --expires-in 3600`
2. Sign with a role whose credentials outlive the URL (role session expiry caps URL life)
3. For uploads use a presigned PUT or POST policy with size limits
4. Log access with S3 server access logs or CloudTrail data events

**Why →** The URL carries the signer's permissions temporarily without making the object public.

**Interview tip →** URLs signed with temporary credentials die when those credentials expire.

</details>

### Q96. Design a backup strategy across accounts for RDS, EBS and DynamoDB with ransomware protection. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Centralized, immutable, cross-account backups.

**What I'd do →**

1. Use AWS Backup with a backup plan and tag-based selection
2. Copy to a vault in a separate backup account and region
3. Enable Backup Vault Lock in compliance mode for immutability
4. Restrict vault access policy
5. Run restore tests regularly

**Why →** An attacker with workload-account admin cannot delete locked copies in another account.

**Interview tip →** Untested backups are a hope, not a plan.

</details>

### Q97. The application writes and then immediately reads from S3 and sometimes gets stale data. Is that possible? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** S3 has been strongly consistent for reads after writes since December 2020.

**What I'd do →**

1. Check whether reads pass through CloudFront cache
2. Check replication destinations, which are eventual
3. Check app-level caching
4. Verify with `aws s3api head-object` right after the PUT

**Why →** The staleness almost always comes from a layer above S3.

**Interview tip →** If someone says S3 is eventually consistent, correct them politely.

</details>

### Q98. How do you encrypt data at rest in S3 and who manages the keys? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** SSE-S3 by default; SSE-KMS gives key-level audit and control.

**What I'd do →**

1. Default encryption SSE-S3 applies to new objects automatically
2. For regulated data set SSE-KMS with a customer managed key
3. Enable S3 Bucket Keys to cut KMS request cost
4. Use `aws s3api put-bucket-encryption`
5. Grant `kms:Decrypt` to consumers

**Why →** KMS adds an audit trail and revocable key access.

**Interview tip →** SSE-C and client-side encryption exist but add key handling burden.

</details>

### Q99. Plan a migration of a 5 TB on-prem Oracle database to AWS with minimal downtime. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** DMS with ongoing replication after schema conversion.

**What I'd do →**

1. Assess with Schema Conversion Tool and decide target (RDS Oracle, or Aurora PostgreSQL if converting)
2. Create the target, load schema, then start a DMS task with full load plus CDC
3. Validate rows with DMS data validation
4. Cut over: stop writes, wait for lag zero, repoint the app
5. Keep reverse replication for rollback

**Why →** CDC keeps the target within seconds of the source so cutover is minutes.

**Interview tip →** Mention a network path (DX or VPN) sized for the initial load.

</details>

## Route 53, CloudFront and Messaging

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q100. You want users routed to the nearest healthy region. Which Route 53 policy do you use? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Latency routing with health checks.

**What I'd do →**

1. Create latency records for `api.example.com` in eu-west-1 and us-east-1
2. Attach Route 53 health checks to each
3. Use alias records to the regional ALBs with `EvaluateTargetHealth` true
4. Test: `dig api.example.com` from different locations
5. Keep TTL low, 60 seconds

**Why →** Latency routing picks the lowest-latency region, health checks remove failed ones.

**Interview tip →** Latency is measured to AWS regions, not to the user's exact location.

</details>

### Q101. Primary region fails; how do you fail over DNS in a controlled way? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Failover routing with a primary and secondary record.

**What I'd do →**

1. Primary record with a health check on a deep endpoint `/healthz`
2. Secondary record pointing to the DR region
3. Alias with evaluate target health
4. For explicit control use Application Recovery Controller routing controls
5. Remember clients cache DNS beyond TTL

**Why →** Health-checked failover is automatic; ARC gives a manual, safe switch that avoids flapping.

**Interview tip →** DNS failover is never instant; mention TTL and resolver caching.

</details>

### Q102. What is the difference between a CNAME and an alias record in Route 53? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Alias is a Route 53 extension that works at the zone apex and is free for AWS targets.

**What I'd do →**

1. CNAME cannot exist at `example.com` apex
2. Alias to ALB, CloudFront, S3 website, API Gateway, with no query charge
3. Alias records resolve to A or AAAA, so health tracking is built in
4. Create via `aws route53 change-resource-record-sets`

**Why →** Alias gives apex support and removes an extra DNS hop.

**Interview tip →** Use alias for AWS resources, CNAME for external ones.

</details>

### Q103. Roll out a new version to 5 percent of users using DNS. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Weighted routing.

**What I'd do →**

1. Create two records with weights 95 and 5 and different `SetIdentifier`
2. Attach health checks
3. Increase weight over time
4. Monitor errors per version
5. Prefer ALB weighted target groups for finer control because DNS caching blurs percentages

**Why →** DNS weights are probabilities per resolver query, not per user.

**Interview tip →** Say DNS canaries are coarse; ALB or CloudFront continuous deployment is better.

</details>

### Q104. CloudFront serves stale content after you deployed a new JS bundle. How do you handle cache correctly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hash filenames for assets and invalidate only the entry HTML.

**What I'd do →**

1. Build with content hashes like `app.3f9a1c.js` and long TTL `Cache-Control: max-age=31536000, immutable`
2. Keep `index.html` at short TTL
3. Invalidate: `aws cloudfront create-invalidation --distribution-id E123 --paths /index.html`
4. Use a cache policy rather than legacy settings

**Why →** Versioned filenames avoid invalidations, which are slow and cost beyond the free quota.

**Interview tip →** Mention first 1,000 paths per month are free.

</details>

### Q105. CloudFront returns 403 from an S3 origin. List the checks. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** It's usually OAC or bucket policy, or a missing object.

**What I'd do →**

1. Verify the bucket policy allows `cloudfront.amazonaws.com` with `AWS:SourceArn` equal to the distribution ARN
2. Check the origin uses OAC and the correct bucket regional domain
3. Check the object key exists, since missing keys return 403 without `s3:ListBucket`
4. Check SSE-KMS key policy allows CloudFront
5. Inspect with `curl -I https://d111.cloudfront.net/index.html`

**Why →** S3 hides existence of keys from principals lacking list permission, producing 403 instead of 404.

**Interview tip →** KMS-encrypted origins need extra OAC key policy.

</details>

### Q106. Protect an origin so that only CloudFront can reach your ALB. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use the managed prefix list and a secret header.

**What I'd do →**

1. In the ALB SG allow 443 from the `com.amazonaws.global.cloudfront.origin-facing` prefix list
2. Have CloudFront add a custom header `X-Origin-Verify: <secret>`
3. ALB listener rule forwards only when the header matches, else 403
4. Rotate the secret with Secrets Manager
5. Or use VPC origins for private ALBs

**Why →** The prefix list stops direct IP access, the header stops other CloudFront distributions.

**Interview tip →** VPC origins removes the public ALB entirely.

</details>

### Q107. Add WAF protection against SQL injection and bot floods for a public app. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS WAF on CloudFront or ALB with managed rule groups and rate limits.

**What I'd do →**

1. Create a web ACL with AWS managed rules `AWSManagedRulesCommonRuleSet` and `AWSManagedRulesSQLiRuleSet`
2. Add a rate-based rule such as 2,000 requests per 5 minutes per IP
3. Start in Count mode, review logs, then Block
4. Send logs to S3 or CloudWatch
5. Add Shield Advanced for DDoS needs

**Why →** Count mode prevents false positives from blocking real customers on day one.

**Interview tip →** Mention WAF logging to inspect matches.

</details>

### Q108. A queue consumer crashes mid-message and the message is processed twice. Is that a bug? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** SQS standard is at-least-once, so duplicates are expected.

**What I'd do →**

1. Make consumers idempotent using a dedupe key in DynamoDB with a conditional write
2. Tune `VisibilityTimeout` above processing time
3. Use FIFO with `MessageDeduplicationId` if ordering and dedupe are needed
4. Configure a DLQ
5. Delete only after success: `aws sqs delete-message`

**Why →** Visibility timeout hides the message during processing, crashes make it reappear.

**Interview tip →** Idempotency is the answer to every at-least-once system.

</details>

### Q109. Messages keep failing and blocking others in a queue. How do you isolate poison messages? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use a dead-letter queue with a redrive path.

**What I'd do →**

1. Set `RedrivePolicy` with `maxReceiveCount` 5 to a DLQ
2. Alarm on `ApproximateNumberOfMessagesVisible` on the DLQ
3. Inspect messages, fix the bug, then redrive: `aws sqs start-message-move-task --source-arn <dlq-arn>`
4. Keep DLQ retention longer than main (14 days)

**Why →** DLQ keeps healthy flow moving and preserves bad messages for diagnosis.

**Interview tip →** Mention FIFO message group blocking.

</details>

### Q110. One event must fan out to email, an SQS worker and a Lambda. What do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SNS topic with multiple subscriptions.

**What I'd do →**

1. Create the topic and subscribe email, SQS queue and Lambda
2. Allow SNS to send to the queue via the queue policy
3. Add subscription filter policies by message attribute
4. Enable raw message delivery to SQS
5. Use FIFO topics if ordering matters

**Why →** Pub/sub decouples publisher from consumers, so adding one needs no publisher change.

**Interview tip →** Filter policies cut cost by delivering only relevant messages.

</details>

### Q111. Choose between SQS, SNS, EventBridge and Kinesis for an order platform. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** They suit different delivery shapes.

**What I'd do →**

1. SQS: buffering and work queues, one consumer group
2. SNS: simple high-throughput fan-out
3. EventBridge: content-based routing, schema registry, SaaS and cross-account events
4. Kinesis or MSK: ordered high-volume streams with replay
5. Combine, such as EventBridge to SQS for durable consumers

**Why →** Choose by routing needs and replay requirement.

**Interview tip →** Replay and ordering point to streams, not queues.

</details>

### Q112. You need a nightly job that runs at 02:00 UTC and triggers an ECS task. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** EventBridge Scheduler is the managed cron.

**What I'd do →**

1. Create the schedule: `aws scheduler create-schedule --name nightly --schedule-expression "cron(0 2 * * ? *)" --target '{"Arn":"<cluster-arn>","RoleArn":"<role>","EcsParameters":{...}}' --flexible-time-window Mode=OFF`
2. Give the role `ecs:RunTask` and `iam:PassRole`
3. Add retry and a DLQ
4. Alarm on failure

**Why →** No server to maintain, and time zones and retries are supported.

**Interview tip →** Mention `iam:PassRole` as the usual gotcha.

</details>

### Q113. Design an event-driven order pipeline that is resilient to downstream outage. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Durable buffering between each stage.

**What I'd do →**

1. API Gateway to Lambda writes the order to DynamoDB and emits an event to EventBridge
2. EventBridge rules route to SQS queues per consumer (billing, inventory, email)
3. Consumers read with batch size and partial batch failure response
4. DLQs and alarms on age of oldest message
5. Use outbox pattern with DynamoDB Streams for exactly-recorded publishing

**Why →** Queues absorb outages, so a billing downtime delays but never loses orders.

**Interview tip →** Mention `ApproximateAgeOfOldestMessage` as the best queue health alarm.

</details>

### Q114. Lambda reading SQS processes the same failed batch repeatedly. How to fix partial failures? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Without partial batch response, one bad record retries the whole batch.

**What I'd do →**

1. Enable `ReportBatchItemFailures` on the event source mapping
2. Return `{"batchItemFailures":[{"itemIdentifier":"<messageId>"}]}`
3. Set a DLQ on the queue
4. Set visibility timeout at least 6 times function timeout
5. Reduce batch size if poison messages are common

**Why →** Only failed records return to the queue.

**Interview tip →** AWS recommends 6x function timeout for visibility.

</details>

### Q115. How do you make a Route 53 private hosted zone resolve names inside two VPCs? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Associate the zone with each VPC.

**What I'd do →**

1. `aws route53 associate-vpc-with-hosted-zone --hosted-zone-id Z123 --vpc VPCRegion=eu-west-1,VPCId=vpc-0abc`
2. For cross-account use `create-vpc-association-authorization` first
3. Enable `enableDnsSupport` and `enableDnsHostnames` on the VPC
4. Test with `dig` from an instance

**Why →** PHZ answers only for associated VPCs, so scope is controlled.

**Interview tip →** Mention split-horizon DNS using the same zone name public and private.

</details>

### Q116. Your website domain must serve both `example.com` and `www.example.com` over HTTPS via CloudFront. What do you configure? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both names as alternate domain names with a matching cert.

**What I'd do →**

1. Request ACM cert in us-east-1 for both names, validate by DNS
2. Add CNAMEs in the distribution
3. Route 53 alias A and AAAA records for both
4. Redirect www to apex with a CloudFront Function
5. Use TLS 1.2 minimum policy

**Why →** CloudFront can only use ACM certificates from us-east-1.

**Interview tip →** Don't forget AAAA records for IPv6.

</details>

### Q117. Reduce latency and origin load for a dynamic API globally. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cache what is safe, and use CloudFront as a global network accelerator even for uncacheable calls.

**What I'd do →**

1. Define cache policies per path: short TTL for GETs, no cache for POST
2. Add Origin Shield in the region closest to the origin
3. Use persistent connections to origin and HTTP/3 to clients
4. Compress with Brotli
5. Consider Global Accelerator for non-HTTP or static IP needs

**Why →** Edge termination of TLS cuts handshake latency even without cache hits.

**Interview tip →** Cache key design is where most wins and bugs live.

</details>

### Q118. A signed-URL protected video is playable by anyone with the link. How do you restrict it more? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use CloudFront signed cookies or URLs with short expiry and IP or policy limits.

**What I'd do →**

1. Create a key group with public key, trusted key groups on the behavior
2. Generate a signed URL with custom policy including expiry and IP range
3. Keep expiry minutes for streaming sessions through cookies
4. Add geo restriction if needed
5. Rotate keys through a new key group

**Why →** Time-limited signatures minimize reuse of leaked links.

**Interview tip →** Use key groups, not the deprecated root account key pairs.

</details>

### Q119. An SNS to HTTP endpoint subscription does not get messages. What do you check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Confirmation or delivery failure.

**What I'd do →**

1. Check the subscription is confirmed (not `PendingConfirmation`)
2. Endpoint must respond 2xx quickly and be reachable publicly over HTTPS
3. Review delivery status logs in CloudWatch
4. Set a redrive DLQ on the subscription
5. Check filter policy not excluding the message

**Why →** SNS only retries according to a delivery policy then drops without a DLQ.

**Interview tip →** Subscription-level DLQ is essential.

</details>

### Q120. What is the purpose of Route 53 health checks and what can they monitor? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** They probe endpoints and drive DNS failover.

**What I'd do →**

1. HTTP, HTTPS, TCP checks from global checkers
2. Calculated health checks combining others
3. CloudWatch alarm health checks for private resources
4. Alarm via SNS on `HealthCheckStatus`

**Why →** They let DNS remove failed endpoints automatically.

**Interview tip →** Private endpoints need a CloudWatch alarm-based check.

</details>

### Q121. Design API ingress for 10,000 requests per second with throttling and auth. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** API Gateway or ALB depending on features and cost.

**What I'd do →**

1. REST or HTTP API Gateway with Cognito or JWT authorizer, usage plans and throttling
2. For high steady volume compare cost with ALB plus Lambda or ECS
3. Add WAF
4. Default quota is 10,000 rps per region; request increases early
5. Add caching and per-client throttling keys

**Why →** API Gateway per-request pricing makes ALB cheaper at sustained high traffic.

**Interview tip →** Mention HTTP API is cheaper than REST API when features suffice.

</details>

### Q122. An EventBridge rule is not triggering a target. Quick checklist. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pattern mismatch or target permission.

**What I'd do →**

1. Check the event bus is the right one (default vs custom)
2. Test the pattern with `aws events test-event-pattern`
3. Check target resource policy or role
4. View `FailedInvocations` and `Invocations` metrics
5. Add a DLQ to the target

**Why →** Most issues are pattern typos or missing invoke permissions.

**Interview tip →** Use `TestEventPattern` before deploying.

</details>

### Q123. Build a DNS-based blue/green cutover for an internal service with rollback. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Weighted alias records plus low TTL.

**What I'd do →**

1. Lower TTL to 30 seconds a day before
2. Add green record weight 0, test via direct name
3. Shift 10 percent, 50 percent, 100 percent
4. Keep blue for rollback
5. Restore TTL after

**Why →** Lowering TTL ahead of time is what makes rollback quick.

**Interview tip →** Plan TTL changes early.

</details>

### Q124. How do you stop a public SNS topic or SQS queue from accepting messages from anyone? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Resource policies must restrict principals.

**What I'd do →**

1. Review queue policy `Principal` and add `aws:SourceArn` conditions
2. Remove `"Principal":"*"` unless conditioned
3. Use IAM Access Analyzer to find external access
4. Enable encryption with KMS

**Why →** Open resource policies are a common data leak vector.

**Interview tip →** Mention Access Analyzer for external findings.

</details>

## ECS, EKS, Lambda and ECR

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q125. ECS on Fargate or on EC2: how do you choose? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fargate removes host management; EC2 gives control and cost optimization at scale.

**What I'd do →**

1. Choose Fargate for most services, bursty workloads and small teams
2. Choose EC2 for GPUs, large steady fleets with Savings Plans, privileged containers or daemon sets
3. Use capacity providers to mix both
4. Compare per vCPU-hour cost

**Why →** Fargate trades roughly 20 to 30 percent premium for no patching.

**Interview tip →** Say "Fargate by default, EC2 when numbers justify".

</details>

### Q126. Deploy a containerized API on ECS behind an ALB. What are the building blocks? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Task definition, service, target group, ALB.

**What I'd do →**

1. Push the image to ECR
2. Register the task definition with `awsvpc` network mode, CPU and memory, log driver `awslogs`, and task role
3. Create a service with desired count 3 across subnets and attach it to the ALB target group (IP type)
4. Configure health check path and deployment circuit breaker
5. `aws ecs update-service --cluster prod --service api --force-new-deployment`

**Why →** The task role grants app permissions, the execution role lets ECS pull images and write logs.

**Interview tip →** Know task role versus execution role.

</details>

### Q127. An ECS service deployment is stuck, tasks start and stop repeatedly. How do you debug? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The task fails health checks or crashes.

**What I'd do →**

1. `aws ecs describe-services` and read the events
2. `aws ecs describe-tasks --tasks <arn>` for `stoppedReason` and exit code
3. Check CloudWatch logs of the container
4. Verify the health check path and grace period `healthCheckGracePeriodSeconds`
5. Enable the deployment circuit breaker with rollback

**Why →** The stopped reason usually names OOM (exit 137) or image pull errors directly.

**Interview tip →** Exit code 137 equals OOM kill.

</details>

### Q128. Roll out a new ECS service version with canary and automatic rollback. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use CodeDeploy blue/green or ECS native blue/green.

**What I'd do →**

1. Two target groups on the ALB
2. Deployment config `CodeDeployDefault.ECSCanary10Percent5Minutes`
3. CloudWatch alarms on 5xx and latency attached to the deployment group
4. Automatic rollback on alarm
5. Test listener for pre-traffic validation hooks

**Why →** Alarms trigger automatic reversal without a human.

**Interview tip →** Mention ECS native blue/green as the newer path.

</details>

### Q129. EKS pods need access to S3 without node-level permissions. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** IAM roles for service accounts or EKS Pod Identity.

**What I'd do →**

1. With Pod Identity: install the agent add-on, then `aws eks create-pod-identity-association --cluster-name prod --namespace app --service-account s3-reader --role-arn arn:aws:iam::111111111111:role/s3-reader`
2. Annotate nothing, pods use the SDK default chain
3. Role trusts `pods.eks.amazonaws.com`
4. Verify with `aws sts get-caller-identity` inside the pod

**Why →** Per-pod roles enforce least privilege instead of every pod inheriting the node role.

**Interview tip →** IRSA still works; Pod Identity is simpler across clusters.

</details>

### Q130. Upgrade an EKS cluster from 1.31 to 1.33 safely. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Only one minor version at a time, control plane first.

**What I'd do →**

1. Check deprecated APIs with `kubent` or `pluto`
2. Update add-ons compatibility
3. Upgrade control plane: `aws eks update-cluster-version --name prod --kubernetes-version 1.32`
4. Upgrade managed node groups with surge settings
5. Repeat for 1.33
6. Verify PDBs exist to avoid outages during drains

**Why →** Skipping versions is unsupported and API removals cause hard failures.

**Interview tip →** Mention the extended support cost for old versions.

</details>

### Q131. Design node scaling for EKS with mixed Spot and On-Demand and fast scale-up. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Karpenter replaces fixed node groups with just-in-time right-sized nodes.

**What I'd do →**

1. Install Karpenter with a `NodePool` allowing multiple instance families and `spot` plus `on-demand` capacity types
2. Set `disruption` consolidation policy `WhenEmptyOrUnderutilized`
3. Keep a small managed node group for system pods
4. Use PDBs and `topologySpreadConstraints`
5. Handle interruption via SQS queue integration

**Why →** Karpenter picks the cheapest fitting instance in seconds rather than scaling fixed groups.

**Interview tip →** Mention bin-packing and consolidation.

</details>

### Q132. An EKS pod is stuck in Pending with "0/3 nodes are available: Insufficient cpu". What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Capacity or requests are too high.

**What I'd do →**

1. `kubectl describe pod` for the scheduler message
2. `kubectl top nodes` and review requests versus usage
3. Scale the node group or let the autoscaler add nodes
4. Right-size over-large requests
5. Check taints and affinity

**Why →** Scheduling uses requests, not actual usage.

**Interview tip →** Requests drive bin-packing and cost.

</details>

### Q133. Lambda function times out after 3 seconds calling an external API. What's wrong? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default timeout is 3 seconds, or the function is in a VPC without egress.

**What I'd do →**

1. Increase timeout: `aws lambda update-function-configuration --function-name f --timeout 30`
2. If VPC-attached, ensure NAT or endpoint access
3. Check logs for the exact duration with REPORT lines
4. Set client timeouts shorter than the function timeout

**Why →** VPC-attached Lambdas have no internet unless routed via NAT.

**Interview tip →** Most "mystery timeouts" are VPC egress.

</details>

### Q134. Lambda cold starts hurt latency for a user-facing API. Options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reduce init time, or pre-warm capacity.

**What I'd do →**

1. Enable Provisioned Concurrency on an alias
2. Or use SnapStart for Java, Python and .NET
3. Shrink package, lazy load libraries, use ARM64
4. Avoid VPC-ENI penalties (now small)
5. Right-size memory since CPU scales with it

**Why →** Provisioned concurrency keeps environments initialized at a cost.

**Interview tip →** Memory tuning often cuts duration and cost together.

</details>

### Q135. Your Lambda scaled so much it overwhelmed a downstream database. How do you cap it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reserved concurrency limits the function.

**What I'd do →**

1. `aws lambda put-function-concurrency --function-name f --reserved-concurrent-executions 50`
2. Buffer with SQS and set event source `MaximumConcurrency`
3. Add RDS Proxy
4. Alarm on `Throttles` and `ConcurrentExecutions`

**Why →** Reserved concurrency is both a guarantee and a ceiling.

**Interview tip →** Account concurrency default is 1,000 per region.

</details>

### Q136. An ECR repo grows to 2 TB of images. How do you control it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Lifecycle policies expire old images.

**What I'd do →**

1. Create a lifecycle policy keeping the last 20 tagged `release-*` and expiring untagged after 7 days
2. `aws ecr put-lifecycle-policy --repository-name app --lifecycle-policy-text file://policy.json`
3. Use `--image-tag-mutability IMMUTABLE` for release tags
4. Enable image scanning on push

**Why →** Storage is billed per GB-month, and old layers accumulate quietly.

**Interview tip →** Preview with the dry-run lifecycle preview.

</details>

### Q137. Multiple accounts must pull images from one central ECR. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repository policy plus replication if cross-region.

**What I'd do →**

1. Add a repository policy granting `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer` to the account or org ID
2. The pulling role also needs `ecr:GetAuthorizationToken`
3. Use ECR replication rules for regional copies
4. Use pull-through cache for public registries

**Why →** Centralizing images gives one scan and promotion point.

**Interview tip →** Pull-through cache avoids Docker Hub rate limits.

</details>

### Q138. Design a CI/CD supply chain for images: scan, sign, and enforce at deploy. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Security gates in the pipeline plus admission enforcement.

**What I'd do →**

1. Build with BuildKit, push to ECR with immutable digest
2. Scan with ECR enhanced scanning (Inspector) and fail on Critical
3. Sign with Cosign or AWS Signer
4. Admission policy (Kyverno or Gatekeeper) verifies signatures and allows only the org ECR
5. Deploy by digest not tag

**Why →** Digest pinning plus signature verification stops tag-swap attacks.

**Interview tip →** Mention SBOM generation.

</details>

### Q139. An ECS task cannot pull its image from ECR in a private subnet: "CannotPullContainerError". Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No network path to ECR or missing execution role permission.

**What I'd do →**

1. Add interface endpoints `ecr.api`, `ecr.dkr` and an S3 gateway endpoint for layers
2. Or provide NAT
3. Execution role needs `AmazonECSTaskExecutionRolePolicy`
4. Check SG allows 443 to the endpoints
5. Retry the service

**Why →** Layers are stored in S3, which most people forget when building endpoints.

**Interview tip →** S3 gateway endpoint is part of the set.

</details>

### Q140. Choose between Lambda, Fargate and EKS for a new service. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Based on runtime shape and team capability.

**What I'd do →**

1. Lambda: event-driven, spiky, short (under 15 minutes)
2. Fargate: long-running containers without cluster operations
3. EKS: need Kubernetes ecosystem, portability, complex scheduling
4. Compare cost at expected steady load

**Why →** Operational overhead grows from Lambda to Fargate to EKS.

**Interview tip →** Do not choose EKS for resume value.

</details>

### Q141. Secure an EKS cluster's API endpoint and workloads. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Layered security from endpoint to pod.

**What I'd do →**

1. Make the API endpoint private or restrict public CIDRs: `aws eks update-cluster-config --name prod --resources-vpc-config endpointPublicAccess=false,endpointPrivateAccess=true`
2. Use access entries instead of aws-auth ConfigMap
3. Enforce Pod Security Standards restricted
4. NetworkPolicies and Security Groups for pods
5. Enable control plane logging and GuardDuty EKS protection

**Why →** API endpoint exposure is the highest-impact decision.

**Interview tip →** Access entries are the current auth method.

</details>

### Q142. An application on Lambda needs a secret. Should you put it in environment variables? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Env vars are readable by anyone with `lambda:GetFunctionConfiguration`.

**What I'd do →**

1. Store in Secrets Manager or SSM Parameter Store SecureString
2. Fetch at init and cache, or use the Parameters and Secrets Lambda extension
3. Grant `secretsmanager:GetSecretValue` on that secret
4. Use KMS CMK for extra control

**Why →** Central storage gives rotation and audit.

**Interview tip →** Extension caches calls and cuts cost.

</details>

### Q143. ECS tasks run out of memory only in production. How do you right-size? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Container insights show actual usage.

**What I'd do →**

1. Enable Container Insights
2. Check `MemoryUtilized` versus reserved
3. Set a hard `memory` limit and soft `memoryReservation`
4. Use load test to find p99 memory
5. Alert on OOM exit 137

**Why →** Dev traffic rarely resembles production memory behavior.

**Interview tip →** Hard limit equals kill, soft limit equals scheduling hint.

</details>

### Q144. Run a multi-tenant SaaS on EKS with strong tenant isolation. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Isolation tiers: namespace, node, cluster.

**What I'd do →**

1. Namespaces with ResourceQuotas, NetworkPolicies, RBAC per tenant
2. Dedicated nodes via taints for sensitive tenants
3. Pod Identity roles per tenant
4. Separate clusters or accounts for regulated tenants
5. Observability per tenant labels

**Why →** Choose isolation by tenant risk and cost.

**Interview tip →** Mention noisy neighbor controls.

</details>

### Q145. Run a one-off migration container on ECS before each deploy. What pattern? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Run a standalone task as a pipeline step.

**What I'd do →**

1. `aws ecs run-task --cluster prod --task-definition migrate --launch-type FARGATE --network-configuration ...`
2. Wait: `aws ecs wait tasks-stopped --cluster prod --tasks <arn>`
3. Check exit code via `describe-tasks`
4. Fail the pipeline on non-zero
5. Then deploy the service

**Why →** Separating migration from service start avoids race conditions among replicas.

**Interview tip →** Use backward-compatible migrations.

</details>

### Q146. How does an ECS task get AWS permissions at runtime? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Through its task role, via a credentials endpoint.

**What I'd do →**

1. Define `taskRoleArn` in the task definition
2. ECS injects `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`
3. SDK picks it up automatically
4. Never share the instance role with containers

**Why →** Per-task roles give least privilege.

**Interview tip →** Differentiate task role and execution role.

</details>

### Q147. Lambda deployments should shift traffic gradually. What do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Aliases with weighted routing or CodeDeploy.

**What I'd do →**

1. Publish a version: `aws lambda publish-version --function-name f`
2. Alias `live` weights 90/10
3. Or SAM `DeploymentPreference: Canary10Percent5Minutes` with alarms
4. Roll back by shifting alias

**Why →** Aliases decouple callers from versions.

**Interview tip →** Always call via alias.

</details>

### Q148. Containers on EKS should scale on SQS queue depth. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** KEDA scales pods on external metrics.

**What I'd do →**

1. Install KEDA
2. Create a `ScaledObject` with an `aws-sqs-queue` trigger and `queueLength` 20
3. Authenticate using Pod Identity
4. Set min and max replicas
5. Combine with Karpenter for nodes

**Why →** Queue depth reflects work to do better than CPU.

**Interview tip →** Scale-to-zero is possible.

</details>

### Q149. Plan a migration of 200 Docker Compose services to AWS. Where do you start? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Triage by complexity and make a landing pattern first.

**What I'd do →**

1. Classify stateless, stateful, and batch
2. Build a standard ECS Fargate template (Terraform module)
3. Migrate low-risk services first with weighted traffic
4. Move data stores with DMS
5. Automate observability and CI templates

**Why →** Golden path templates avoid 200 snowflakes.

**Interview tip →** Pilot first, then scale.

</details>

### Q150. An image push to ECR fails with "denied: Your authorization token has expired". What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** ECR tokens last 12 hours.

**What I'd do →**

1. Re-authenticate: `aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 111111111111.dkr.ecr.eu-west-1.amazonaws.com`
2. In CI, run login each job
3. Use credential helper `docker-credential-ecr-login`

**Why →** Tokens are short-lived by design.

**Interview tip →** Credential helper avoids manual login.

</details>

## Observability, Secrets, KMS and CloudFormation

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. You need to be paged when 5xx errors on the ALB exceed 2 percent for 5 minutes. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A CloudWatch alarm using metric math.

**What I'd do →**

1. Metric math: `100 * m1 / m2` with m1 `HTTPCode_Target_5XX_Count` and m2 `RequestCount`
2. Alarm: 5-minute period, threshold 2, 1 of 1 or 2 of 3 datapoints
3. Treat missing data as `notBreaching`
4. Action to an SNS topic wired to PagerDuty or Chatbot

**Why →** Percentages beat raw counts, since traffic varies.

**Interview tip →** Mention treating missing data deliberately.

</details>

### Q152. How do you centralize application logs from many EC2 and ECS services? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** CloudWatch Logs with agents or log drivers, then subscription filters.

**What I'd do →**

1. EC2 uses the CloudWatch agent with `amazon-cloudwatch-agent-ctl -a fetch-config`
2. ECS uses the `awslogs` driver
3. Set retention: `aws logs put-retention-policy --log-group-name /app/api --retention-in-days 30`
4. Subscription filter to Kinesis Firehose to S3 for archive
5. Query with Logs Insights

**Why →** Retention defaults to never expire, which is a silent cost.

**Interview tip →** Always set retention.

</details>

### Q153. CloudWatch costs exploded. Where do you look? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Usually log ingestion and custom metrics.

**What I'd do →**

1. Cost Explorer by usage type `DataProcessing-Bytes` and `MetricMonitorUsage`
2. Find noisy log groups with `aws logs describe-log-groups` sorted by `storedBytes`
3. Reduce debug logging, set retention, use Infrequent Access log class
4. High-cardinality custom metric dimensions are charged per series
5. Use EMF carefully

**Why →** Ingestion costs around 50 cents per GB, so chatty debug logs dominate.

**Interview tip →** Cardinality kills metric budgets.

</details>

### Q154. Investigate a latency spike across microservices on AWS. What tools do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Follow a trace.

**What I'd do →**

1. Enable X-Ray or OpenTelemetry via ADOT
2. Service map to find slow hops
3. Logs Insights with request ID: `fields @timestamp, @message | filter @message like /req-123/`
4. Correlate with ALB `TargetResponseTime`
5. Check downstream throttling

**Why →** Traces show where time went; metrics show that it went.

**Interview tip →** Propagate a correlation ID everywhere.

</details>

### Q155. An auditor asks for proof of every configuration change to security groups for 12 months. What do you provide? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS Config timeline plus CloudTrail.

**What I'd do →**

1. Config records configuration items for `AWS::EC2::SecurityGroup`
2. Retrieve history: `aws configservice get-resource-config-history --resource-type AWS::EC2::SecurityGroup --resource-id sg-0abc`
3. CloudTrail shows who made the call
4. Retain in S3 with lifecycle for 12 months

**Why →** Config shows what changed, CloudTrail shows who.

**Interview tip →** Know the split.

</details>

### Q156. Auto-remediate any security group opened to 0.0.0.0/0 on port 22. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Config rule plus SSM Automation remediation.

**What I'd do →**

1. Enable managed rule `restricted-ssh`
2. Attach remediation `AWS-DisableIncomingSSHOnPort22` or custom runbook
3. Set automatic remediation with 3 retries
4. Notify through EventBridge to Slack
5. Exempt tagged bastion exceptions

**Why →** Closing the hole in minutes beats ticket queues.

**Interview tip →** Include exception handling.

</details>

### Q157. Where should application database passwords live and how do they rotate? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secrets Manager with managed rotation.

**What I'd do →**

1. Create a secret: `aws secretsmanager create-secret --name prod/db --secret-string file://creds.json`
2. Enable rotation: `aws secretsmanager rotate-secret --secret-id prod/db --rotation-rules AutomaticallyAfterDays=30`
3. Use the multi-user rotation strategy for zero downtime
4. App reads at runtime, refreshes on auth failure

**Why →** Rotation without redeploy removes the temptation to never rotate.

**Interview tip →** Aurora and RDS have managed rotation without custom Lambda.

</details>

### Q158. Secrets Manager or SSM Parameter Store: when to use which? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secrets Manager rotates and costs per secret; Parameter Store is cheaper for config.

**What I'd do →**

1. Parameter Store standard parameters are free, SecureString uses KMS
2. Secrets Manager costs about $0.40 per secret per month plus API calls
3. Use Secrets Manager for DB credentials and keys needing rotation or cross-region replication
4. Use Parameter Store for flags and non-rotated config

**Why →** Match cost to need.

**Interview tip →** Cross-account sharing is simpler with Secrets Manager resource policies.

</details>

### Q159. Design KMS key strategy for a multi-account platform. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Keys per data domain with tight key policies.

**What I'd do →**

1. Customer managed keys per account and environment, aliased `alias/app-prod`
2. Key policy delegates to IAM and lists admins separately from users
3. Cross-account grants via key policy plus IAM
4. Annual rotation enabled: `aws kms enable-key-rotation`
5. Multi-Region keys for DR

**Why →** Separate admin and usage permissions so key admins cannot read data.

**Interview tip →** Mention deletion waiting period 7 to 30 days.

</details>

### Q160. An engineer scheduled a KMS key for deletion by mistake. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** There's a waiting period; cancel it.

**What I'd do →**

1. `aws kms cancel-key-deletion --key-id <key>`
2. Then `aws kms enable-key --key-id <key>`
3. Alarm on `ScheduleKeyDeletion` events through EventBridge
4. Restrict `kms:ScheduleKeyDeletion` with SCP
5. Audit what the key encrypts

**Why →** Deleting a key makes all its data unrecoverable.

**Interview tip →** Disable first, delete later.

</details>

### Q161. Write a CloudFormation template that creates an S3 bucket with encryption and block public access. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Declarative resource with properties.

**What I'd do →**

1. Define `AWS::S3::Bucket` with `BucketEncryption` SSE and `PublicAccessBlockConfiguration` all true
2. Add `DeletionPolicy: Retain` for data
3. Validate: `aws cloudformation validate-template --template-body file://t.yaml`
4. Deploy: `aws cloudformation deploy --template-file t.yaml --stack-name app-bucket`

**Why →** Defaults in code remove manual drift.

**Interview tip →** Retain policy protects data on stack deletion.

</details>

### Q162. A CloudFormation stack is stuck in UPDATE_ROLLBACK_FAILED. What now? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A resource cannot return to prior state.

**What I'd do →**

1. Read events: `aws cloudformation describe-stack-events --stack-name s`
2. Fix the blocking cause, e.g. manually deleted resource
3. `aws cloudformation continue-update-rollback --stack-name s --resources-to-skip LogicalId`
4. Then redeploy

**Why →** Skipping gets the stack back to a clean state.

**Interview tip →** Avoid manual console changes.

</details>

### Q163. Detect and handle drift between deployed resources and CloudFormation. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Drift detection compares live to template.

**What I'd do →**

1. `aws cloudformation detect-stack-drift --stack-name s`
2. `describe-stack-resource-drifts` for details
3. Revert in the template or update the stack to correct
4. Schedule regular detection
5. Restrict console write access to reduce drift

**Why →** Drift defeats the repeatability of IaC.

**Interview tip →** Prevention with permissions beats detection.

</details>

### Q164. Roll out a CloudFormation change to 80 accounts safely. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** StackSets with staged rollout.

**What I'd do →**

1. Use service-managed StackSets with org targets
2. Operation preferences: max concurrent 10 percent, failure tolerance 1 account
3. Deploy to canary OU first
4. Region order sequence
5. Monitor `describe-stack-set-operation`

**Why →** Failure tolerance stops a bad change early.

**Interview tip →** Know RegionConcurrencyType.

</details>

### Q165. Stack deletion fails because a bucket is not empty. How do you handle it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CloudFormation will not delete non-empty buckets.

**What I'd do →**

1. Empty: `aws s3 rm s3://b --recursive`, versioned buckets need `s3api delete-objects` for versions
2. Retry delete
3. Or use a custom resource that empties on delete
4. Use `Retain` for important buckets

**Why →** Safety prevents accidental data loss.

**Interview tip →** Versioned buckets also need markers removed.

</details>

### Q166. You need change previews before applying an infrastructure change. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Change sets.

**What I'd do →**

1. `aws cloudformation create-change-set --stack-name s --change-set-name c1 --template-body file://t.yaml`
2. `describe-change-set` to see replacements
3. Look for `Replacement: True` on stateful resources
4. `execute-change-set` after review

**Why →** Replacement of a database is a data-loss event.

**Interview tip →** Use stack policies for protected resources.

</details>

### Q167. Implement CloudTrail for tamper-proof audit logs across an organization. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Org trail into a locked log archive.

**What I'd do →**

1. Create an organization trail with log file validation
2. Deliver to S3 in the log-archive account with bucket policy and Object Lock
3. Encrypt with KMS
4. Send to CloudWatch Logs for alerts
5. Validate: `aws cloudtrail validate-logs --trail-arn ... --start-time ...`

**Why →** Log file validation produces digests that expose tampering.

**Interview tip →** Add CloudTrail Lake for query.

</details>

### Q168. Alert on any root account usage. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Event-based alerting.

**What I'd do →**

1. EventBridge rule on CloudTrail event with `userIdentity.type` equal to `Root`
2. Route to SNS and pager
3. Also detect via Config and Security Hub
4. Enforce with SCP denying root actions in member accounts

**Why →** Root should be almost never used.

**Interview tip →** Centralized root management exists in Organizations.

</details>

### Q169. Build a dashboard and SLO alarm for an API. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Golden signals.

**What I'd do →**

1. Metrics: latency p99, errors, traffic, saturation
2. Dashboards via CloudFormation
3. Composite alarms to cut noise
4. Anomaly detection alarm for traffic
5. Error budget burn rate alarms

**Why →** Composite alarms reduce pager fatigue.

**Interview tip →** Alert on symptoms, not causes.

</details>

### Q170. What does GuardDuty do and how do you enable it org-wide? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Threat detection from logs.

**What I'd do →**

1. Delegate an admin account
2. `aws guardduty create-detector --enable`
3. Auto-enable for new org accounts
4. Findings to Security Hub and EventBridge
5. Enable EKS, S3, Malware Protection plans

**Why →** No agents required.

**Interview tip →** Needs response runbooks, not just enabling.

</details>

### Q171. Respond to a GuardDuty finding: `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration`. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Instance role credentials used from outside AWS.

**What I'd do →**

1. Revoke sessions: attach `AWSRevokeOlderSessions` style deny with `aws:TokenIssueTime` condition
2. Isolate instance with a quarantine SG
3. Snapshot disk and memory for forensics
4. Review CloudTrail for actions
5. Enforce IMDSv2 and rotate secrets

**Why →** Revoking sessions invalidates stolen tokens.

**Interview tip →** Isolate before terminate.

</details>

### Q172. Make secrets available to ECS tasks without putting them in the task definition in plaintext. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reference secrets by ARN.

**What I'd do →**

1. In container definition use `secrets: [{name: DB_PASS, valueFrom: <secret-arn>}]`
2. Execution role needs `secretsmanager:GetSecretValue` and `kms:Decrypt`
3. Value is injected at start
4. Restart tasks after rotation

**Why →** Plain env vars appear in console.

**Interview tip →** Injection happens only at task start.

</details>

### Q173. How do you enforce that every CloudFormation or Terraform change is reviewed for security before deploy? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Policy as code in CI.

**What I'd do →**

1. Run `cfn-guard validate` or `checkov` on templates in pull requests
2. Fail on high severity
3. Require approval in the pipeline
4. Use SCP as the backstop
5. Report in Security Hub

**Why →** Shifting left is cheaper than detecting later.

**Interview tip →** Policies in version control are auditable.

</details>

## Cost Optimization and Disaster Recovery

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q174. Your AWS bill jumped 40 percent this month. How do you find the cause? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Start with Cost Explorer grouped by service, then account, then usage type.

**What I'd do →**

1. Cost Explorer daily view grouped by Service to find the day it jumped
2. Drill by Usage Type and Linked Account
3. Filter by tag to find the team
4. Check Cost Anomaly Detection alerts
5. Cross-check CloudTrail for the launch event

**Why →** Daily granularity points to a specific change.

**Interview tip →** Set budgets with alerts so you hear first.

</details>

### Q175. Set up budgets so no one is surprised by spend. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS Budgets with alerts and optional actions.

**What I'd do →**

1. `aws budgets create-budget` for monthly cost with 80 percent and 100 percent thresholds
2. Notify SNS and email
3. Forecast alerts
4. Budget actions can attach a deny SCP or stop instances in dev
5. Cost Anomaly Detection monitors per service

**Why →** Forecast alerts warn before the money is spent.

**Interview tip →** Use per-account budgets.

</details>

### Q176. Dev environments run 24/7 but are used 9 to 6 on weekdays. How do you cut cost? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Schedule them off.

**What I'd do →**

1. Instance Scheduler on AWS or EventBridge Scheduler calling SSM `AWS-StopEC2Instance`
2. Scale ASGs to zero at night
3. Stop RDS (auto-restarts after 7 days) or use Aurora Serverless v2 min capacity
4. Tag `schedule=office-hours`
5. Report savings

**Why →** Running 50 hours instead of 168 saves about 70 percent.

**Interview tip →** Mention RDS 7-day restart.

</details>

### Q177. Find and remove idle and orphaned resources. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unattached volumes, idle load balancers, old snapshots, unused EIPs.

**What I'd do →**

1. `aws ec2 describe-volumes --filters Name=status,Values=available`
2. Unused ELBs with zero requests via CloudWatch
3. Trusted Advisor and Compute Optimizer idle findings
4. Snapshots without AMIs
5. Review NAT gateways with no traffic

**Why →** Orphans pile up after teardown with no owner.

**Interview tip →** Tag owner on everything.

</details>

### Q178. Reduce data transfer cost in a media-heavy architecture. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data egress usually tops the bill.

**What I'd do →**

1. Serve through CloudFront, cheaper per GB than S3 direct egress and with a free tier
2. Use CloudFront committed-use savings bundle
3. Compress
4. Keep inter-AZ traffic low
5. Use endpoints to avoid NAT per-GB charges
6. Consider S3 Intelligent-Tiering only for storage, not transfer

**Why →** Egress tiers favour CDNs at volume.

**Interview tip →** Always ask where the bytes flow.

</details>

### Q179. Choose S3 storage classes for data accessed unpredictably. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Intelligent-Tiering.

**What I'd do →**

1. Move objects with lifecycle to `INTELLIGENT_TIERING`
2. Frequent, Infrequent, Archive Instant tiers automatically
3. Optional archive tiers
4. No retrieval charges
5. Small objects under 128 KB are not tiered

**Why →** It removes guesswork for unknown patterns.

**Interview tip →** Monitoring fee per object applies.

</details>

### Q180. Define RPO and RTO and map them to AWS DR strategies. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** RPO is acceptable data loss; RTO is acceptable downtime.

**What I'd do →**

1. Backup and restore: hours RTO, cheapest
2. Pilot light: core data replicated, compute off
3. Warm standby: scaled-down full stack running
4. Multi-site active-active: near-zero RTO and RPO
5. Choose per workload criticality

**Why →** Cost rises as targets shrink.

**Interview tip →** State both numbers when asked.

</details>

### Q181. Design warm standby DR for a web app and database in another region. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Aurora Global Database plus scaled-down stack.

**What I'd do →**

1. Aurora Global Database replica in the DR region, lag usually under a second
2. Minimal ASG or ECS service running
3. S3 CRR and replicated secrets and ECR
4. Route 53 failover with ARC
5. Promote: `aws rds failover-global-cluster --global-cluster-identifier g --target-db-cluster-identifier dr-cluster`
6. Scale up and cut DNS

**Why →** Warm standby shortens recovery to minutes.

**Interview tip →** Test it twice a year.

</details>

### Q182. Your DR region lacks capacity when you fail over. How do you prevent that? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Capacity is not guaranteed in a regional event.

**What I'd do →**

1. Pre-provision critical capacity or buy On-Demand Capacity Reservations
2. Keep warm standby at meaningful size
3. Avoid relying on control plane at failure time
4. Increase service quotas in the DR region
5. Run failover drills

**Why →** Many teams discover quota limits mid-incident.

**Interview tip →** Static stability.

</details>

### Q183. Recover an accidentally deleted EBS volume or EC2 instance. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Needs prior snapshots or AMI.

**What I'd do →**

1. Use Data Lifecycle Manager or AWS Backup schedule
2. Restore a volume: `aws ec2 create-volume --snapshot-id snap-0abc --availability-zone eu-west-1a`
3. Recycle Bin for snapshots and AMIs retention rules
4. Attach and mount
5. Verify

**Why →** Recycle Bin protects against deletion mistakes.

**Interview tip →** Enable it for AMIs and snapshots.

</details>

### Q184. A region-wide outage hits us-east-1. Which dependencies could still break a multi-region app? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Some global control planes live in us-east-1.

**What I'd do →**

1. IAM and Route 53 control planes are in us-east-1; data planes remain available
2. Pre-create resources, avoid creating new IAM or DNS changes during failover
3. ACM certificates for CloudFront
4. Cross-region replicated secrets and images
5. Use ARC data plane routing controls

**Why →** Static stability means failover requires no control plane calls.

**Interview tip →** Mention the Dec 2021 and Oct 2025 style events.

</details>

### Q185. Run a game day to validate your DR plan. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Practice with real failures.

**What I'd do →**

1. Define steady state and hypotheses
2. Use AWS Fault Injection Service to stop instances or block AZ network
3. Execute failover runbook, time RTO
4. Verify data integrity
5. Record gaps and fix

**Why →** Only drills reveal hidden dependencies.

**Interview tip →** Run them in prod-like environments with guardrails.

</details>

### Q186. Use Savings Plans and Spot together without overcommitting. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Layer purchases by predictability.

**What I'd do →**

1. Cover baseline with Savings Plans
2. Cover variable with On-Demand
3. Cover fault-tolerant with Spot
4. Track coverage and utilization
5. Review monthly

**Why →** Utilization below 95 percent signals overcommit.

**Interview tip →** Know coverage versus utilization.

</details>

### Q187. Build a FinOps process across 100 accounts. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Visibility, accountability, optimization.

**What I'd do →**

1. CUR delivered to S3 and queried with Athena or QuickSight
2. Mandatory tags through tag policies
3. Showback dashboards per team
4. Monthly review of Compute Optimizer and idle findings
5. Central commitment management in the payer account

**Why →** Teams change behavior when they see their own costs.

**Interview tip →** Mention unit costs like cost per request.

</details>

### Q188. What is the AWS Free Tier trap for new accounts? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Free tier has limits and expiry.

**What I'd do →**

1. 12-month offers expire
2. Set a zero-spend budget
3. Delete resources when done
4. Check Billing dashboard Free Tier usage
5. Avoid leaving NAT gateways or Elastic IPs

**Why →** NAT gateways and public IPv4 incur charges immediately.

**Interview tip →** Beginners should set budgets first.

</details>

### Q189. Reduce RDS cost without hurting performance. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Right-size and commitment.

**What I'd do →**

1. Check CPU and memory via Performance Insights
2. Move to Graviton instances (db.r7g)
3. Use gp3 storage
4. Buy RDS Reserved Instances
5. Drop unused read replicas and old snapshots

**Why →** Graviton gives better price performance.

**Interview tip →** Reserved Instances still matter for RDS.

</details>

### Q190. Choose between Multi-AZ and Multi-Region for a database. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** They solve different failures.

**What I'd do →**

1. Multi-AZ protects against instance and AZ failure with automatic failover
2. Multi-Region protects against regional disasters with manual or orchestrated failover
3. Add cross-region read replica or global database
4. Combine both for critical systems
5. Cost doubles roughly per region

**Why →** Match scope to the threat.

**Interview tip →** Multi-AZ is not a backup.

</details>

### Q191. Design cross-account disaster recovery for ransomware where the attacker has prod admin. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Assume prod account compromise.

**What I'd do →**

1. Isolated recovery account with immutable vaults
2. Backup Vault Lock compliance mode
3. Logically air-gapped vault
4. Separate identity path and break-glass
5. Rehearse clean-room restore

**Why →** Attackers cannot delete what they cannot reach.

**Interview tip →** Compliance mode cannot be shortened even by root.

</details>

### Q192. Estimate the cost of a new architecture before building it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS Pricing Calculator plus assumptions.

**What I'd do →**

1. List components and quantities
2. Use the calculator and export
3. Add data transfer and logging
4. Add 20 percent margin
5. Compare against measured pilot

**Why →** Hidden costs are NAT, logs, and cross-AZ.

**Interview tip →** Validate with a pilot.

</details>

### Q193. Switch workloads to Graviton. What is the migration path? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Check architecture compatibility.

**What I'd do →**

1. Rebuild images multi-arch with `docker buildx build --platform linux/amd64,linux/arm64`
2. Test in staging on `m7g`
3. Update AMIs
4. Canary then roll
5. Measure price-performance

**Why →** ARM gives lower cost per unit work.

**Interview tip →** Native dependencies are the trap.

</details>

### Q194. Meet a 4-hour RTO with low cost for a mid-tier internal app. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pilot light or backup restore.

**What I'd do →**

1. IaC for full stack in the DR region
2. Cross-region snapshots through AWS Backup
3. Runbook to deploy and restore
4. Time a restore test
5. Document DNS cutover

**Why →** IaC makes rebuild minutes not days.

**Interview tip →** Prove it with a timed test.

</details>

### Q195. What backup would you set for an EC2 fleet and how do you verify it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Centralized backup with tests.

**What I'd do →**

1. AWS Backup plan with daily rule, 35-day retention, tag selection
2. Copy to another region
3. `aws backup start-restore-job` in a test
4. Verify application boots
5. Alarm on failed jobs

**Why →** Restore tests find silent failures.

**Interview tip →** Backups need alarms.

</details>

### Q196. Optimize S3 costs for 2 PB of logs mostly never read. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Archive tiers and compression.

**What I'd do →**

1. Lifecycle to Glacier Instant, Flexible, Deep Archive
2. Compress and convert to Parquet
3. Reduce small objects
4. Set expiration
5. Use S3 Storage Lens

**Why →** Deep Archive costs about $1 per TB per month.

**Interview tip →** Retrieval times and fees matter.

</details>

### Q197. Plan a landing zone from scratch for a startup growing to 50 engineers. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Start simple with a scalable skeleton.

**What I'd do →**

1. Create Organizations with Security, Infra, Workloads OUs
2. Control Tower with guardrails
3. Identity Center with an IdP
4. Network account with Transit Gateway later
5. Baseline CloudTrail, Config, GuardDuty
6. Account vending through IaC

**Why →** Retrofitting accounts later costs far more.

**Interview tip →** Start with accounts per environment.

</details>

### Q198. The CFO wants to know unit cost per customer. How do you measure? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tag by tenant or allocate shared costs.

**What I'd do →**

1. Tag resources `tenant` where possible
2. Split Cost Allocation Data for EKS and ECS
3. Use CUR and Athena
4. Allocate shared by usage metrics
5. Report cost per active customer

**Why →** Unit economics guides pricing.

**Interview tip →** Split cost allocation covers containers.

</details>

### Q199. Leadership wants to leave one cloud region for a second one for compliance with data residency. How do you plan the multi-region data layer? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data residency means some data must never leave its region, so replication design is a legal decision before a technical one.

**What I'd do →**

1. Classify data by residency rule and tag it
2. Keep regulated stores single-region with in-region Multi-AZ and in-region backups only
3. Replicate only non-regulated or anonymized data across regions
4. Enforce with SCP region deny, S3 replication rules scoped by tag and KMS keys that are single-region
5. Document and test with a Config rule that flags cross-region replication on tagged resources

**Why →** A single misconfigured replication rule can violate residency, so enforcement must be automated, not procedural.

**Interview tip →** Use single-Region KMS keys for regulated data so ciphertext cannot be decrypted elsewhere.

</details>

### Q200. What does the AWS shared responsibility model mean in practice for an EC2 web server? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS secures the cloud itself; you secure everything you put in it.

**What I'd do →**

1. AWS: hardware, hypervisor, facilities, managed-service internals
2. You: guest OS patching, security groups, IAM, application code, data encryption choices
3. For RDS the line moves up: AWS patches the engine, you own parameters, access and data
4. Document the split per service in your security review

**Why →** The line moves up the stack as you adopt more managed services.

**Interview tip →** Give one EC2 and one RDS example to show you understand the shift.

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
