# AWS — Core Concepts

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

> Core concepts, internals and comparisons for AWS in DevOps interviews · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to AWS](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Global Infrastructure and IAM Basics](#global-infrastructure-and-iam-basics) — Q1–Q25
- [IAM Advanced, STS and Multi-Account](#iam-advanced-sts-and-multi-account) — Q26–Q50
- [VPC and Networking](#vpc-and-networking) — Q51–Q75
- [EC2, Auto Scaling and Load Balancing](#ec2-auto-scaling-and-load-balancing) — Q76–Q100
- [Storage and Databases](#storage-and-databases) — Q101–Q125
- [Containers and Serverless](#containers-and-serverless) — Q126–Q150
- [DNS, CDN and Messaging](#dns-cdn-and-messaging) — Q151–Q175
- [Observability, Security Services and IaC](#observability-security-services-and-iac) — Q176–Q200

---

## Global Infrastructure and IAM Basics

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What is the difference between a Region, an Availability Zone and an edge location? 🟢

<details>
<summary><b>Answer</b></summary>

A **Region** is a separate geographic area (for example `ap-south-1`, Mumbai) made of multiple isolated **Availability Zones**. An AZ is one or more discrete data centers with independent power, cooling and networking, connected to its sibling AZs by low-latency links. **Edge locations** are CloudFront/Route 53/Global Accelerator points of presence that sit outside Regions and cache or terminate traffic close to users.

Regions are fully independent: resources, IAM-unrelated quotas and most service endpoints are per Region. AZs are the unit you spread across for high availability.

</details>

### Q2. How do you choose a Region for a workload? 🟢

<details>
<summary><b>Answer</b></summary>

Four factors, in this order: **compliance and data residency** (some data legally cannot leave a country), **latency** to users, **service and feature availability** (new services land in us-east-1, us-west-2 and eu-west-1 first), and **price** (the same instance can differ 20–30% between Regions).

In production we also weigh DR pairing (is there a sensible second Region nearby that has the same services) and whether the team already has landing-zone guardrails there.

</details>

### Q3. What is the difference between an AZ name like `us-east-1a` and an AZ ID like `use1-az1`? 🟡

<details>
<summary><b>Answer</b></summary>

AZ names are mapped to physical zones **per account**, randomly, to spread load. Your `us-east-1a` may be a different physical zone than another account's `us-east-1a`. The AZ ID (`use1-az1`) is the same physical zone for everyone.

This matters for cross-account setups: PrivateLink endpoint services, shared subnets via RAM and latency-sensitive pairs must match on **AZ ID**, not name.

```bash
aws ec2 describe-availability-zones --region us-east-1 \
  --query 'AvailabilityZones[].[ZoneName,ZoneId]' --output table
```

</details>

### Q4. Explain the shared responsibility model. 🟢

<details>
<summary><b>Answer</b></summary>

AWS is responsible for security **of** the cloud: hardware, facilities, hypervisor (Nitro), and the managed-service software layer. You are responsible for security **in** the cloud: IAM, data classification and encryption choices, network rules, OS patching on EC2, application code.

The split moves with the service. On EC2 you patch the guest OS. On RDS AWS patches the engine host but you choose parameters, network exposure and credentials. On Lambda AWS owns the runtime patching, you own the code and its permissions.

</details>

### Q5. What is the root user and how should you treat it? 🟢

<details>
<summary><b>Answer</b></summary>

The root user is the identity created with the account email. It has unrestricted access that cannot be limited by IAM policies, so it must be locked down: strong unique password, **MFA** (hardware key or passkey), no access keys, and never used for daily work.

Only a handful of tasks need root: changing the account email or support plan, closing the account, restoring IAM permissions when locked out, and a few S3/SQS bucket-policy recoveries. In an Organization you can centralize and even remove member-account root credentials.

</details>

### Q6. What is the difference between an IAM user, group and role? 🟢

<details>
<summary><b>Answer</b></summary>

A **user** is a long-lived identity with a password and/or access keys. A **group** is just a collection of users to attach policies to; groups cannot be principals and cannot be nested. A **role** has no credentials of its own: it is assumed by a principal (a user, AWS service, another account, a federated identity) and yields **temporary credentials** through STS.

Modern practice: humans sign in through IAM Identity Center, workloads use roles, and IAM users are the exception.

</details>

### Q7. What are the main IAM policy types? 🟡

<details>
<summary><b>Answer</b></summary>

- **Identity-based** policies attach to users, groups and roles.
- **Resource-based** policies attach to a resource (S3 bucket, SQS queue, KMS key, role trust policy) and name a `Principal`.
- **Permission boundaries** cap what an identity-based policy can grant.
- **SCPs** (and RCPs) in Organizations cap what accounts can do.
- **Session policies** passed during `AssumeRole` or federation further restrict that session.
- **ACLs** are legacy resource-level grants (S3, VPC NACLs are unrelated).

Only identity-based and resource-based policies **grant** permissions. The rest only restrict.

</details>

### Q8. Walk through the anatomy of an IAM policy document. 🟢

<details>
<summary><b>Answer</b></summary>

A policy has a `Version` (always `"2012-10-17"`) and one or more `Statement`s. Each statement has `Effect` (Allow/Deny), `Action` (or `NotAction`), `Resource` (or `NotResource`), an optional `Condition`, and `Principal` in resource-based policies.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "ReadLogs",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": ["arn:aws:s3:::acme-logs", "arn:aws:s3:::acme-logs/*"],
    "Condition": {"StringEquals": {"aws:RequestedRegion": "ap-south-1"}}
  }]
}
```

Note the bucket ARN and object ARN are separate resources.

</details>

### Q9. What is the difference between AWS managed, customer managed and inline policies? 🟡

<details>
<summary><b>Answer</b></summary>

**AWS managed** policies are created and updated by AWS (`ReadOnlyAccess`); convenient but broad and they can change under you. **Customer managed** policies are standalone documents you own, versioned (up to 5 versions) and reusable across identities. **Inline** policies are embedded in a single identity and die with it.

We default to customer managed policies in code (Terraform/CloudFormation) so permissions are reviewable and reusable, and use inline only for strictly one-to-one cases.

</details>

### Q10. What is the AWS credential provider chain? 🟡

<details>
<summary><b>Answer</b></summary>

SDKs and the CLI look for credentials in a fixed order: explicit parameters, environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SESSION_TOKEN`), shared config/credentials files and profiles (including SSO and `role_arn`), web identity token (`AWS_WEB_IDENTITY_TOKEN_FILE`, used by EKS and GitHub OIDC), container credentials (ECS task role), and finally the EC2 instance metadata service.

This is why a stray `AWS_ACCESS_KEY_ID` in a shell silently overrides the instance role. When debugging "wrong account" issues, run `aws sts get-caller-identity` first.

</details>

### Q11. What is the difference between a trust policy and a permissions policy on a role? 🟡

<details>
<summary><b>Answer</b></summary>

The **trust policy** (assume-role policy document) is a resource-based policy on the role that says *who may assume it*. The **permissions policies** say *what the role can do once assumed*. Both must be right: a perfect permissions policy is useless if the trust policy does not list your principal, and the caller also needs `sts:AssumeRole` permission if it is in another account.

```json
{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}
```

</details>

### Q12. What is an instance profile? 🟢

<details>
<summary><b>Answer</b></summary>

An instance profile is the container that passes an IAM role to an EC2 instance. The console hides it by creating a same-named profile, but the CLI and CloudFormation expose it (`AWS::IAM::InstanceProfile`). The instance reads rotating credentials from the metadata service at `/latest/meta-data/iam/security-credentials/<role>`.

```bash
aws ec2 associate-iam-instance-profile --instance-id i-0abc \
  --iam-instance-profile Name=app-role
```

</details>

### Q13. Which AWS services are global and which are regional? 🟢

<details>
<summary><b>Answer</b></summary>

Truly global control planes: IAM, Organizations, Route 53, CloudFront, WAF (for CloudFront scope), Global Accelerator, and Shield Advanced. Almost everything else is regional: EC2, S3 (buckets are regional even though names are global), RDS, Lambda, VPC.

Practical gotchas: IAM and CloudFront ACM certificates live in **us-east-1** control-plane context, and CloudFormation, CloudWatch and CloudTrail are regional so you need an aggregation strategy for multi-Region.

</details>

### Q14. What is an ARN and how is it structured? 🟢

<details>
<summary><b>Answer</b></summary>

`arn:partition:service:region:account-id:resource`. Partition is `aws`, `aws-cn` or `aws-us-gov`. Some fields are empty for global or account-less resources: S3 buckets have no region or account (`arn:aws:s3:::my-bucket`), IAM has no region (`arn:aws:iam::123456789012:role/app`).

Wildcards work in policies, so `arn:aws:s3:::my-bucket/*` matches all objects. Hard-coding the partition breaks GovCloud portability; use `${AWS::Partition}` in CloudFormation.

</details>

### Q15. What does `aws sts get-caller-identity` return and why is it the first command you run? 🟢

<details>
<summary><b>Answer</b></summary>

It returns the `UserId`, `Account` and `Arn` of the credentials the CLI is currently using. It needs no IAM permission, so it never fails for authorization reasons, which makes it the universal "who am I and in which account" check.

```bash
$ aws sts get-caller-identity
{"UserId":"AROAEXAMPLE:alice","Account":"123456789012",
 "Arn":"arn:aws:sts::123456789012:assumed-role/Admin/alice"}
```

An `assumed-role` ARN means temporary credentials; a bare `user/` ARN means long-lived keys.

</details>

### Q16. How does MFA work in IAM and how do you enforce it? 🟡

<details>
<summary><b>Answer</b></summary>

IAM supports virtual TOTP apps, FIDO2 security keys/passkeys and hardware TOTP tokens. You enforce it with a policy that denies everything except MFA self-management when `aws:MultiFactorAuthPresent` is false.

```json
{"Effect":"Deny","NotAction":["iam:*MFADevice*","iam:ListVirtualMFADevices","sts:GetSessionToken"],
 "Resource":"*",
 "Condition":{"BoolIfExists":{"aws:MultiFactorAuthPresent":"false"}}}
```

`BoolIfExists` matters: requests from roles or keys without MFA context would otherwise slip past a plain `Bool`. With Identity Center, enforce MFA in the identity source instead.

</details>

### Q17. What is IAM Access Analyzer? 🟡

<details>
<summary><b>Answer</b></summary>

It is a set of features: **external access findings** (resources like S3 buckets, roles, KMS keys, SQS queues shared outside your zone of trust, using automated reasoning), **unused access findings** (unused roles, keys, permissions), **policy validation** (lint and check against `cfn`/Terraform plans), and **policy generation** from CloudTrail activity.

The zone of trust is an account or the whole Organization. We run an org-level analyzer from the delegated admin account and route findings to Security Hub.

</details>

### Q18. How do you find unused IAM users, keys and permissions? 🟡

<details>
<summary><b>Answer</b></summary>

Use the **credential report** for account-wide user/key age and last-used data, **Access Advisor** (`iam generate-service-last-accessed-details`) for service-level last access per identity, and Access Analyzer unused-access findings at scale.

```bash
aws iam generate-credential-report
aws iam get-credential-report --query Content --output text | base64 -d > cred.csv
```

Look at `access_key_1_last_used_date`, `password_last_used` and `mfa_active` columns.

</details>

### Q19. IAM Identity Center vs IAM users: when do you use which? 🟡

<details>
<summary><b>Answer</b></summary>

Identity Center (formerly AWS SSO) gives workforce users single sign-on across all Organization accounts using permission sets, backed by its own directory or an external IdP (Okta, Entra ID) via SAML/SCIM. Under the hood it creates `AWSReservedSSO_*` roles in each account and hands out short-lived credentials.

Use it for all humans. Keep IAM users only for rare cases such as a third-party tool that cannot assume roles, or emergency break-glass accounts, and treat each as an exception to justify.

</details>

### Q20. What is a service-linked role? 🟡

<details>
<summary><b>Answer</b></summary>

A role predefined by an AWS service, with a trust policy and permissions fixed by that service, so it can act on your behalf (for example `AWSServiceRoleForAutoScaling`, `AWSServiceRoleForElasticLoadBalancing`). You cannot edit its permissions, and you can delete it only when the service no longer uses it.

They are created automatically on first use if the caller has `iam:CreateServiceLinkedRole`. SCPs do not apply to them.

</details>

### Q21. Name useful IAM condition keys and what they do. 🟡

<details>
<summary><b>Answer</b></summary>

- `aws:SourceIp` / `aws:SourceVpc` / `aws:SourceVpce`: restrict by network origin.
- `aws:RequestedRegion`: limit Regions an action may target.
- `aws:PrincipalOrgID`: allow any principal in your Organization.
- `aws:PrincipalTag/*` and `aws:ResourceTag/*`: attribute-based access control.
- `aws:SecureTransport`: deny non-TLS requests.
- `aws:MultiFactorAuthPresent`, `aws:CurrentTime`, `aws:ViaAWSService`.

Note `aws:SourceIp` is not evaluated sensibly when the call comes via an AWS service on your behalf; combine it with `aws:ViaAWSService`.

</details>

### Q22. What is the difference between edge locations, Regional Edge Caches and Global Accelerator? 🟡

<details>
<summary><b>Answer</b></summary>

CloudFront serves content from **edge locations**; on a miss it consults a **Regional Edge Cache**, a larger cache tier, before the origin. **Global Accelerator** is not a cache: it gives you two static anycast IPs that enter the AWS backbone at the nearest edge and then route over the private network to your regional endpoint (ALB, NLB, EC2, Elastic IP).

Use CloudFront for cacheable HTTP, Global Accelerator for non-HTTP or latency-sensitive TCP/UDP and for fast regional failover without DNS TTL waits.

</details>

### Q23. What are Local Zones, Wavelength and Outposts? 🟡

<details>
<summary><b>Answer</b></summary>

**Local Zones** extend a Region into a metro area for single-digit-millisecond latency to users (for example media, gaming). **Wavelength Zones** embed AWS compute inside telecom 5G networks. **Outposts** are AWS-owned racks or servers installed in your own data center, managed by AWS, for low latency to on-prem systems or data residency.

All three attach to a parent Region's VPC and control plane, so you create a subnet in the zone and launch instances as usual.

</details>

### Q24. Does data leave a Region unless you move it? 🟢

<details>
<summary><b>Answer</b></summary>

No. AWS does not move customer content out of the Region you selected unless you configure replication, copying, global tables or similar, or call a global service like CloudFront that caches content at edges. Metadata such as IAM, billing and Route 53 configuration is global.

For regulated workloads we back this with an SCP that denies actions outside approved Regions via `aws:RequestedRegion`.

</details>

### Q25. What is the AWS Nitro System and why does it matter to you as a DevOps engineer? 🔴

<details>
<summary><b>Answer</b></summary>

Nitro offloads virtualization functions (networking, EBS storage, monitoring, security) to dedicated hardware cards and uses a minimal hypervisor, so nearly all host CPU/RAM goes to your instance and there is no operator access to instance memory.

Practical consequences: EBS appears as NVMe devices (`/dev/nvme1n1`, so device names in `fstab` should use UUIDs), ENA is the networking driver, instance types 5th generation and later get consistent performance, and bare-metal instance types exist (`m7i.metal-24xl`) for running your own hypervisor or licensed software.

</details>

## IAM Advanced, STS and Multi-Account

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. Explain the IAM policy evaluation logic. 🔴

<details>
<summary><b>Answer</b></summary>

Every request starts at **implicit deny**. AWS evaluates all applicable policies and the order of reasoning is:

1. Any **explicit Deny** in any policy wins immediately.
2. **Organization SCPs/RCPs** must allow the action (otherwise deny).
3. **Resource-based policy** may grant access (same account: enough on its own for IAM roles/users in many cases).
4. **Identity-based policy** must allow.
5. **Permission boundary** must allow, if set.
6. **Session policy** must allow, if passed.

The effective permission is the intersection of the restricting layers with the union of the granting ones. Remember: an Allow in an SCP never grants anything, it only permits.

</details>

### Q27. How does cross-account access evaluation differ from same-account? 🔴

<details>
<summary><b>Answer</b></summary>

Within one account, an Allow in either the identity policy or the resource policy is enough (the union). Across accounts, **both sides must allow**: the resource owner's resource policy (or a trust policy) must name the external principal, and the calling account's identity policy must allow the action.

Typical example: to read an S3 bucket in account B from a role in account A, B's bucket policy allows A's role (or the account root) and A's role policy allows `s3:GetObject` on that bucket. If the object was uploaded by another account without bucket-owner-enforced ownership, ACL ownership also bites, which is why we enable **BucketOwnerEnforced**.

</details>

### Q28. How does `sts:AssumeRole` work, step by step? 🟡

<details>
<summary><b>Answer</b></summary>

1. Caller invokes `AssumeRole` with the target role ARN and a session name.
2. STS checks the role's **trust policy** allows the caller and, cross-account, that the caller's own policy allows `sts:AssumeRole`.
3. STS returns `AccessKeyId`, `SecretAccessKey` and `SessionToken`, valid 15 minutes to the role's max session duration (default 1 hour, up to 12).
4. The caller signs requests with these; CloudTrail logs them with the assumed-role ARN and the session name for attribution.

```bash
aws sts assume-role --role-arn arn:aws:iam::222222222222:role/deploy \
  --role-session-name ci-$BUILD_ID --duration-seconds 3600
```

</details>

### Q29. What is the difference between AssumeRole, AssumeRoleWithWebIdentity and AssumeRoleWithSAML? 🟡

<details>
<summary><b>Answer</b></summary>

`AssumeRole` is called by an already-authenticated AWS principal. `AssumeRoleWithWebIdentity` exchanges an OIDC token (Cognito, Google, GitHub Actions, EKS service-account token) for credentials, with no AWS credentials needed to make the call. `AssumeRoleWithSAML` exchanges a SAML assertion from an enterprise IdP.

The latter two are unsigned calls, so the trust policy must carefully constrain claims (`sub`, `aud`) to prevent any token from the IdP assuming the role.

</details>

### Q30. What is a permission boundary and when would you use one? 🟡

<details>
<summary><b>Answer</b></summary>

A permission boundary is a managed policy set on a user or role that defines the **maximum** permissions it can ever have. Effective access is the intersection of the identity policy and the boundary. It never grants anything itself.

Use it for **delegated administration**: let developers create roles for their Lambdas, but require `iam:PermissionsBoundary` to equal your boundary ARN on every `CreateRole`, so they cannot create a role more powerful than themselves.

```bash
aws iam put-role-permissions-boundary --role-name dev-app \
  --permissions-boundary arn:aws:iam::123456789012:policy/DevBoundary
```

</details>

### Q31. What do Service Control Policies do and not do? 🟡

<details>
<summary><b>Answer</b></summary>

SCPs set the maximum permissions for **all IAM users and roles (including root) in member accounts** of an OU or account. They do not grant permissions, do not affect the **management account**, do not affect service-linked roles, and do not apply to resource-based access from principals outside the Organization.

An identity needs an Allow in its own policy **and** an Allow through every SCP level from root to account. The default `FullAWSAccess` SCP is attached everywhere; remove it only when using the allow-list approach.

</details>

### Q32. Deny-list vs allow-list SCP strategy: which do you pick? 🔴

<details>
<summary><b>Answer</b></summary>

**Deny-list** keeps `FullAWSAccess` and adds targeted denies (leave Org, disable CloudTrail, unapproved Regions, delete flow logs). It is low-maintenance and new services work automatically. **Allow-list** removes `FullAWSAccess` and explicitly allows services; it is tight but every new service or API needs an SCP change and you hit the 5,120-character size limit quickly.

We use deny-lists for almost everything and reserve allow-lists for regulated OUs. Always test SCPs in a sandbox OU, since a bad deny can lock out even your pipeline roles.

</details>

### Q33. What is a session policy? 🔴

<details>
<summary><b>Answer</b></summary>

An inline policy (or up to 10 managed policy ARNs) passed as the `Policy`/`PolicyArns` parameter when calling `AssumeRole`, `AssumeRoleWithWebIdentity`, `GetFederationToken` and similar. It limits the resulting session to the intersection of the role's permissions and the session policy. It cannot expand permissions.

Use it to give a short-lived session to a tenant or job with scope narrower than the base role, for example restricting an S3 prefix per customer in a multi-tenant app.

</details>

### Q34. What is the confused deputy problem and how does ExternalId solve it? 🔴

<details>
<summary><b>Answer</b></summary>

A third-party SaaS that assumes a role in your account could be tricked by another customer into using *their* role ARN for your resources. The vendor generates a unique **ExternalId** per customer, you place it in the trust policy condition, and the vendor must pass it on `AssumeRole`.

```json
"Condition": {"StringEquals": {"sts:ExternalId": "7b1f-customer-unique-id"}}
```

For AWS-service-to-service calls the equivalent protection is `aws:SourceArn` and `aws:SourceAccount` conditions (for example on SNS topic or S3 bucket policies for CloudTrail/Config).

</details>

### Q35. What are the limits of role chaining? 🔴

<details>
<summary><b>Answer</b></summary>

Role chaining is when a role session assumes another role. The resulting session is **capped at 1 hour** regardless of the target role's max session duration, and you cannot refresh it with the original duration. Session tags can be passed along transitively if marked transitive.

Pipelines that hop account to account (CI role to hub role to spoke role) hit this; keep chains short and each stage under an hour.

</details>

### Q36. How does GitHub Actions authenticate to AWS without access keys? 🟡

<details>
<summary><b>Answer</b></summary>

You create an IAM OIDC identity provider for `token.actions.githubusercontent.com`, a role whose trust policy allows `sts:AssumeRoleWithWebIdentity` for that provider, and conditions that pin `aud` and `sub` to your repo and branch/environment. The workflow requests an OIDC token and the `aws-actions/configure-aws-credentials` action exchanges it.

```json
"Condition": {
  "StringEquals": {"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"},
  "StringLike": {"token.actions.githubusercontent.com:sub": "repo:acme/app:ref:refs/heads/main"}
}
```

The workflow also needs `permissions: id-token: write`.

</details>

### Q37. What are AWS Organizations, OUs and the management account? 🟢

<details>
<summary><b>Answer</b></summary>

Organizations groups accounts under a single **management account** in a hierarchy of **organizational units** (OUs). It provides consolidated billing, SCP/RCP/tag/backup policies attached at root, OU or account level, and integration with services like CloudTrail, Config, GuardDuty and Identity Center for org-wide enablement.

The management account should hold no workloads; it is the most privileged account in the company.

</details>

### Q38. What is AWS Control Tower? 🟡

<details>
<summary><b>Answer</b></summary>

Control Tower is an opinionated landing-zone service built on Organizations, Identity Center, CloudFormation StackSets, Config and CloudTrail. It sets up a **Log Archive** and **Audit** account, baseline OUs (Security, Sandbox), an org CloudTrail, and an **Account Factory** to vend accounts with baselines.

It applies **controls** (formerly guardrails) and offers a dashboard for drift. Customizations come through Account Factory for Terraform (AFT) or Customizations for Control Tower (CfCT).

</details>

### Q39. What types of Control Tower controls exist? 🟡

<details>
<summary><b>Answer</b></summary>

- **Preventive**: enforced with SCPs (and RCPs), block the action outright.
- **Detective**: implemented as AWS Config rules, report non-compliance after the fact.
- **Proactive**: CloudFormation Hooks that block non-compliant resources before provisioning.

Each is categorized by behavior: mandatory, strongly recommended, or elective. Preventive controls cost nothing; detective controls incur Config charges.

</details>

### Q40. How can you grant another account access to an S3 bucket? Compare the options. 🟡

<details>
<summary><b>Answer</b></summary>

1. **Bucket policy** naming the other account's principal (or `aws:PrincipalOrgID` for the whole Org) plus an identity policy on their side.
2. **Cross-account role**: they assume a role in your account that has S3 access. Credentials are scoped to your account and logged there.
3. **Access Points** with their own policy, good for many consumers.
4. **Presigned URLs** for ad-hoc, time-limited sharing.

We prefer role assumption or org-conditioned bucket policies; avoid ACLs and enable BucketOwnerEnforced.

</details>

### Q41. What are the pitfalls of `NotAction` and `NotResource`? 🔴

<details>
<summary><b>Answer</b></summary>

`NotAction` with `Allow` means "allow everything except these", which silently grants every future API. `NotResource` with `Allow` similarly allows all other resources. They are only safe on **Deny** statements, where they express "deny everything except".

```json
{"Effect":"Deny","NotAction":["iam:*MFADevice*","sts:GetSessionToken"],"Resource":"*"}
```

If you see `Allow` + `NotAction` in a review, treat it as a red flag.

</details>

### Q42. How does attribute-based access control (ABAC) work in AWS? 🔴

<details>
<summary><b>Answer</b></summary>

Instead of one policy per project, you tag principals and resources and write a single policy comparing tags, for example allow `ec2:StartInstances` only when `aws:ResourceTag/Project` equals `aws:PrincipalTag/Project`.

```json
"Condition": {"StringEquals": {"aws:ResourceTag/Project": "${aws:PrincipalTag/Project}"}}
```

It scales with teams, but only works if tagging is protected: deny `iam:TagRole`/`ec2:CreateTags` changes on the controlling tags, and know that not every service supports tag-based authorization.

</details>

### Q43. What does `iam:PassRole` do and why is it dangerous? 🔴

<details>
<summary><b>Answer</b></summary>

`iam:PassRole` is the permission to hand a role to an AWS service (an EC2 instance, a Lambda function, an ECS task, CloudFormation). It is not an API call; it is checked when you call another API like `lambda:CreateFunction --role`.

If a user can pass an admin role to a service they can create, they effectively get admin through that service. Scope it with the `Resource` (specific role ARNs) and the `iam:PassedToService` condition.

</details>

### Q44. What is the maximum session duration of a role and how do you change it? 🟢

<details>
<summary><b>Answer</b></summary>

Default 1 hour, configurable from 1 hour to 12 hours per role. A caller can request any duration from 15 minutes up to that cap. Role chaining is hard-capped at 1 hour, and console role switching also respects the role's max.

```bash
aws iam update-role --role-name deploy --max-session-duration 7200
```

</details>

### Q45. How do you test whether a policy will allow an action before deploying it? 🟢

<details>
<summary><b>Answer</b></summary>

Use the **IAM Policy Simulator** (console) or the CLI, which evaluates identity policies, boundaries and optionally SCPs against a list of actions and resources.

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/app \
  --action-names s3:GetObject --resource-arns arn:aws:s3:::acme-data/x.csv
```

The simulator does not model every resource policy or condition context key, so confirm with a real call in a sandbox.

</details>

### Q46. What is IAM Roles Anywhere? 🟡

<details>
<summary><b>Answer</b></summary>

It lets workloads outside AWS (on-prem servers, other clouds) get temporary role credentials using **X.509 certificates** issued by your CA instead of long-lived access keys. You register a trust anchor (your CA or ACM Private CA), create a profile that maps to roles, and use the `aws_signing_helper` as a credential process.

```bash
aws_signing_helper credential-process --certificate cert.pem --private-key key.pem \
  --trust-anchor-arn $TA --profile-arn $PROFILE --role-arn $ROLE
```

</details>

### Q47. What are Resource Control Policies (RCPs)? 🔴

<details>
<summary><b>Answer</b></summary>

RCPs are an Organizations policy type (introduced late 2024) that sets the maximum permissions on **resources** in your accounts, regardless of who calls, including external principals. SCPs constrain your principals; RCPs constrain what can be done to your resources.

A typical use is a data perimeter: deny access to S3, KMS, SQS, STS and Secrets Manager resources unless `aws:PrincipalOrgID` equals your org, closing the hole that an SCP cannot cover (an outsider with a permissive bucket policy).

</details>

### Q48. Do SCPs apply to the management account? 🟢

<details>
<summary><b>Answer</b></summary>

No. SCPs never restrict the management account or its users and roles, which is a major reason to keep it empty of workloads and to guard it with strong controls like hardware MFA and minimal access. They also do not restrict service-linked roles.

</details>

### Q49. How consistent is IAM, and what does that mean for automation? 🟢

<details>
<summary><b>Answer</b></summary>

IAM is **eventually consistent**: a newly created role, policy change or attachment may take seconds to propagate across Regions' IAM endpoints. Scripts that create a role and immediately use it can fail with `InvalidParameterValueException: The role defined for the function cannot be assumed by Lambda` or `AccessDenied`.

Terraform and CloudFormation add retries; in shell scripts add a short wait-and-retry loop and avoid putting IAM changes in the critical path of high-frequency flows.

</details>

### Q50. How do you set up SSO-based CLI access? 🟢

<details>
<summary><b>Answer</b></summary>

Configure an SSO profile once, then log in to get cached short-lived credentials.

```bash
aws configure sso          # prompts for start URL, region, account, permission set
aws sso login --profile prod-admin
aws s3 ls --profile prod-admin
```

Profiles live in `~/.aws/config` as `sso_session`, `sso_account_id` and `sso_role_name`. No access keys are stored on disk, and `aws sso logout` clears the cache.

</details>

## VPC and Networking

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. What is a VPC and what are the CIDR constraints? 🟢

<details>
<summary><b>Answer</b></summary>

A VPC is a logically isolated virtual network in one Region, spanning all AZs. The primary IPv4 CIDR must be between **/16 and /28**; you can add secondary CIDRs later. Use RFC 1918 ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) and plan them centrally, because overlapping CIDRs make peering and Transit Gateway routing impossible.

</details>

### Q52. What makes a subnet public or private? 🟢

<details>
<summary><b>Answer</b></summary>

A subnet is **public** if its route table has a route `0.0.0.0/0` to an **Internet Gateway** (and instances have a public/Elastic IP). A **private** subnet has no direct IGW route; outbound internet goes via a NAT gateway in a public subnet or not at all.

A subnet lives in exactly one AZ, so each tier needs a subnet per AZ.

</details>

### Q53. How do route tables work, including the local route and longest-prefix match? 🟡

<details>
<summary><b>Answer</b></summary>

Every subnet is associated with one route table (the main one if not explicit). Every table has an immutable `local` route for the VPC CIDR. When multiple routes match, the **most specific prefix wins**; for equal prefixes, static routes beat propagated ones and prefix-list/endpoint routes beat others.

```bash
aws ec2 create-route --route-table-id rtb-0abc \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-0def
```

</details>

### Q54. Internet Gateway vs NAT Gateway: what does each do? 🟢

<details>
<summary><b>Answer</b></summary>

An **IGW** is a horizontally scaled, redundant gateway that provides two-way internet connectivity for resources with public IPs, performing 1:1 NAT between private and public addresses. A **NAT gateway** lets instances in private subnets initiate outbound connections to the internet while blocking inbound connections initiated from outside.

NAT gateways are AZ-scoped and live in a public subnet with an Elastic IP.

</details>

### Q55. NAT Gateway vs NAT instance? 🟡

<details>
<summary><b>Answer</b></summary>

The managed **NAT gateway** scales to 100 Gbps, is highly available within an AZ, and needs no patching, but charges per hour plus per GB processed. A **NAT instance** is an EC2 you manage with source/destination check disabled; it is cheaper at tiny scale and allows port forwarding, security groups and custom tooling, but you own patching, failover and throughput limits.

In production we use NAT gateways, one per AZ. NAT instances survive only in dev environments where cost matters.

</details>

### Q56. Security group vs network ACL: compare them. 🟢

<details>
<summary><b>Answer</b></summary>

| | Security group | Network ACL |
|---|---|---|
| Level | ENI / instance | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and Deny, numbered |
| Evaluation | All rules together | Lowest number first |
| Default | Deny in, allow out | Default NACL allows all |

Security groups are the primary control; NACLs are a coarse backstop, for example to block a bad CIDR subnet-wide.

</details>

### Q57. What are the nuances of security group statefulness? 🔴

<details>
<summary><b>Answer</b></summary>

Security groups track connections, so return traffic is allowed automatically. Two surprises: if a SG rule is **removed**, already-tracked connections continue until they idle out or close; and if a rule allows `0.0.0.0/0` on all ports for both directions, the connection is **untracked**, so removing the rule does take effect immediately.

Also, tracking has idle timeouts (350 seconds for established TCP by default), which affects long-idle connections through NLB or firewalls and is fixed with TCP keepalives.

</details>

### Q58. Why do NACLs need ephemeral port rules? 🟡

<details>
<summary><b>Answer</b></summary>

NACLs are stateless, so return traffic is evaluated as a new flow. A client connecting to port 443 receives replies on a random **ephemeral port** (1024–65535 for most Linux and Windows; 49152–65535 per IANA). You must allow outbound ephemeral ports on the server subnet and inbound ephemeral ports on the client subnet.

Forgetting this is the classic reason a "working" SG setup breaks after someone tightens the NACL.

</details>

### Q59. Gateway endpoints vs interface endpoints? 🟡

<details>
<summary><b>Answer</b></summary>

A **gateway endpoint** exists only for S3 and DynamoDB. It adds a prefix-list route to your route table, costs nothing and is not reachable from on-prem or other VPCs. An **interface endpoint** (PrivateLink) puts an ENI with a private IP in your subnets for most other services, supports security groups and private DNS, is reachable over VPN/DX/TGW, and bills per hour and per GB.

Both support endpoint policies to restrict which buckets or principals can be used through them.

</details>

### Q60. How does AWS PrivateLink work? 🔴

<details>
<summary><b>Answer</b></summary>

A provider fronts its service with an NLB (or GWLB) and publishes a **VPC endpoint service**. Consumers create an interface endpoint in their VPC; traffic goes over the AWS network to the provider NLB with no peering, no overlapping-CIDR concerns, and no public IPs. The connection is unidirectional (consumer initiates).

The provider sees traffic from the NLB's IPs unless Proxy Protocol v2 is enabled. Endpoint services can require acceptance and be restricted by allowed principals.

</details>

### Q61. What are the limitations of VPC peering? 🟡

<details>
<summary><b>Answer</b></summary>

CIDRs must not overlap. Peering is **non-transitive**: A to B and B to C does not give A to C, and you cannot use a peer's IGW, NAT, VPN or endpoint (no edge-to-edge routing). Each side needs routes and SG/NACL rules, and it works cross-account and cross-Region (traffic stays on the backbone, encrypted).

Peering has no hourly charge, only data transfer, which makes it cheap for a few VPC pairs but unmanageable as a full mesh.

</details>

### Q62. Transit Gateway vs VPC peering? 🟡

<details>
<summary><b>Answer</b></summary>

Transit Gateway is a regional hub-and-spoke router: attach VPCs, VPNs, Direct Connect gateways and peer TGWs in other Regions, with transitive routing and multiple route tables for segmentation. It charges per attachment-hour plus per GB processed.

Peering is cheaper and lower latency for a couple of VPCs. Past roughly 4–5 VPCs, or whenever on-prem connectivity or segmentation is needed, TGW is easier to operate.

</details>

### Q63. Explain Transit Gateway route table association and propagation. 🔴

<details>
<summary><b>Answer</b></summary>

Each attachment is **associated** with exactly one TGW route table, which decides where traffic *from* that attachment goes. An attachment **propagates** its CIDRs into one or more route tables, which decides who can reach *it*.

For isolation (prod cannot reach dev, both reach shared services) use separate route tables: associate prod and dev attachments with their own tables, propagate shared-services routes into both, and propagate prod/dev routes only into the shared table. VPC route tables still need a route to the TGW.

</details>

### Q64. Direct Connect vs Site-to-Site VPN? 🟡

<details>
<summary><b>Answer</b></summary>

**Direct Connect** is a dedicated private circuit (1, 10 or 100 Gbps dedicated; hosted connections from partners) with consistent latency and lower egress rates, but takes weeks to provision and is not encrypted by default (add MACsec or run a VPN over it). **Site-to-Site VPN** is IPsec over the internet, up in minutes, two tunnels per connection, about 1.25 Gbps per tunnel.

The common production design is DX primary with VPN backup, using BGP so failover is automatic.

</details>

### Q65. How many IP addresses does AWS reserve in each subnet? 🟢

<details>
<summary><b>Answer</b></summary>

Five: the first four and the last one. For `10.0.1.0/24`: `.0` network, `.1` VPC router, `.2` DNS (Amazon-provided resolver), `.3` reserved for future use, and `.255` broadcast (not supported but reserved). So a /24 gives 251 usable addresses and a /28 only 11.

</details>

### Q66. Elastic IP vs auto-assigned public IP? 🟢

<details>
<summary><b>Answer</b></summary>

An auto-assigned public IPv4 is released when the instance stops and changes on restart. An **Elastic IP** is a static address you own until released, and can be remapped between instances or ENIs for failover. Since February 2024 AWS charges for **every** public IPv4 address, in use or not, so idle EIPs cost money.

Prefer load balancers and DNS names over EIPs; use EIPs for NAT gateways and allowlisted egress.

</details>

### Q67. What are VPC Flow Logs and what do they capture? 🟡

<details>
<summary><b>Answer</b></summary>

Flow Logs record metadata about IP traffic at VPC, subnet or ENI level: source/destination IP and port, protocol, packets, bytes, `ACCEPT` or `REJECT`, and with custom format also `pkt-srcaddr`, `flow-direction`, `tcp-flags` and the AZ. They do not capture payloads, DNS to the Amazon resolver, DHCP or instance metadata traffic.

Deliver to S3 (cheapest, query with Athena) or CloudWatch Logs. A `REJECT` on a SG shows as one-way, a NACL reject shows both directions.

</details>

### Q68. How does DNS work inside a VPC? 🟡

<details>
<summary><b>Answer</b></summary>

The Amazon-provided resolver lives at the VPC base +2 (for example `10.0.0.2`) and at `169.254.169.253`. Two VPC attributes control it: `enableDnsSupport` (resolver on) and `enableDnsHostnames` (instances get public DNS names, and interface endpoint private DNS requires both).

Private hosted zones resolve only for associated VPCs. For hybrid DNS use Route 53 Resolver inbound and outbound endpoints. The resolver limit is 1,024 packets per second per ENI.

</details>

### Q69. How does IPv6 work in a VPC, and what is an egress-only internet gateway? 🟡

<details>
<summary><b>Answer</b></summary>

You can attach an Amazon-provided /56 IPv6 CIDR, and subnets get /64s. Every IPv6 address is globally unique and public-capable, so there is no NAT. To allow private subnets to reach the internet but block inbound, route `::/0` to an **egress-only internet gateway**.

Dual-stack saves public IPv4 charges and NAT costs, but security groups and NACLs need separate IPv6 rules.

</details>

### Q70. What do you do when a VPC runs out of IP addresses? 🟡

<details>
<summary><b>Answer</b></summary>

Add a secondary CIDR block to the VPC (up to five by default) and create new subnets from it. For EKS pod-IP exhaustion use the `100.64.0.0/10` carrier-grade range for pods via custom networking, or enable prefix delegation, or move to IPv6.

Long term, fix the root cause with a central IPAM plan (AWS VPC IPAM) and right-sized subnets, since you cannot resize an existing subnet.

</details>

### Q71. Bastion host vs SSM Session Manager? 🟢

<details>
<summary><b>Answer</b></summary>

A bastion is a public EC2 you SSH through; it needs an open port, key management and patching. **Session Manager** uses the SSM agent over outbound HTTPS to the SSM service, requires no inbound ports or public IPs, authenticates through IAM, and logs sessions to CloudTrail, S3 or CloudWatch.

```bash
aws ssm start-session --target i-0abc123
```

For private instances you need NAT or the `ssm`, `ssmmessages` and `ec2messages` interface endpoints.

</details>

### Q72. What are ENA, EFA and an ENI? 🔴

<details>
<summary><b>Answer</b></summary>

An **ENI** is a virtual network card (private IPs, MAC, security groups) that can be detached and moved between instances in the same AZ. **ENA** is the SR-IOV driver giving up to 100–200 Gbps on Nitro instances. **EFA** is an ENA-plus device with OS-bypass (libfabric/SRD) for HPC and distributed ML, offering lower latency and higher throughput for MPI/NCCL.

The number of ENIs and IPs per instance depends on type, which limits pod density in EKS with the VPC CNI.

</details>

### Q73. What is the default VPC and should you use it? 🟢

<details>
<summary><b>Answer</b></summary>

Each Region in each account starts with a default VPC (`172.31.0.0/16`) with a public subnet per AZ, an IGW and permissive defaults, so instances launched without a subnet are internet-facing. It is convenient for demos but wrong for production.

We delete default VPCs in all Regions at account vending time, and an SCP/Config rule alerts if a new one appears.

</details>

### Q74. What is a Gateway Load Balancer? 🔴

<details>
<summary><b>Answer</b></summary>

GWLB is a layer-3 load balancer that transparently inserts a fleet of virtual appliances (firewalls, IDS) into the traffic path. Traffic is encapsulated with **GENEVE** (port 6081) to appliances, preserving original source and destination, and returned the same way. It combines a transparent gateway with load balancing and health checks, and is consumed through a Gateway Load Balancer endpoint with route table entries.

It is the building block for centralized inspection VPCs.

</details>

### Q75. How do you debug "why can't A reach B" in a VPC? 🟡

<details>
<summary><b>Answer</b></summary>

Use **VPC Reachability Analyzer**: it models the configuration (route tables, SGs, NACLs, endpoints, TGW) between a source and destination and tells you the blocking component. Then confirm with Flow Logs (`REJECT` entries) and `nc -zv host port` from the instance.

```bash
aws ec2 create-network-insights-path --source eni-0a --destination eni-0b \
  --protocol tcp --destination-port 5432
aws ec2 start-network-insights-analysis --network-insights-path-id nip-0abc
```

</details>

## EC2, Auto Scaling and Load Balancing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. How are EC2 instance types named? 🟢

<details>
<summary><b>Answer</b></summary>

`m7g.2xlarge` decodes as family `m` (general purpose), generation `7`, attribute `g` (Graviton/ARM; `i` = Intel, `a` = AMD, `d` = local NVMe, `n` = network optimized), size `2xlarge`. Families: `t`/`m` general, `c` compute, `r`/`x` memory, `i`/`d` storage, `p`/`g` GPU, `hpc`.

Newer generations are usually cheaper per unit of performance, and Graviton (`g`) types often give 20–40% better price-performance for compatible workloads.

</details>

### Q77. What are the EC2 purchasing options? 🟢

<details>
<summary><b>Answer</b></summary>

- **On-Demand**: pay per second, no commitment.
- **Savings Plans** (Compute or EC2 Instance): 1 or 3 year dollar-per-hour commitment, up to ~70% off.
- **Reserved Instances**: older commitment model tied to instance attributes.
- **Spot**: spare capacity at up to ~90% off, can be reclaimed with 2 minutes' notice.
- **Dedicated Hosts/Instances**: single-tenant hardware for licensing or compliance.
- **Capacity Reservations**: guarantee capacity in an AZ, billed whether used or not.

</details>

### Q78. What is an AMI and what does it contain? 🟢

<details>
<summary><b>Answer</b></summary>

An Amazon Machine Image is a template with the root volume snapshot (or instance-store image), launch permissions, and block device mappings. AMIs are **regional**; copy them (`aws ec2 copy-image`) to use elsewhere, and share them to accounts or the Org with launch permissions. Encrypted AMIs need the KMS key shared too.

Use EC2 Image Builder or Packer to bake AMIs and look up the latest through an SSM public parameter:

```bash
aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text
```

</details>

### Q79. How does EC2 user data work? 🟡

<details>
<summary><b>Answer</b></summary>

User data (up to 16 KB) is passed to the instance and executed by **cloud-init** as root on first boot only by default; it can be a shell script (starting with `#!`) or `#cloud-config` YAML. Output goes to `/var/log/cloud-init-output.log`. Changing user data requires the instance to be stopped, and to rerun it you must clear cloud-init state or use a `#cloud-boothook`/`scripts-user: always` setting.

User data is visible via the metadata service, so never put secrets in it.

</details>

### Q80. Compare the main EBS volume types. 🟡

<details>
<summary><b>Answer</b></summary>

- **gp3**: SSD default, 3,000 IOPS and 125 MB/s baseline included, scalable independently to 16,000 IOPS and 1,000 MB/s.
- **gp2**: older, IOPS tied to size (3 per GB) with burst credits.
- **io2 / io2 Block Express**: provisioned IOPS up to 256,000 for databases needing sub-millisecond latency and 99.999% durability.
- **st1**: throughput HDD for big sequential workloads (logs, Kafka).
- **sc1**: cold HDD, cheapest.

Only SSD types can be boot volumes. Default recommendation: gp3 everywhere, io2 for latency-critical databases.

</details>

### Q81. EBS vs instance store? 🟢

<details>
<summary><b>Answer</b></summary>

EBS is network-attached, persistent block storage with snapshots, detach/reattach and resize. **Instance store** is NVMe or HDD physically on the host: very fast and included in price, but data is lost on stop, terminate or hardware failure (it survives reboot).

Use instance store for caches, scratch space, shuffle data and replicated systems like Cassandra/Kafka brokers with replication; use EBS for anything you cannot lose.

</details>

### Q82. How do EBS snapshots work and what is lazy loading? 🔴

<details>
<summary><b>Answer</b></summary>

Snapshots are stored in S3 (managed by AWS) and are **incremental at block level**: the first is full, later ones store changed blocks, and deleting an older snapshot keeps data still referenced by newer ones. A volume restored from a snapshot is usable immediately but blocks are fetched lazily from S3 on first read, causing high latency on first access.

Mitigate with **Fast Snapshot Restore** (pre-warmed per AZ, billed) or by pre-reading with `fio`/`dd` for critical restores.

</details>

### Q83. What are placement groups? 🟡

<details>
<summary><b>Answer</b></summary>

- **Cluster**: pack instances close in one AZ for low-latency, high-throughput networking (HPC, tightly coupled jobs); higher risk of correlated failure.
- **Spread**: place each instance on distinct hardware, up to 7 per AZ per group, for a few critical instances.
- **Partition**: groups of instances on separate racks (up to 7 partitions per AZ) for large distributed systems like HDFS, Cassandra and Kafka, with partition metadata available to the application.

A `InsufficientInstanceCapacity` error is common with cluster groups; launch all instances at once and use one instance type.

</details>

### Q84. What is IMDSv2 and why enforce it? 🟡

<details>
<summary><b>Answer</b></summary>

IMDSv2 is the session-oriented instance metadata service: you first `PUT` to get a token, then pass it as a header. It blocks SSRF-style attacks that trick an app into making a simple GET (the 2019 Capital One breach pattern) and rejects requests with `X-Forwarded-For`.

```bash
TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

Enforce with `--metadata-options HttpTokens=required`; set `HttpPutResponseHopLimit=2` for containers on the host.

</details>

### Q85. How do burstable T instances work? 🟡

<details>
<summary><b>Answer</b></summary>

T-family instances have a baseline CPU percentage per vCPU and earn **CPU credits** while below it, spending them when above. In **standard** mode, exhausting credits throttles CPU to baseline; in **unlimited** mode (default for T3 and later) they keep bursting and you pay a surplus-credit charge.

Monitor `CPUCreditBalance` and `CPUSurplusCreditBalance`. If surplus charges persist, move to an `m` type, which is cheaper than a sustained-burst T.

</details>

### Q86. What is the difference between stopping, terminating and hibernating an instance? 🟢

<details>
<summary><b>Answer</b></summary>

**Stop**: instance shuts down, EBS volumes persist, no compute charge, the instance may move hosts on start and loses public IP and instance store data. **Terminate**: instance is deleted; root volume goes away unless `DeleteOnTermination` is false. **Hibernate**: RAM contents are saved to the encrypted root EBS volume and restored on start, preserving process state.

Termination protection (`--disable-api-termination`) blocks API terminations, but not ASG scale-in or shutdown-behavior terminate unless you configure it.

</details>

### Q87. How do Spot interruptions work? 🔴

<details>
<summary><b>Answer</b></summary>

When AWS needs the capacity back it sends a **2-minute interruption notice** via instance metadata (`/latest/meta-data/spot/instance-action`) and an EventBridge event, then stops or terminates the instance. An earlier, non-guaranteed **rebalance recommendation** signals elevated risk.

Handle it by draining work, checkpointing, and using Capacity Rebalancing in ASGs. Diversify across many instance types and AZs with the `price-capacity-optimized` allocation strategy to cut interruptions.

</details>

### Q88. Dedicated Hosts vs Dedicated Instances? 🟡

<details>
<summary><b>Answer</b></summary>

Both give single-tenant hardware. A **Dedicated Host** is a whole physical server you control: visibility of sockets and cores, host affinity, and the ability to bring per-socket/per-core licenses (BYOL for Windows Server, SQL Server, Oracle). A **Dedicated Instance** runs on single-tenant hardware but without host visibility or placement control, billed per instance plus a regional fee.

Pick Hosts when licensing requires it; otherwise Dedicated Instances satisfy basic isolation requirements.

</details>

### Q89. What are the components of an Auto Scaling group? 🟢

<details>
<summary><b>Answer</b></summary>

An ASG needs a **launch template** (AMI, type, SGs, IAM profile, user data), subnets across AZs, and **min / desired / max** capacity. Optionally add a load balancer target group, health checks, scaling policies, lifecycle hooks and a mixed-instances policy for On-Demand/Spot blends.

```bash
aws autoscaling create-auto-scaling-group --auto-scaling-group-name web \
  --launch-template LaunchTemplateName=web,Version='$Latest' \
  --min-size 2 --max-size 10 --vpc-zone-identifier "subnet-a,subnet-b" \
  --target-group-arns $TG_ARN --health-check-type ELB --health-check-grace-period 120
```

</details>

### Q90. What scaling policy types does Auto Scaling offer? 🟡

<details>
<summary><b>Answer</b></summary>

- **Target tracking**: hold a metric near a target (for example `ASGAverageCPUUtilization` at 50 or `ALBRequestCountPerTarget`). Simplest and the default choice.
- **Step scaling**: add or remove capacity in steps by alarm breach size.
- **Simple scaling**: legacy, waits for cooldown.
- **Scheduled**: known patterns.
- **Predictive**: ML forecast from 14 days of history, pre-scales ahead of cycles.

Target tracking policies create and own their CloudWatch alarms; do not edit them by hand.

</details>

### Q91. What are Auto Scaling lifecycle hooks? 🔴

<details>
<summary><b>Answer</b></summary>

Hooks pause an instance in `Pending:Wait` (on launch) or `Terminating:Wait` (on terminate) so you can run custom actions: register with a service discovery system, warm a cache, drain connections, upload logs. The instance waits until you call `complete-lifecycle-action` or the heartbeat timeout expires (default 1 hour, with `CONTINUE` or `ABANDON` as default result).

```bash
aws autoscaling complete-lifecycle-action --lifecycle-hook-name drain \
  --auto-scaling-group-name web --lifecycle-action-result CONTINUE --instance-id i-0abc
```

Notifications go through EventBridge, SNS or SQS.

</details>

### Q92. What are ASG health checks and the grace period? 🟡

<details>
<summary><b>Answer</b></summary>

**EC2** health checks only detect failed instance status. **ELB** health checks (enable with `--health-check-type ELB`) also replace instances that the load balancer marks unhealthy, which is what you want for web tiers. The **grace period** (default 300 s) delays health checks after launch so the app can boot.

If the grace period is shorter than boot time, you get a replace loop where instances are terminated before they ever become healthy.

</details>

### Q93. What is an Auto Scaling instance refresh? 🟡

<details>
<summary><b>Answer</b></summary>

Instance refresh rolls the group to a new launch template version by replacing instances in batches while keeping a minimum healthy percentage, with optional checkpoints, warmup time and automatic rollback on CloudWatch alarm.

```bash
aws autoscaling start-instance-refresh --auto-scaling-group-name web \
  --preferences '{"MinHealthyPercentage":90,"InstanceWarmup":120,"AutoRollback":true}'
```

It is the native rolling-deployment mechanism for AMI-based deployments.

</details>

### Q94. ALB vs NLB vs GWLB vs Classic: when to use which? 🟢

<details>
<summary><b>Answer</b></summary>

**ALB** (layer 7): HTTP/HTTPS/gRPC, path/host/header routing, WebSockets, OIDC auth, WAF. **NLB** (layer 4): TCP/UDP/TLS, millions of requests per second, static or Elastic IPs per AZ, preserves source IP. **GWLB** (layer 3): transparent appliance insertion. **Classic**: legacy, EC2-Classic-era, migrate away.

Default to ALB for web apps and microservices, NLB for non-HTTP protocols, PrivateLink fronting, and extreme throughput.

</details>

### Q95. How do ALB listeners, rules and target groups fit together? 🟡

<details>
<summary><b>Answer</b></summary>

A **listener** (port and protocol) holds ordered **rules**, each with conditions (host header, path pattern, HTTP header, query string, source IP) and an action (forward, redirect, fixed-response, authenticate). Forward actions point to **target groups** (instances, IPs, Lambda) which carry the health check, protocol and stickiness settings. You can forward to multiple weighted target groups for canaries.

Rules evaluate by priority; the default rule catches the rest.

</details>

### Q96. How does cross-zone load balancing differ between ALB and NLB? 🔴

<details>
<summary><b>Answer</b></summary>

With cross-zone enabled each load balancer node distributes across targets in **all** AZs equally. **ALB**: always on at the load balancer level (can be turned off per target group) with no inter-AZ data charge. **NLB** and **GWLB**: off by default; enabling it incurs cross-AZ data transfer charges.

With it off, uneven target counts per AZ produce uneven load, for example 2 targets in one AZ and 8 in another each getting 50% of traffic.

</details>

### Q97. What is deregistration delay? 🟡

<details>
<summary><b>Answer</b></summary>

When a target is deregistered or goes unhealthy-draining, the load balancer stops sending new requests but lets in-flight requests finish for the **deregistration delay** (default 300 seconds, range 0–3600). After it expires, remaining connections are closed.

Set it to match your longest request (for example 30 seconds for an API) to speed up deployments. The app must also handle SIGTERM and keep serving until draining finishes.

</details>

### Q98. What are the NLB features that make it different from ALB? 🟡

<details>
<summary><b>Answer</b></summary>

NLB gives a static IP (or Elastic IP) per AZ, preserves the client source IP for instance targets (and optionally IP targets), handles sudden spikes without pre-warming, supports TLS termination, UDP, and security groups (added in 2023). It does not understand HTTP, so no path routing, no WAF integration and no cookies.

Idle TCP timeout is 350 seconds; clients must send keepalives for longer idle connections.

</details>

### Q99. Why would an ALB target show as unhealthy? 🟢

<details>
<summary><b>Answer</b></summary>

The usual causes: the health check path returns a non-matching status (default success is 200), the target security group does not allow the ALB's SG on the health check port, the app is not listening on that port, the check times out (default 5 s timeout, 30 s interval, 5 healthy / 2 unhealthy thresholds), or the app redirects to HTTPS and returns 301.

```bash
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State,TargetHealth.Reason]'
```

The `Reason` field (`Target.Timeout`, `Target.ResponseCodeMismatch`) points directly at the cause.

</details>

### Q100. What do ALB 502, 503 and 504 errors mean? 🟡

<details>
<summary><b>Answer</b></summary>

- **502 Bad Gateway**: the target closed the connection, sent a malformed response, or its keep-alive timeout is shorter than the ALB idle timeout (60 s).
- **503 Service Unavailable**: no healthy targets registered, or the target group is empty.
- **504 Gateway Timeout**: the target did not respond within the idle timeout, or connection to it timed out (often an SG or NACL issue).

Fix the common 502 by setting the application keep-alive timeout higher than the ALB idle timeout (for example Nginx `keepalive_timeout 75s`).

</details>

## Storage and Databases

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. Compare the S3 storage classes. 🟢

<details>
<summary><b>Answer</b></summary>

- **Standard**: frequent access, no retrieval fee.
- **Intelligent-Tiering**: automatic tiering by access pattern, small monitoring fee per object.
- **Standard-IA / One Zone-IA**: infrequent access, 30-day minimum, 128 KB minimum billable size, retrieval fee; One Zone stores in a single AZ.
- **Glacier Instant Retrieval**: archive with millisecond access, 90-day minimum.
- **Glacier Flexible Retrieval**: minutes to hours, 90-day minimum.
- **Glacier Deep Archive**: cheapest, hours, 180-day minimum.
- **Express One Zone**: single-digit-ms directory buckets for hot data.

All except One Zone and Express are stored across at least 3 AZs at 11 nines durability.

</details>

### Q102. What is S3's consistency model? 🟡

<details>
<summary><b>Answer</b></summary>

Since December 2020 S3 offers **strong read-after-write consistency** for all object PUTs, overwrites and DELETEs, and strongly consistent LIST operations, in all Regions. There is no longer a need for workarounds like random key-prefix hacks or consistency layers such as S3Guard.

Bucket-level configuration changes (policies, lifecycle) are still eventually consistent, so allow a few seconds.

</details>

### Q103. How does S3 versioning and delete work? 🟢

<details>
<summary><b>Answer</b></summary>

With versioning enabled, every overwrite creates a new version ID and a plain `DELETE` just adds a **delete marker**, hiding the object while keeping all versions. To permanently delete you must specify the `VersionId`. Versioning cannot be disabled once enabled, only suspended.

```bash
aws s3api list-object-versions --bucket acme-data --prefix report.csv
aws s3api delete-object --bucket acme-data --key report.csv --version-id <id>
```

Pair with lifecycle rules for noncurrent versions or storage grows forever.

</details>

### Q104. How do S3 lifecycle rules work? 🟡

<details>
<summary><b>Answer</b></summary>

A rule filters by prefix, tag or size and applies actions: **transition** to another class after N days, **expire** current objects, expire noncurrent versions (optionally keeping the newest N), remove expired delete markers, and **abort incomplete multipart uploads**.

```json
{"Rules":[{"ID":"logs","Status":"Enabled","Filter":{"Prefix":"logs/"},
 "Transitions":[{"Days":30,"StorageClass":"STANDARD_IA"},{"Days":90,"StorageClass":"GLACIER"}],
 "Expiration":{"Days":365}}]}
```

Minimum-duration constraints apply: transitioning to Standard-IA requires 30 days in Standard first.

</details>

### Q105. How do bucket policies, ACLs and Block Public Access interact? 🟡

<details>
<summary><b>Answer</b></summary>

**Block Public Access** (account and bucket level) overrides public grants from policies and ACLs, and is the master safety switch. **Bucket policies** are the main resource-based mechanism. **ACLs** are legacy and disabled by default on new buckets via **Object Ownership = BucketOwnerEnforced**, which makes the bucket owner own all objects.

The modern baseline: BPA on, ACLs disabled, access only via policies, and `aws:SecureTransport` deny.

</details>

### Q106. What are the S3 server-side encryption options? 🔴

<details>
<summary><b>Answer</b></summary>

- **SSE-S3** (AES-256, keys managed by S3), now applied by default to all new objects.
- **SSE-KMS**: uses a KMS key, gives audit trail in CloudTrail and key-policy control, subject to KMS request quotas and cost (use **Bucket Keys** to cut KMS calls up to 99%).
- **DSSE-KMS**: dual-layer encryption for compliance.
- **SSE-C**: you supply the key on every request; AWS does not store it.

Client-side encryption is separate. Use a bucket policy to deny `PutObject` without the required encryption header if you need to enforce a specific type.

</details>

### Q107. How do presigned URLs work? 🟡

<details>
<summary><b>Answer</b></summary>

A presigned URL embeds a SigV4 signature made with the signer's credentials, granting temporary access to one operation (GET or PUT) on one object. It has the **permissions of the signer** and expires at the earlier of the URL expiry (max 7 days with IAM user keys) or the signing credential's expiry (for role sessions, as short as the session).

```bash
aws s3 presign s3://acme-data/report.pdf --expires-in 900
```

If the signing identity loses permission, the URL stops working.

</details>

### Q108. How does S3 replication work and what does it require? 🟡

<details>
<summary><b>Answer</b></summary>

Cross-Region (CRR) and Same-Region (SRR) replication asynchronously copy new objects to a destination bucket. Requirements: **versioning enabled on both buckets**, an IAM role S3 can assume for replication, and (for KMS) permission on both keys. Existing objects need **S3 Batch Replication**. Delete markers replicate optionally; permanent version deletes do not.

S3 Replication Time Control gives a 15-minute SLA for 99.99% of objects. Replication is not transitive: A to B to C needs separate rules.

</details>

### Q109. Why use multipart upload and what is the cleanup trap? 🟡

<details>
<summary><b>Answer</b></summary>

Multipart splits large objects into parts (5 MiB to 5 GiB each, up to 10,000 parts, max 5 TiB object) uploaded in parallel and resumable. Recommended above 100 MB, required above 5 GB per PUT. Incomplete uploads are **invisible in listings but billed**.

Add a lifecycle rule `AbortIncompleteMultipartUpload` with `DaysAfterInitiation: 7`, and audit with `aws s3api list-multipart-uploads`.

</details>

### Q110. What is S3 Object Lock? 🔴

<details>
<summary><b>Answer</b></summary>

Object Lock gives WORM protection on object versions, requiring versioning. **Governance mode**: users with `s3:BypassGovernanceRetention` can override. **Compliance mode**: nobody, including root, can delete or shorten retention until it expires. **Legal hold** is an independent on/off flag with no expiry.

It is the basis for SEC 17a-4 and ransomware-resistant backups. Choose compliance mode carefully: a mistake with a long retention cannot be undone.

</details>

### Q111. What request rate can S3 sustain and what is a prefix? 🔴

<details>
<summary><b>Answer</b></summary>

S3 scales to at least **3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD requests per second per prefix**, with no limit on the number of prefixes. A prefix is the key path up to the final object name, so spreading keys across many prefixes parallelizes throughput.

You may see `503 Slow Down` on sudden spikes to a new prefix while S3 repartitions; clients should retry with exponential backoff and jitter (the SDKs do this by default).

</details>

### Q112. What are S3 Access Points and Multi-Region Access Points? 🔴

<details>
<summary><b>Answer</b></summary>

An **Access Point** is a named endpoint with its own policy and optional VPC restriction attached to a bucket, so you can give each team or application a tailored policy instead of growing one giant bucket policy. A **Multi-Region Access Point** provides a single global endpoint that routes requests to the closest replicated bucket, with failover controls.

Delegate access control by writing a bucket policy that allows access only through access points owned by the account.

</details>

### Q113. EBS vs EFS vs S3 vs FSx: how do you choose? 🟢

<details>
<summary><b>Answer</b></summary>

**EBS**: block storage for one instance (multi-attach on io1/io2 in an AZ), low latency, AZ-scoped. **EFS**: managed NFS shared by thousands of Linux clients across AZs, elastic. **S3**: object storage, effectively unlimited, accessed by API, not mountable as a true file system. **FSx**: managed specialty file systems (Windows File Server, Lustre for HPC, NetApp ONTAP, OpenZFS).

</details>

### Q114. What is the difference between RDS Multi-AZ and read replicas? 🟢

<details>
<summary><b>Answer</b></summary>

**Multi-AZ** is for **availability**: a synchronous standby in another AZ, automatic failover (typically 60–120 seconds) with the same endpoint, and the standby cannot serve reads. **Read replicas** are for **scaling reads**: asynchronous copies with their own endpoints, can be cross-Region, and can be promoted to standalone (manual DR).

They are complementary. Production databases use Multi-AZ, and add replicas when reads dominate.

</details>

### Q115. How does an RDS Multi-AZ DB cluster differ from a Multi-AZ instance deployment? 🔴

<details>
<summary><b>Answer</b></summary>

A Multi-AZ **DB cluster** (MySQL and PostgreSQL) has one writer and **two readable standbys** in three AZs with semi-synchronous replication, giving faster commits using local NVMe, typically under 35 s failover, and a reader endpoint. The classic Multi-AZ **instance** has one non-readable synchronous standby using storage-level replication.

Choose the cluster when you want fast failover and read scaling without moving to Aurora.

</details>

### Q116. Describe Aurora's architecture. 🔴

<details>
<summary><b>Answer</b></summary>

Aurora separates compute from a **shared, distributed storage volume** that auto-grows to 128 TiB and keeps **6 copies across 3 AZs**. Writes need a 4-of-6 quorum, reads 3-of-6. Up to 15 read replicas share the same storage, so replica lag is usually under 100 ms and failover promotes a replica in around 30 seconds.

Endpoints: cluster (writer), reader (load-balanced across replicas), and custom endpoints. Backups are continuous to S3 with no performance hit.

</details>

### Q117. What are Aurora Serverless v2 and Aurora Global Database? 🔴

<details>
<summary><b>Answer</b></summary>

**Serverless v2** scales capacity in fine-grained **ACUs** (0.5 ACU steps) in place, with no connection drops, and can scale to zero (auto-pause) in recent versions. It suits spiky or unpredictable loads and dev environments.

**Global Database** replicates storage-level to up to 5 secondary Regions with typical lag under 1 second and managed failover/switchover (RTO about a minute, RPO seconds). Secondary Regions serve local reads; writes forwarded optionally.

</details>

### Q118. What is RDS Proxy and when do you need it? 🟡

<details>
<summary><b>Answer</b></summary>

RDS Proxy is a managed connection pooler between your app and RDS/Aurora. It multiplexes many client connections onto fewer database connections, preserves connections during failover (cutting failover impact by up to 66%), and supports IAM authentication with credentials from Secrets Manager.

You need it for Lambda or other bursty clients that would exhaust `max_connections`. Session pinning (temp tables, `SET` statements) reduces pooling efficiency.

</details>

### Q119. How do RDS backups and point-in-time recovery work? 🟢

<details>
<summary><b>Answer</b></summary>

RDS takes a daily automated snapshot in your backup window and archives transaction logs every 5 minutes, enabling **point-in-time restore** to any second within the retention period (1–35 days, 0 disables). Manual snapshots persist until deleted. A restore always creates a **new instance** with a new endpoint.

```bash
aws rds restore-db-instance-to-point-in-time --source-db-instance-identifier prod-db \
  --target-db-instance-identifier prod-db-restore --restore-time 2026-11-01T09:30:00Z
```

</details>

### Q120. Explain DynamoDB keys and partitioning. 🟢

<details>
<summary><b>Answer</b></summary>

A table has a **partition key** (hash) and optional **sort key** (range); together they form the primary key. DynamoDB hashes the partition key to place items on physical partitions, so high-cardinality, evenly accessed partition keys spread load. Items are up to 400 KB.

Query by key (`Query`) is efficient; `Scan` reads the whole table and should be avoided for production paths.

</details>

### Q121. DynamoDB on-demand vs provisioned capacity? 🟡

<details>
<summary><b>Answer</b></summary>

**On-demand** charges per request, scales instantly to prior peak and double, and needs no planning, best for unpredictable traffic. **Provisioned** sets RCUs/WCUs (optionally with auto scaling) and is cheaper at steady load. One RCU = one strongly consistent 4 KB read per second (two eventually consistent); one WCU = one 1 KB write per second; transactions cost double.

You can switch modes once every 24 hours.

</details>

### Q122. GSI vs LSI in DynamoDB? 🟡

<details>
<summary><b>Answer</b></summary>

A **GSI** has a different partition and sort key, its own provisioned throughput, is eventually consistent, and can be added any time. An **LSI** shares the table's partition key with an alternative sort key, supports strong consistency, **must be created at table creation**, and limits each partition key's item collection to 10 GB.

Default to GSIs. Back-pressure on a GSI can throttle writes to the base table.

</details>

### Q123. What consistency and transaction options does DynamoDB offer? 🟡

<details>
<summary><b>Answer</b></summary>

Reads are eventually consistent by default, with `ConsistentRead=true` for strong reads on the base table and LSIs (not GSIs or global tables replicas). **TransactWriteItems / TransactGetItems** give ACID across up to 100 items in one Region. Conditional writes (`ConditionExpression`) implement optimistic locking.

```bash
aws dynamodb put-item --table-name orders --item file://item.json \
  --condition-expression "attribute_not_exists(pk)"
```

</details>

### Q124. What are DynamoDB Streams, TTL and global tables? 🔴

<details>
<summary><b>Answer</b></summary>

**Streams** emit an ordered, 24-hour change log per item, used to trigger Lambda for CDC and materialized views. **TTL** deletes expired items (epoch seconds attribute) at no write cost, typically within 48 hours of expiry, and expirations appear in streams. **Global tables** provide multi-Region, multi-active replication with last-writer-wins conflict resolution.

Global tables need streams enabled and a plan for conflicting concurrent writes to the same item.

</details>

### Q125. ElastiCache Redis vs Memcached? 🟢

<details>
<summary><b>Answer</b></summary>

**Redis (and its open-source fork Valkey)** supports rich data structures, persistence, replication, Multi-AZ failover, pub/sub, Lua and cluster mode sharding. **Memcached** is a simple multithreaded in-memory key-value cache with no replication or persistence.

Choose Redis/Valkey for nearly everything; Memcached only for the simplest horizontally scaled cache. ElastiCache Valkey is priced lower than Redis OSS.

</details>

## Containers and Serverless

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. ECS vs EKS vs Fargate: what is the relationship? 🟢

<details>
<summary><b>Answer</b></summary>

**ECS** is AWS's proprietary container orchestrator, simple and deeply integrated. **EKS** is managed Kubernetes. **Fargate** is a serverless **compute engine** for both: you do not manage nodes, you only specify CPU and memory per task or pod. The alternative compute option is EC2 instances you manage.

So the choice is orchestrator (ECS or EKS) and compute (Fargate or EC2).

</details>

### Q127. What are ECS task definitions, tasks and services? 🟢

<details>
<summary><b>Answer</b></summary>

A **task definition** is the versioned blueprint: containers, images, CPU/memory, ports, env vars, roles, logging, volumes. A **task** is a running instantiation. A **service** keeps N tasks running, replaces failed ones and integrates with load balancers and deployments. One-off jobs use `run-task` without a service.

```bash
aws ecs update-service --cluster prod --service api --task-definition api:42 --force-new-deployment
```

</details>

### Q128. What are ECS capacity providers? 🟡

<details>
<summary><b>Answer</b></summary>

A capacity provider links an ECS cluster to compute: **FARGATE**, **FARGATE_SPOT**, or an EC2 Auto Scaling group. Services use a **capacity provider strategy** with `base` and `weight`, for example 2 tasks on Fargate then split 1:3 between Fargate and Fargate Spot. For ASG providers, **managed scaling** adjusts the ASG based on task demand (`CapacityProviderReservation` metric).

</details>

### Q129. What is the difference between the ECS task role and the task execution role? 🟡

<details>
<summary><b>Answer</b></summary>

The **execution role** is used by the ECS agent/Fargate to set up the task: pull images from ECR, write logs to CloudWatch, and fetch secrets from Secrets Manager or SSM for `secrets:` entries. The **task role** is what your **application code** assumes at runtime to call AWS APIs like S3 or DynamoDB.

Mixing them up is the classic cause of `CannotPullContainerError` (execution role) versus `AccessDenied` from the app (task role).

</details>

### Q130. How does `awsvpc` network mode work in ECS? 🟡

<details>
<summary><b>Answer</b></summary>

Each task gets its own **ENI** with a private IP in your subnet and its own security group, so tasks behave like first-class VPC citizens. It is required for Fargate. Load balancers use `ip` target type. The limit on EC2 is the number of ENIs per instance, mitigated by **ENI trunking** which increases task density.

</details>

### Q131. What deployment strategies does ECS support? 🔴

<details>
<summary><b>Answer</b></summary>

The default is **rolling update** controlled by `minimumHealthyPercent` and `maximumPercent`. With the **deployment circuit breaker** enabled, failed deployments roll back automatically. **Blue/green** uses CodeDeploy (or native ECS blue/green) with two target groups, test listener and traffic shifting (canary, linear, all-at-once). **External** controllers allow third-party tools.

```json
"deploymentConfiguration": {"minimumHealthyPercent":100,"maximumPercent":200,
 "deploymentCircuitBreaker":{"enable":true,"rollback":true}}
```

</details>

### Q132. What does the EKS control plane include and who manages it? 🟢

<details>
<summary><b>Answer</b></summary>

AWS runs and scales the Kubernetes API servers and etcd across 3 AZs in an AWS-managed account, patches them, and gives you an endpoint. You manage the data plane (nodes or Fargate), add-ons, workloads and upgrades. The control plane is billed per cluster-hour, with extra charge for versions in extended support.

```bash
aws eks update-kubeconfig --name prod --region ap-south-1
```

</details>

### Q133. What are the compute options for EKS? 🟡

<details>
<summary><b>Answer</b></summary>

- **Managed node groups**: AWS-managed ASGs with graceful upgrades.
- **Self-managed nodes**: full control, custom AMIs.
- **Fargate profiles**: one pod per microVM, no DaemonSets or privileged pods.
- **Karpenter**: open-source node provisioner that launches right-sized nodes just in time.
- **EKS Auto Mode**: AWS manages nodes, networking, storage and load balancing add-ons end to end.

</details>

### Q134. How do you grant IAM principals access to an EKS cluster? 🟡

<details>
<summary><b>Answer</b></summary>

Newer clusters use **access entries** (API authentication mode): you map an IAM principal to Kubernetes permissions via access policies like `AmazonEKSClusterAdminPolicy` or to RBAC groups. The older method is the `aws-auth` ConfigMap in `kube-system`, which is error-prone and can lock you out if corrupted.

```bash
aws eks create-access-entry --cluster-name prod --principal-arn arn:aws:iam::123456789012:role/dev
aws eks associate-access-policy --cluster-name prod --principal-arn $ARN \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=app
```

</details>

### Q135. IRSA vs EKS Pod Identity? 🔴

<details>
<summary><b>Answer</b></summary>

**IRSA** uses the cluster's OIDC provider: a service account is annotated with a role ARN, the pod gets a projected token, and the SDK calls `AssumeRoleWithWebIdentity`. Each cluster needs an OIDC provider and each role's trust policy references that cluster's issuer, which becomes painful at scale.

**Pod Identity** uses an agent DaemonSet and an `aws eks create-pod-identity-association`; roles trust `pods.eks.amazonaws.com` once, are reusable across clusters, and support session tags. New workloads should use Pod Identity unless on Fargate or outside EKS.

</details>

### Q136. How does the Amazon VPC CNI assign pod IPs? 🔴

<details>
<summary><b>Answer</b></summary>

The VPC CNI gives each pod a **real VPC IP** from secondary addresses on the node's ENIs, so pods are directly routable and use security groups for pods if enabled. Max pods per node follows `ENIs × (IPs per ENI − 1) + 2`. IP exhaustion is the common failure.

Options: prefix delegation (a /28 per slot, many more pods), custom networking with a secondary CIDR, tuning `WARM_IP_TARGET`/`WARM_ENI_TARGET`, or IPv6 clusters.

</details>

### Q137. What does the AWS Load Balancer Controller do? 🟡

<details>
<summary><b>Answer</b></summary>

It watches Kubernetes objects and provisions AWS load balancers: an **Ingress** creates an ALB, and a `Service` of type `LoadBalancer` with the controller annotation creates an NLB. Using `target-type: ip` registers pod IPs directly, skipping kube-proxy hops, and enables pod readiness gates for zero-downtime rollouts.

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
```

</details>

### Q138. What features does ECR offer? 🟡

<details>
<summary><b>Answer</b></summary>

ECR is a managed private (and public) registry with IAM-based auth, **image scanning** (basic or enhanced with Inspector), **lifecycle policies** to expire old images, **tag immutability**, cross-Region and cross-account **replication**, **pull-through cache** for upstream registries like Docker Hub, and encryption with KMS.

Lifecycle policies are essential, since untagged and old images quietly accumulate storage cost.

</details>

### Q139. How do you authenticate Docker to ECR? 🟢

<details>
<summary><b>Answer</b></summary>

ECR issues a token valid for 12 hours, retrieved with the CLI and piped to Docker.

```bash
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/api:1.4.2
```

The caller needs `ecr:GetAuthorizationToken` plus push/pull permissions on the repository.

</details>

### Q140. What is a Lambda cold start? 🟢

<details>
<summary><b>Answer</b></summary>

When no warm execution environment is available, Lambda creates one: download code, start the runtime, run your initialization code outside the handler, then invoke. This adds latency from tens of milliseconds (Go, Rust, Node) to seconds (large Java or .NET functions, VPC-attached in the past). Subsequent invocations reuse the environment (warm).

Reduce it by shrinking packages, lazy-loading, using arm64, SnapStart or provisioned concurrency.

</details>

### Q141. What are the key Lambda limits? 🟡

<details>
<summary><b>Answer</b></summary>

Timeout up to **15 minutes**, memory 128 MB to **10,240 MB** (CPU scales proportionally, about 1 vCPU at 1,769 MB), `/tmp` 512 MB to 10 GB, **6 MB** synchronous payload, deployment package 50 MB zipped (250 MB unzipped) or a 10 GB container image, and account concurrency defaults to 1,000 per Region (raisable).

Know these when deciding Lambda versus Fargate for long or heavy jobs.

</details>

### Q142. Reserved vs provisioned concurrency in Lambda? 🟡

<details>
<summary><b>Answer</b></summary>

**Reserved concurrency** carves a guaranteed slice out of the account pool for a function and also **caps** it (useful to protect a downstream database); it is free. **Provisioned concurrency** keeps N execution environments initialized and ready, removing cold starts for that capacity, and is billed hourly.

```bash
aws lambda put-function-concurrency --function-name api --reserved-concurrent-executions 50
aws lambda put-provisioned-concurrency-config --function-name api --qualifier prod \
  --provisioned-concurrent-executions 20
```

</details>

### Q143. What are Lambda's invocation models? 🟡

<details>
<summary><b>Answer</b></summary>

- **Synchronous** (API Gateway, ALB, SDK `RequestResponse`): caller waits and handles errors.
- **Asynchronous** (S3, SNS, EventBridge): Lambda queues the event, retries twice on failure by default, and can send failures to a DLQ or **on-failure destination**.
- **Poll-based event source mappings** (SQS, Kinesis, DynamoDB Streams, Kafka): Lambda polls and invokes in batches, with retry behavior depending on the source.

</details>

### Q144. How does a VPC-attached Lambda reach the internet and AWS services? 🟡

<details>
<summary><b>Answer</b></summary>

Lambda creates (Hyperplane) ENIs in your subnets, shared across environments, so it can reach private resources like RDS. It then loses default internet access: public subnets do not help because Lambda ENIs have no public IP. You need a **NAT gateway** for internet, or **VPC endpoints** for AWS services like S3, DynamoDB, SQS and Secrets Manager.

Put the function in private subnets across at least two AZs.

</details>

### Q145. Explain Lambda versions, aliases and layers. 🔴

<details>
<summary><b>Answer</b></summary>

Publishing creates an immutable numbered **version** of code plus configuration; `$LATEST` is mutable. An **alias** (like `prod`) is a pointer to a version that supports **weighted routing** between two versions for canary releases. **Layers** are zip archives of shared libraries/code mounted in `/opt`, up to 5 per function, counted against the 250 MB unzipped limit.

Event sources and permissions should reference an alias so rollouts and rollbacks are a pointer change.

</details>

### Q146. API Gateway REST vs HTTP API vs Lambda function URL? 🟡

<details>
<summary><b>Answer</b></summary>

**REST API**: richest feature set (API keys, usage plans, request validation, caching, WAF, resource policies, private endpoints), higher price. **HTTP API**: cheaper (about 70% less), lower latency, JWT authorizers, fewer features. **Function URL**: a built-in HTTPS endpoint on a Lambda with IAM or no auth, no routing, throttling or WAF directly.

Default to HTTP API; choose REST when you need usage plans, caching, WAF or private APIs; use function URLs for simple webhooks.

</details>

### Q147. How does Lambda process SQS messages and handle partial failures? 🔴

<details>
<summary><b>Answer</b></summary>

The event source mapping polls the queue with long polling and invokes your function with a batch (up to 10, or 10,000 with a batching window). If the function throws, the **whole batch** returns to the queue after the visibility timeout, causing reprocessing of successes. Enable `ReportBatchItemFailures` and return the failed IDs.

```json
{"batchItemFailures":[{"itemIdentifier":"msg-id-3"}]}
```

Set visibility timeout to at least 6x the function timeout, configure a DLQ with `maxReceiveCount`, and use `MaximumConcurrency` on the mapping to protect downstreams.

</details>

### Q148. What is Lambda SnapStart? 🔴

<details>
<summary><b>Answer</b></summary>

SnapStart takes a Firecracker microVM snapshot of the initialized execution environment when you publish a version, then resumes new environments from the cached snapshot, cutting cold starts of Java functions (up to about 10x) and also available for Python and .NET. It works on published versions and aliases, not `$LATEST`.

Gotchas: uniqueness of random values and network connections created during init must be re-established; use runtime hooks (CRaC for Java).

</details>

### Q149. What are Fargate's constraints? 🟢

<details>
<summary><b>Answer</b></summary>

Fixed CPU/memory combinations per task (up to 16 vCPU and 120 GB), 20 GiB ephemeral storage by default (configurable to 200 GiB), no privileged containers or host access, no GPUs on ECS Fargate, and EFS (not EBS persistence) for durable shared storage. You pay per vCPU and GB per second with a 1-minute minimum.

Pod startup is slower than a warm EC2 node because an image pull happens on every new task, so keep images small.

</details>

### Q150. Step Functions Standard vs Express workflows? 🟡

<details>
<summary><b>Answer</b></summary>

**Standard**: up to 1 year duration, exactly-once execution semantics, full execution history, priced per state transition; for long, auditable business workflows. **Express**: up to 5 minutes, at-least-once (async) or at-most-once (sync), very high throughput (100,000+/s), priced per request and duration with logs in CloudWatch; for high-volume event processing.

States are defined in Amazon States Language (JSON), with retries, catch and parallel/map states built in.

</details>

## DNS, CDN and Messaging

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. What are the Route 53 record types and why use Alias instead of CNAME? 🟢

<details>
<summary><b>Answer</b></summary>

Standard types: A, AAAA, CNAME, MX, TXT, NS, SOA, SRV, CAA, plus Route 53 **Alias**. A CNAME cannot exist at the zone apex (`example.com`), but an Alias A record can, and it points to AWS resources (CloudFront, ALB, API Gateway, S3 website, another record) with **no query charge** and automatic tracking of IP changes.

Alias targets can also evaluate target health for failover routing.

</details>

### Q152. Describe Route 53 routing policies. 🟡

<details>
<summary><b>Answer</b></summary>

- **Simple**: one or more values, no health awareness.
- **Weighted**: split traffic by weights (canary, migration).
- **Latency**: route to the Region with the lowest latency.
- **Failover**: active-passive using health checks.
- **Geolocation**: by user country/continent (compliance, localization).
- **Geoproximity**: by distance with bias, via Traffic Flow.
- **IP-based**: by client CIDR.
- **Multivalue answer**: up to 8 healthy records.

</details>

### Q153. How do Route 53 health checks and DNS failover work? 🔴

<details>
<summary><b>Answer</b></summary>

Health checks probe an endpoint (HTTP/HTTPS/TCP) from multiple global checkers at 30 s (or 10 s fast) intervals, mark it unhealthy after consecutive failures (default 3), or aggregate other checks or a CloudWatch alarm (the only way to check private resources). Failover records answer with the secondary when the primary is unhealthy.

Total failover time is detection time plus DNS TTL, so use low TTLs (30–60 s) on failover records.

</details>

### Q154. What is a Route 53 private hosted zone? 🟡

<details>
<summary><b>Answer</b></summary>

A private hosted zone resolves names only inside associated VPCs (same or other accounts, after authorizing the association). It enables internal names like `db.prod.internal` and **split-horizon DNS**, where the same name resolves differently inside and outside. The VPC needs `enableDnsSupport` and `enableDnsHostnames`.

```bash
aws route53 associate-vpc-with-hosted-zone --hosted-zone-id Z123 --vpc VPCRegion=ap-south-1,VPCId=vpc-0abc
```

</details>

### Q155. How does DNS TTL relate to failover and migrations? 🟢

<details>
<summary><b>Answer</b></summary>

TTL tells resolvers and clients how long to cache a record. Long TTLs reduce query cost and latency but delay propagation of changes; short TTLs make cutovers quick. Lower the TTL (to 60 s) **before** a planned migration, wait out the old TTL, switch, then raise it again.

Many clients (JVMs, some proxies) cache DNS beyond TTL, so test real behaviour.

</details>

### Q156. What are Route 53 Resolver endpoints and rules? 🔴

<details>
<summary><b>Answer</b></summary>

For hybrid DNS: an **inbound endpoint** gives on-prem resolvers ENIs in your VPC to forward queries about AWS private zones to. An **outbound endpoint** plus **forwarding rules** sends queries for specific domains (for example `corp.example.com`) from VPCs to on-prem DNS servers. Rules can be shared across accounts with RAM.

Each endpoint should have ENIs in at least two AZs. Resolver DNS Firewall adds domain allow/deny lists for exfiltration control.

</details>

### Q157. How does CloudFront work: distributions, origins and behaviors? 🟡

<details>
<summary><b>Answer</b></summary>

A **distribution** has one or more **origins** (S3, ALB, API Gateway, custom HTTP) and **behaviors**, path-pattern rules that choose the origin, allowed methods, cache policy, origin request policy, viewer protocol policy and function associations. The **cache key** (URL plus whichever headers, cookies and query strings you include) decides cache hits, so include only what the origin actually varies on.

TLS certificates for custom domains must be issued in **us-east-1** in ACM.

</details>

### Q158. What is Origin Access Control (OAC)? 🟡

<details>
<summary><b>Answer</b></summary>

OAC lets CloudFront sign requests to a private S3 bucket (SigV4) so objects are not public. The bucket policy allows the `cloudfront.amazonaws.com` service principal conditioned on `AWS:SourceArn` equal to your distribution ARN. It replaces the older Origin Access Identity (OAI) and supports SSE-KMS and all Regions.

```json
{"Effect":"Allow","Principal":{"Service":"cloudfront.amazonaws.com"},"Action":"s3:GetObject",
 "Resource":"arn:aws:s3:::site/*",
 "Condition":{"StringEquals":{"AWS:SourceArn":"arn:aws:cloudfront::123456789012:distribution/E123"}}}
```

</details>

### Q159. Cache invalidation vs versioned object names? 🟢

<details>
<summary><b>Answer</b></summary>

An **invalidation** removes objects from edge caches by path (the first 1,000 paths per month are free, then charged), takes seconds to minutes, and `/*` is allowed. **Versioned names** (`app.3f9a1c.js`) make each release a new URL, so no invalidation is needed and old caches are harmless.

Best practice: long TTLs with hashed asset names, and a short TTL or invalidation only for `index.html`.

```bash
aws cloudfront create-invalidation --distribution-id E123 --paths "/index.html"
```

</details>

### Q160. CloudFront Functions vs Lambda@Edge? 🔴

<details>
<summary><b>Answer</b></summary>

**CloudFront Functions**: JavaScript, run at the edge location itself, sub-millisecond, viewer request/response events only, no network or file access; for header manipulation, redirects, URL rewrites, simple auth. **Lambda@Edge**: Node.js/Python, runs in regional edge caches, supports all four events (viewer and origin request/response), network access and longer execution; created in us-east-1 and replicated.

Use Functions by default; Lambda@Edge when you need to call out or manipulate bodies.

</details>

### Q161. Signed URLs vs signed cookies in CloudFront? 🟡

<details>
<summary><b>Answer</b></summary>

Both restrict viewer access using a **trusted key group** (public keys uploaded to CloudFront, private key held by your app). **Signed URLs** protect a single file and work for clients that do not support cookies. **Signed cookies** protect multiple files (such as an HLS stream with many segments) without changing URLs.

The policy can include expiry, IP range and path wildcards.

</details>

### Q162. Global Accelerator vs CloudFront? 🟡

<details>
<summary><b>Answer</b></summary>

CloudFront **caches** HTTP(S) content and runs edge code. Global Accelerator provides two static anycast IPv4 addresses (and dual-stack) that proxy any TCP/UDP traffic over the AWS backbone to endpoints in one or more Regions, with health-based failover in seconds and traffic dials.

Use Global Accelerator for gaming, VoIP, static IP allowlisting and multi-Region active-active without DNS caching issues.

</details>

### Q163. SQS Standard vs FIFO? 🟢

<details>
<summary><b>Answer</b></summary>

**Standard**: nearly unlimited throughput, **at-least-once** delivery, best-effort ordering. **FIFO**: strict ordering within a message group, **exactly-once processing** via deduplication (5-minute window), queue name must end in `.fifo`, and throughput up to 300 messages/s per API action (3,000 with batching), or far higher in high-throughput mode.

Default to Standard and make consumers idempotent; use FIFO when order truly matters.

</details>

### Q164. What is the SQS visibility timeout? 🟢

<details>
<summary><b>Answer</b></summary>

When a consumer receives a message, SQS hides it for the visibility timeout (default 30 s, max 12 hours). If the consumer deletes it in time the message is gone; otherwise it reappears for another consumer. Set it above your worst-case processing time, and extend it with `ChangeMessageVisibility` for long jobs.

Too short causes duplicate processing; too long delays retry after a crash.

</details>

### Q165. How do dead-letter queues and redrive work in SQS? 🟡

<details>
<summary><b>Answer</b></summary>

A redrive policy on the source queue specifies a DLQ and `maxReceiveCount`; once a message has been received that many times without deletion, SQS moves it to the DLQ. A DLQ must be of the same type (standard or FIFO). Retention does not reset when moved, so set DLQ retention to 14 days.

After fixing the bug, use **DLQ redrive** (console or `aws sqs start-message-move-task`) to send messages back to the source.

</details>

### Q166. What is SQS long polling? 🟡

<details>
<summary><b>Answer</b></summary>

With long polling (`WaitTimeSeconds` 1–20), `ReceiveMessage` waits until a message arrives instead of returning immediately empty. It cuts empty responses, API cost and CPU, and reduces false-empty responses from sampling a subset of servers in short polling.

```bash
aws sqs receive-message --queue-url $Q --wait-time-seconds 20 --max-number-of-messages 10
```

It can also be set queue-wide with `ReceiveMessageWaitTimeSeconds`.

</details>

### Q167. What is SNS and how does fan-out with SQS work? 🟡

<details>
<summary><b>Answer</b></summary>

SNS is a pub/sub service: publishers send to a **topic**, which pushes to many subscribers (SQS, Lambda, HTTP/S, email, SMS, Firehose). The fan-out pattern subscribes several SQS queues to one topic so each downstream consumer gets its own durable copy and processes at its own pace.

The queue policy must allow `sns.amazonaws.com` with `aws:SourceArn` of the topic. Use raw message delivery to skip the SNS JSON envelope.

</details>

### Q168. How does SNS message filtering work? 🟡

<details>
<summary><b>Answer</b></summary>

A subscription **filter policy** on message attributes (or on the message body, with `FilterPolicyScope: MessageBody`) delivers only matching messages to that subscriber, so one topic serves many consumers without each filtering in code.

```json
{"eventType":["order_created"],"amount":[{"numeric":[">=",100]}]}
```

Messages that match no subscription are simply dropped for those subscribers.

</details>

### Q169. How does EventBridge route events? 🔴

<details>
<summary><b>Answer</b></summary>

Events arrive on an **event bus** (default, custom or partner). **Rules** with event patterns match on fields of the JSON event and route to up to 5 targets (Lambda, SQS, Step Functions, API destinations, other buses, Kinesis, ECS tasks and more), optionally transforming input. AWS services emit events to the default bus automatically.

```json
{"source":["aws.ec2"],"detail-type":["EC2 Instance State-change Notification"],"detail":{"state":["terminated"]}}
```

</details>

### Q170. EventBridge vs SNS vs SQS: when to use which? 🟡

<details>
<summary><b>Answer</b></summary>

**SQS** is a queue: pull-based buffering with retry and a DLQ, one consumer group per queue. **SNS** is high-throughput push fan-out with simple attribute filtering. **EventBridge** is a content-based event router with rich pattern matching, schema registry, SaaS and AWS-service sources, archive/replay and cross-account buses, with lower throughput than SNS.

Common combo: EventBridge to SQS to Lambda for decoupled, buffered processing.

</details>

### Q171. What are EventBridge Scheduler and Pipes? 🔴

<details>
<summary><b>Answer</b></summary>

**Scheduler** is a serverless scheduler for one-time and recurring (cron/rate) invocations of over 270 services, with time zones, flexible windows, retries and millions of schedules, replacing most "EventBridge rule on a schedule" use. **Pipes** connect a source (SQS, Kinesis, DynamoDB Streams, Kafka) to a target with optional filtering, enrichment (Lambda, API) and transformation, with no glue code.

</details>

### Q172. How do FIFO message group IDs and deduplication work? 🔴

<details>
<summary><b>Answer</b></summary>

`MessageGroupId` defines the ordering scope: messages in one group are delivered in order to one consumer at a time, while different groups are processed in parallel. So a high-cardinality group ID (for example order ID) gives both ordering per entity and parallelism. `MessageDeduplicationId` (or content-based dedup using a SHA-256 of the body) drops duplicates sent within 5 minutes.

A single hot group ID serializes everything and becomes the throughput bottleneck, and a failed message blocks later messages in that group.

</details>

### Q173. Kinesis Data Streams vs SQS? 🟡

<details>
<summary><b>Answer</b></summary>

Kinesis is an ordered **log** partitioned in shards: multiple consumers can read the same data independently, replay within retention (24 hours to 365 days), and ordering holds per shard key. SQS is a **queue**: messages are consumed and deleted, no replay, one logical consumer per message.

Use Kinesis for streaming analytics, many consumers and replay; SQS for work distribution and decoupling.

</details>

### Q174. Why must SQS consumers be idempotent? 🟢

<details>
<summary><b>Answer</b></summary>

Standard SQS guarantees **at-least-once** delivery, so a message can be delivered twice (retries, visibility timeouts, redrive). A consumer must produce the same end state whether it processes a message once or several times, for example using a unique business key with a conditional write, or storing processed message IDs in DynamoDB with a TTL.

</details>

### Q175. What improves CloudFront cache hit ratio? 🔴

<details>
<summary><b>Answer</b></summary>

Normalize the cache key: forward only necessary query strings, headers and cookies (use cache policies, not legacy forwarding), lowercase and sort query strings via a Function, enable compression, set appropriate `Cache-Control`/TTLs at the origin, and enable **Origin Shield** to add a centralized cache layer in front of the origin and collapse requests.

Check the `CacheHitRate` metric and standard logs, and look at the `x-cache` response header (`Hit from cloudfront` or `Miss from cloudfront`).

</details>

## Observability, Security Services and IaC

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. How are CloudWatch metrics organized? 🟢

<details>
<summary><b>Answer</b></summary>

Metrics belong to a **namespace** (`AWS/EC2`), are identified by a name plus **dimensions** (`InstanceId=i-0abc`), and are stored as time series with periods of 1 minute (standard) or down to 1 second (high resolution custom metrics). Basic EC2 monitoring is 5 minutes; detailed is 1 minute. Metrics are regional and kept up to 15 months, rolled up over time.

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc --start-time 2026-11-01T00:00:00Z \
  --end-time 2026-11-01T01:00:00Z --period 300 --statistics Average
```

</details>

### Q177. How do CloudWatch Logs groups, streams and Logs Insights work? 🟢

<details>
<summary><b>Answer</b></summary>

A **log group** holds streams (one per source such as an instance or container) and defines retention and encryption. Retention defaults to **never expire**, so set it. **Logs Insights** is a query language over one or more groups, and **metric filters** turn log patterns into metrics.

```
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() by bin(5m)
```

</details>

### Q178. Explain CloudWatch alarm states and missing data handling. 🔴

<details>
<summary><b>Answer</b></summary>

States are `OK`, `ALARM` and `INSUFFICIENT_DATA`. An alarm evaluates a metric over `EvaluationPeriods` and trips on `DatapointsToAlarm` out of M (for example 3 out of 5). **TreatMissingData** can be `missing`, `notBreaching`, `breaching` or `ignore`; for request-count style metrics that vanish at zero traffic, `notBreaching` avoids false alarms, while for heartbeats `breaching` is correct.

Actions: SNS, Auto Scaling, EC2 actions, Systems Manager OpsItems.

</details>

### Q179. What is the CloudWatch Embedded Metric Format? 🔴

<details>
<summary><b>Answer</b></summary>

EMF lets you emit high-cardinality custom metrics by writing a structured JSON log line with an `_aws` metadata block; CloudWatch extracts metrics asynchronously from the log, avoiding the `PutMetricData` API call and keeping the raw detail for Logs Insights queries. It is how Lambda and containers publish metrics cheaply.

```json
{"_aws":{"Timestamp":1730000000000,"CloudWatchMetrics":[{"Namespace":"App","Dimensions":[["Service"]],
 "Metrics":[{"Name":"Latency","Unit":"Milliseconds"}]}]},"Service":"api","Latency":87}
```

</details>

### Q180. What does EC2 not report to CloudWatch by default? 🟡

<details>
<summary><b>Answer</b></summary>

The hypervisor only sees CPU, network, disk I/O (instance store) and status checks. **Memory, swap, disk space usage and process metrics are not reported**; you need the **CloudWatch agent** (config via SSM Parameter Store, `amazon-cloudwatch-agent-ctl`) which also ships logs. The instance role needs `CloudWatchAgentServerPolicy`.

</details>

### Q181. What does CloudTrail record? 🟢

<details>
<summary><b>Answer</b></summary>

CloudTrail records AWS API calls: who, what, when, from where, and the response. **Management events** are on by default and viewable in Event history for 90 days. **Data events** (S3 object-level, Lambda invokes, DynamoDB item operations) and **Insights events** must be enabled and cost extra. A **trail** delivers logs to S3 (and optionally CloudWatch Logs) indefinitely, with an Organization trail covering all accounts.

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket
```

Enable **log file validation** to detect tampering.

</details>

### Q182. What is AWS Config and how do rules work? 🟡

<details>
<summary><b>Answer</b></summary>

Config continuously records resource configuration as **configuration items**, keeping history and relationships. **Config rules** (AWS managed, or custom with Lambda or Guard) evaluate resources for compliance on change or periodically, and can auto-remediate with SSM Automation documents. **Conformance packs** deploy sets of rules, and **aggregators** combine data across accounts and Regions.

Example managed rules: `s3-bucket-public-read-prohibited`, `restricted-ssh`, `encrypted-volumes`.

</details>

### Q183. CloudWatch vs CloudTrail vs Config: what is the difference? 🟢

<details>
<summary><b>Answer</b></summary>

**CloudWatch** answers "how is it performing?" (metrics, logs, alarms). **CloudTrail** answers "who did what?" (API audit). **Config** answers "what does it look like and was it compliant over time?" (resource state and history).

Example: an SG opened to the world. Config shows the change and flags non-compliance, CloudTrail shows who made the call, CloudWatch can alarm on the event via EventBridge.

</details>

### Q184. How does KMS envelope encryption work? 🟡

<details>
<summary><b>Answer</b></summary>

KMS keys never leave KMS in plaintext and `Encrypt` directly handles at most 4 KB. For real data, you call `GenerateDataKey`, receive a plaintext **data key** and the same key encrypted under the KMS key, encrypt data locally with the plaintext key, discard it, and store the encrypted data key alongside the ciphertext. To decrypt, call `Decrypt` on the encrypted data key.

S3, EBS and RDS do this transparently. Key types: AWS owned, AWS managed (`aws/s3`) and customer managed keys.

</details>

### Q185. How do KMS key policies differ from IAM policies? 🔴

<details>
<summary><b>Answer</b></summary>

Every KMS key has a **key policy**, the primary access control: if it does not allow the principal (or delegate to IAM by granting the account root `kms:*`), IAM policies alone cannot grant access. The default policy delegates to IAM. **Grants** are programmatic, revocable, temporary permissions often used by AWS services (EBS attaching an encrypted volume).

Cross-account use needs both the key policy to allow the external principal and that account's IAM policy to allow `kms:Decrypt` etc. Deleting a key has a 7 to 30 day waiting period.

</details>

### Q186. How does KMS key rotation work? 🟡

<details>
<summary><b>Answer</b></summary>

For symmetric customer managed keys, **automatic rotation** (annual by default, configurable 90 to 2,560 days) creates new backing key material while the key ID, ARN and aliases stay the same. Old material is retained to decrypt existing ciphertext, so no re-encryption is needed. AWS managed keys rotate yearly automatically; asymmetric and imported keys do not auto-rotate.

```bash
aws kms enable-key-rotation --key-id alias/app-data --rotation-period-in-days 365
```

</details>

### Q187. Secrets Manager vs SSM Parameter Store? 🟢

<details>
<summary><b>Answer</b></summary>

**Secrets Manager** is built for secrets: native automatic rotation (RDS and others, via Lambda), cross-Region replication, resource policies, versioning stages, priced per secret per month plus API calls. **Parameter Store** holds configuration and secrets (`SecureString` with KMS); standard parameters are free (4 KB), advanced up to 8 KB with policies; no built-in rotation.

We use Parameter Store for config and low-risk values and Secrets Manager for database credentials and anything needing rotation.

</details>

### Q188. How does Secrets Manager rotation work? 🔴

<details>
<summary><b>Answer</b></summary>

A rotation Lambda runs four steps driven by version stages: `createSecret` (new value as `AWSPENDING`), `setSecret` (apply to the database), `testSecret` (verify), `finishSecret` (move `AWSCURRENT` to the new version). Clients fetching `AWSCURRENT` get the new password. AWS provides templates for RDS, Redshift and DocumentDB, including a **alternating users** strategy for zero-downtime.

```bash
aws secretsmanager rotate-secret --secret-id prod/db --rotation-lambda-arn $FN --rotation-rules AutomaticallyAfterDays=30
```

</details>

### Q189. What can AWS Systems Manager do for EC2 fleets? 🔴

<details>
<summary><b>Answer</b></summary>

With the SSM agent and an instance role (`AmazonSSMManagedInstanceCore`): **Session Manager** (shell without SSH), **Run Command** (execute scripts across tags), **Patch Manager** (patch baselines and maintenance windows), **State Manager** (desired configuration), **Automation** (runbooks), **Inventory** and **Parameter Store**.

```bash
aws ssm send-command --document-name AWS-RunShellScript \
  --targets Key=tag:Env,Values=prod --parameters 'commands=["uptime"]'
```

</details>

### Q190. What are CloudFormation stacks, change sets and drift detection? 🟢

<details>
<summary><b>Answer</b></summary>

A **stack** is the deployed instance of a template; resources are managed as a unit. A **change set** previews what an update will add, modify, replace or delete before executing it. **Drift detection** compares live resource properties with the template and reports out-of-band changes.

```bash
aws cloudformation deploy --template-file app.yaml --stack-name app --capabilities CAPABILITY_NAMED_IAM
aws cloudformation detect-stack-drift --stack-name app
```

</details>

### Q191. What does `UPDATE_ROLLBACK_FAILED` mean and how do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

An update failed and the automatic rollback also failed, usually because a resource was changed or deleted manually, or IAM permissions disappeared, leaving the stack stuck and blocking updates. Fix the underlying cause, then continue the rollback, skipping unfixable resources if needed.

```bash
aws cloudformation continue-update-rollback --stack-name app \
  --resources-to-skip LogicalId1 LogicalId2
```

Prevent it with stack policies, SCPs against manual changes, and by avoiding console edits of managed resources.

</details>

### Q192. What do `Ref`, `Fn::GetAtt` and `Fn::Sub` do in CloudFormation? 🟢

<details>
<summary><b>Answer</b></summary>

`!Ref` returns a resource's primary identifier (instance ID, bucket name) or a parameter value. `!GetAtt Resource.Attribute` returns a specific attribute (`MyBucket.Arn`, `Alb.DNSName`). `!Sub` substitutes `${Var}` in strings, including pseudo parameters like `${AWS::Region}` and `${AWS::AccountId}`.

```yaml
Resource: !Sub "arn:${AWS::Partition}:s3:::${Bucket}/*"
```

</details>

### Q193. What are DeletionPolicy and UpdateReplacePolicy? 🟡

<details>
<summary><b>Answer</b></summary>

`DeletionPolicy` (`Delete`, `Retain`, `Snapshot`, `RetainExceptOnCreate`) controls what happens to a resource when its logical ID is removed or the stack is deleted. `UpdateReplacePolicy` does the same when an update **replaces** the resource. Stateful resources (RDS, EBS, S3) should use `Snapshot` or `Retain`.

```yaml
Database:
  Type: AWS::RDS::DBInstance
  DeletionPolicy: Snapshot
  UpdateReplacePolicy: Snapshot
```

</details>

### Q194. How does the AWS CDK work? 🟡

<details>
<summary><b>Answer</b></summary>

CDK lets you define infrastructure in TypeScript, Python, Java, Go or C# using **constructs**: L1 (raw CloudFormation resources, `CfnBucket`), L2 (opinionated, sensible defaults and helper methods such as `bucket.grantRead(role)`), and L3 patterns (multi-resource). `cdk synth` generates CloudFormation, and `cdk deploy` deploys it. Each account/Region needs a one-time **bootstrap** (the `CDKToolkit` stack: asset bucket, roles).

```bash
cdk bootstrap aws://123456789012/ap-south-1 && cdk diff && cdk deploy
```

</details>

### Q195. CloudFormation vs Terraform vs CDK: how do they compare? 🔴

<details>
<summary><b>Answer</b></summary>

CloudFormation is AWS-native, no state file to manage (AWS holds state), automatic rollback, day-one support for many new features, but AWS-only and verbose. Terraform is multi-cloud with a large ecosystem, explicit state and a plan/apply workflow, but you manage the state backend and locking. CDK is a programming layer that **compiles to CloudFormation**.

Choice usually follows team skills and whether you manage non-AWS resources too.

</details>

### Q196. What are the six pillars of the AWS Well-Architected Framework? 🟢

<details>
<summary><b>Answer</b></summary>

**Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization and Sustainability.** Each has design principles and best-practice questions; the **Well-Architected Tool** records a workload review and highlights risks, and lenses add domain guidance (serverless, SaaS, ML, financial services).

</details>

### Q197. What do GuardDuty, Inspector, Macie and Security Hub each do? 🟡

<details>
<summary><b>Answer</b></summary>

**GuardDuty**: threat detection from CloudTrail, VPC Flow Logs, DNS logs, EKS and S3 data events (anomalous API use, crypto-mining, exfiltration). **Inspector**: vulnerability scanning of EC2, ECR images and Lambda. **Macie**: finds sensitive data (PII) in S3. **Security Hub**: aggregates findings from these and others, runs compliance checks (CIS, AWS Foundational Security Best Practices) and normalizes to ASFF/OCSF.

</details>

### Q198. What do Trusted Advisor and Compute Optimizer provide? 🟢

<details>
<summary><b>Answer</b></summary>

**Trusted Advisor** runs checks across cost, performance, security, fault tolerance, service limits and operational excellence (the full set needs Business or Enterprise support). **Compute Optimizer** uses CloudWatch metrics and ML to recommend right-sized EC2, ASG, EBS, Lambda and ECS-on-Fargate configurations, including Graviton options.

</details>

### Q199. Nested stacks vs StackSets? 🔴

<details>
<summary><b>Answer</b></summary>

**Nested stacks** (`AWS::CloudFormation::Stack`) compose one deployment from reusable templates in the **same account and Region**, with outputs passed via parameters, and are updated as a unit. **StackSets** deploy one template to **many accounts and Regions**, with service-managed permissions through Organizations and automatic deployment to new accounts in an OU.

Use nested stacks to modularize a big application, and StackSets for baselines like IAM roles, Config and logging.

</details>

### Q200. What are CloudFormation custom resources, `cfn-signal` and `CreationPolicy`? 🔴

<details>
<summary><b>Answer</b></summary>

A **custom resource** (`AWS::CloudFormation::CustomResource` or `Custom::Name`) invokes a Lambda or SNS endpoint during create, update and delete so you can do anything the API lacks; it **must** send a response to the pre-signed S3 URL or the stack hangs for an hour. A **CreationPolicy** with `ResourceSignal` makes CloudFormation wait for a `cfn-signal` from the instance (for example after user data finishes) before marking it `CREATE_COMPLETE`, failing on timeout.

```bash
/opt/aws/bin/cfn-signal -e $? --stack app --resource WebAsg --region ap-south-1
```

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
