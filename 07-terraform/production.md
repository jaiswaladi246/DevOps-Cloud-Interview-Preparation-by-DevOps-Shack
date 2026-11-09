# Terraform — Production & Best Practices

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

> Real-world Terraform: state, structure, CI/CD, security, testing, scale and trade-offs · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Terraform & IaC](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [State and Backends in Production](#state-and-backends-in-production) — Q1–Q30
- [Repository Structure, Environments and Module Strategy](#repository-structure-environments-and-module-strategy) — Q31–Q60
- [CI/CD and Automation](#cicd-and-automation) — Q61–Q85
- [Security, Secrets and Access](#security-secrets-and-access) — Q86–Q110
- [Testing and Policy as Code](#testing-and-policy-as-code) — Q111–Q135
- [Refactoring, Migration and Drift](#refactoring-migration-and-drift) — Q136–Q155
- [HCP Terraform, OpenTofu, Scale and Cost](#hcp-terraform-opentofu-scale-and-cost) — Q156–Q180
- [Rapid-Fire Commands](#rapid-fire-commands) — Q181–Q200

---

## State and Backends in Production

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Why should Terraform state never be committed to Git? 🟢

<details>
<summary><b>Answer</b></summary>

State holds secrets in plaintext, has no locking, and merge conflicts on a JSON file with a serial number are a recipe for corruption. In production we use a remote backend with locking, encryption and versioning, and `.gitignore` all `*.tfstate*` files.

</details>

### Q2. How do you bootstrap the remote backend, given the bucket must exist before Terraform can use it? 🟡

<details>
<summary><b>Answer</b></summary>

Create the state bucket once with a small bootstrap stack that uses local state (or the CLI or CloudFormation), then switch that stack's backend to the bucket and run `terraform init -migrate-state`. After that the bootstrap stack manages its own bucket. Keep this stack tiny, `prevent_destroy = true`, and documented, since it is the foundation everything else rests on.

</details>

### Q3. What does a hardened S3 state bucket look like? 🟡

<details>
<summary><b>Answer</b></summary>

Versioning on, default SSE-KMS encryption, all four public-access-block settings enabled, a bucket policy that denies non-TLS requests and restricts principals to the CI roles, access logging, lifecycle rules to expire noncurrent versions after N days, and optionally cross-region replication.
```hcl
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration { status = "Enabled" }
}
```

</details>

### Q4. What is the minimum IAM permission set for the S3 backend? 🟡

<details>
<summary><b>Answer</b></summary>

`s3:ListBucket` on the bucket, `s3:GetObject` and `s3:PutObject` on the state key, and with `use_lockfile` also `s3:DeleteObject` on the `<key>.tflock` object. With the legacy lock table add `dynamodb:GetItem`, `PutItem` and `DeleteItem` on it. If the bucket uses a KMS key, grant `kms:Decrypt`, `kms:GenerateDataKey` on it too. Scope the policy per prefix so team A cannot read team B's state.

</details>

### Q5. How do you move from DynamoDB locking to S3 native locking in a live fleet? 🟡

<details>
<summary><b>Answer</b></summary>

1. Upgrade every CI runner and laptop to Terraform 1.10 or newer (1.11+ preferred, where it is stable).
2. Add `use_lockfile = true` while keeping `dynamodb_table`, so both mechanisms lock during the transition.
3. Run `terraform init -reconfigure` per stack and confirm a plan acquires the lock.
4. Grant `DeleteObject` on the `.tflock` key.
5. Remove `dynamodb_table` and later the table itself.

</details>

### Q6. State was deleted or corrupted. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

With S3 versioning, list object versions of the key and restore the last good one (copy that version over the current object, or delete the delete marker). Then `terraform plan` should show no diff. Without versioning you are left with `terraform import` for every resource, which is why versioning, replication and a restore drill are non-negotiable. Run `terraform state pull` backups nightly in sensitive stacks.
```bash
aws s3api list-object-versions --bucket acme-tfstate --prefix prod/network/
aws s3api copy-object --bucket acme-tfstate --key prod/network/terraform.tfstate \
  --copy-source "acme-tfstate/prod/network/terraform.tfstate?versionId=<id>"
```

</details>

### Q7. How do you split a monolithic state and what drives blast-radius decisions? 🔴

<details>
<summary><b>Answer</b></summary>

Blast radius is "what can one bad apply or lost state damage". Split along change frequency and ownership: foundation (accounts, IAM), network, data stores, compute, app. To split, move resources with `terraform state mv -state-out`, or declare `removed` blocks in the old stack and `import` blocks in the new one, and verify both plans are empty. Do it during a change freeze and keep the old state backed up.

</details>

### Q8. How do you decide where state boundaries go? 🟡

<details>
<summary><b>Answer</b></summary>

Layer by lifecycle and risk: things that change rarely and are catastrophic if broken (VPC, KMS, IAM) in one state, volatile app layers in others; separate by environment and account always; separate by team when ownership differs. Too few states give slow plans and large blast radius; too many give an orchestration headache and cross-state dependencies.

</details>

### Q9. One central state bucket or one per account? 🟡

<details>
<summary><b>Answer</b></summary>

We prefer one bucket per environment account (or per account group) so prod state is only readable from prod-privileged roles and a compromise of dev cannot read prod secrets. A single central bucket with prefix-scoped IAM is simpler to run but concentrates risk. Cross-account is done with `assume_role` in the backend block.

</details>

### Q10. What naming convention do you use for state keys? 🟢

<details>
<summary><b>Answer</b></summary>

Predictable and path-like: `<env>/<region>/<stack>/terraform.tfstate`, for example `prod/ap-south-1/network/terraform.tfstate`. It makes IAM prefixes, lifecycle rules and audits easy, and avoids two stacks pointing to the same key by accident.

</details>

### Q11. How do stacks share data: remote state, SSM or data sources? 🟡

<details>
<summary><b>Answer</b></summary>

`terraform_remote_state` is simple but grants read access to the whole state and couples consumers to output names. Publishing selected values to SSM Parameter Store (or Secrets Manager) gives a controlled, versionable interface that non-Terraform consumers can read too. Data sources that look up by tag or name decouple stacks entirely but need consistent tagging. We use SSM for contracts between teams.

</details>

### Q12. Applies keep failing with `Error acquiring the state lock` in CI. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

First read the lock info (who, operation, created time). If a live run holds it, serialize runs: use a CI concurrency group per stack and `-lock-timeout=10m` so queued runs wait. If it is stale (cancelled job, runner died) confirm no process runs, then `terraform force-unlock <ID>`. Fix the cause: avoid cancelling apply jobs mid-run, and make pipeline cancellation graceful.

</details>

### Q13. How do you protect state against region loss or accidental deletion? 🔴

<details>
<summary><b>Answer</b></summary>

Versioning plus cross-region replication of the bucket, MFA delete or an Object Lock retention for the most critical states, deny-delete bucket policies, and periodic restore tests. Keep the KMS key multi-region or ensure replicated data can be decrypted in the DR region. A backup nobody has restored is only a hope.

</details>

### Q14. What symptoms tell you a state is too large? 🟡

<details>
<summary><b>Answer</b></summary>

Plans taking tens of minutes (thousands of refresh API calls), API throttling errors, 10 MB+ state JSON, lock held for long periods and a diff review that nobody can read. Fix by splitting states, pruning no-longer-needed resources, using modules with `for_each` sparingly at scale and increasing `-parallelism` only if the API tolerates it.

</details>

### Q15. How do you keep secrets out of the state file? 🟡

<details>
<summary><b>Answer</b></summary>

Prefer not creating secrets in Terraform at all: let the service generate them (RDS `manage_master_user_password = true`), reference secret ARNs not values and let apps read from Secrets Manager. Where a secret value must pass through Terraform, use ephemeral resources and write-only arguments (1.10 and 1.11+). Otherwise assume the state contains it and lock the state down.

</details>

### Q16. How can you audit who changed state and when? 🟢

<details>
<summary><b>Answer</b></summary>

S3 versioning keeps every revision with timestamps, CloudTrail data events on the bucket show which principal wrote it, and the CI system's run history links each write to a commit and approver. With HCP Terraform, run and state version history is built in.

</details>

### Q17. What is partial backend configuration and why use it? 🟢

<details>
<summary><b>Answer</b></summary>

You keep only the static parts in the `backend` block and supply environment-specific values at init time.
```hcl
terraform { backend "s3" {} }
```
```bash
terraform init -backend-config=env/prod.backend.hcl
```
Where `prod.backend.hcl` has `bucket`, `key` and `region`. It lets one codebase target several accounts without editing files.

</details>

### Q18. How do you configure the S3 backend to use a role in another account? 🔴

<details>
<summary><b>Answer</b></summary>

```hcl
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-shared"
    key          = "prod/app/terraform.tfstate"
    region       = "ap-south-1"
    use_lockfile = true
    assume_role  = { role_arn = "arn:aws:iam::111111111111:role/tf-state-access" }
  }
}
```
The role needs the bucket permissions above, and the bucket policy must trust it. The backend's credentials are separate from the AWS provider's, so you can state-access with one role and deploy with another.

</details>

### Q19. How do other backends handle locking? 🟢

<details>
<summary><b>Answer</b></summary>

`azurerm` uses blob leases, `gcs` writes a lock object next to the state, `pg` uses advisory locks, GitLab-managed state uses its HTTP API and HCP Terraform locks per workspace and run queue. The commands are the same everywhere; the difference is in what to monitor and how to break a stale lock.

</details>

### Q20. A consumer stack breaks after the producer renamed an output. How do you prevent this? 🟡

<details>
<summary><b>Answer</b></summary>

Treat outputs as an API: version them, never rename without deprecating (keep both for a release), and test consumers in the pipeline. Prefer SSM-parameter contracts with a documented path, or publish a single `outputs` object for stability. Add `precondition` checks in the consumer so a missing value fails with a clear message.

</details>

### Q21. Two engineers applied at the same time from laptops with local state. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

Duplicate resources or conflicting state files are likely. Stop all applies, collect both `terraform.tfstate` files, and decide the truth by inspecting the cloud. Import resources missing from the chosen state and delete duplicates created by the other apply. Then migrate to a remote backend with locking and move apply into CI so this cannot recur.

</details>

### Q22. What does `encrypt = true` do in the S3 backend? 🟢

<details>
<summary><b>Answer</b></summary>

It enables server-side encryption for the state object. With `kms_key_id` it uses your KMS key; without it, S3's default encryption (SSE-S3). Also enforce encryption through the bucket's default encryption so a missing flag does not matter.

</details>

### Q23. What lifecycle policy do you put on the state bucket? 🟢

<details>
<summary><b>Answer</b></summary>

Keep current versions forever, expire noncurrent versions after 90 to 365 days (enough to roll back), abort incomplete multipart uploads and clean up expired delete markers. Be careful not to expire so aggressively that you cannot restore after a slow-burn corruption.
```hcl
noncurrent_version_expiration { noncurrent_days = 180 }
```

</details>

### Q24. How does Terragrunt handle backends? 🟡

<details>
<summary><b>Answer</b></summary>

Its `remote_state` block generates a `backend.tf` per stack from a root template, deriving the key from the folder path (`path_relative_to_include()`), and can create the bucket automatically. That removes copy-pasted backend blocks across hundreds of stacks. The cost is one more tool and its own upgrade cycle.

</details>

### Q25. How do you manage the state bucket with Terraform itself? 🔴

<details>
<summary><b>Answer</b></summary>

Bootstrap with local state, apply, add the `backend "s3"` block and migrate. Protect with `prevent_destroy`, deny deletion in bucket policy and keep the stack in its own, rarely-changed repo path with extra approvers. If you ever destroy it you lose everything, so we also snapshot the bucket's configuration in documentation.

</details>

### Q26. A plan suddenly wants to create everything. What happened? 🟡

<details>
<summary><b>Answer</b></summary>

Terraform is reading empty or the wrong state: the backend `key` changed, you are in the wrong workspace, `init -reconfigure` pointed to a new bucket, or state was deleted. Do not apply. Check `terraform workspace show`, `terraform state list` and the backend config, then restore the right state.

</details>

### Q27. How do you rotate the KMS key used for state encryption? 🟡

<details>
<summary><b>Answer</b></summary>

Enable automatic key rotation on the KMS key (the key ID stays the same, so Terraform needs no change). To switch to a different key, update `kms_key_id`, and re-encrypt existing objects by copying them in place (`aws s3 cp` with the new key) since S3 does not re-encrypt automatically.

</details>

### Q28. How do you prevent teams from running stacks with local backends? 🔴

<details>
<summary><b>Answer</b></summary>

CI policy: fail the pipeline if no `backend` or `cloud` block exists (grep or a Checkov/OPA rule on the config), run `terraform init` with `-backend=true` only, and remove human IAM write access to production so only the pipeline's role can apply. Add a template repo that already includes a correct backend.

</details>

### Q29. Who should be allowed to write production state? 🟢

<details>
<summary><b>Answer</b></summary>

Only the CI/CD deployment role. Humans get read access through a break-glass or audited role. This ensures every change is reviewed, planned and logged, and nobody applies from a laptop with unreviewed local code.

</details>

### Q30. How do you audit read access to state? 🟡

<details>
<summary><b>Answer</b></summary>

Enable CloudTrail data events for the state bucket (GetObject), send them to a log archive and alert on principals that are not the CI roles. In a mature setup an anomaly on reads of the `prod/iam` key pages the security team.

</details>

## Repository Structure, Environments and Module Strategy

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q31. Monorepo or multi-repo for Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

A monorepo makes cross-stack changes, shared modules and consistent tooling easy, but needs path-filtered pipelines and CODEOWNERS. Multi-repo gives clear ownership and independent versioning but invites drift between repos. We usually use a modules repo (versioned via tags) plus one live-infrastructure repo per team or domain, whichever fits the org's size.

</details>

### Q32. How do you lay out environments in a repo? 🟢

<details>
<summary><b>Answer</b></summary>

```
live/
  dev/ap-south-1/network/  main.tf  backend.hcl  terraform.tfvars
  prod/ap-south-1/network/
modules/
  vpc/  eks/  rds/
```
Each leaf directory is a stack with its own state. Environments differ in values, not in code, and modules are consumed by version. The extra files are repetitive but explicit and safe to review.

</details>

### Q33. When would you adopt Terragrunt? 🟢

<details>
<summary><b>Answer</b></summary>

When you have dozens of stacks across accounts and regions and copy-paste of backend, provider and input blocks becomes the main cost. Terragrunt provides DRY config via `include`, dependency wiring between stacks and `run-all` orchestration. For a handful of stacks, plain Terraform with a small Makefile or pipeline is simpler; many teams find Terraform's own features enough.

</details>

### Q34. Do you use tfvars files per environment or code branches? 🟡

<details>
<summary><b>Answer</b></summary>

tfvars (or Terragrunt inputs) per environment, one branch. Environment-per-branch means promotions are merges that nobody can reason about and fixes need cherry-picking. A single main branch with per-environment directories and version-pinned modules keeps the diff for each promotion visible.

</details>

### Q35. How do you promote a module change from dev to qa to prod? 🔴

<details>
<summary><b>Answer</b></summary>

Tag the module (`v2.3.0`), bump the version in dev's stack, apply, soak and run tests; then bump qa, then prod through approvals. Because each environment pins a version, you always know what is deployed where, and rollback is reverting the pin (state-compatible changes only). For breaking releases, ship `moved` blocks and document the migration.

</details>

### Q36. What is an internal module registry strategy? 🟢

<details>
<summary><b>Answer</b></summary>

Publish vetted modules (VPC, EKS, RDS, S3) in a private registry (HCP Terraform, GitLab or Artifactory) or Git tags, with semver, changelogs and example usage. A platform team owns them, application teams consume by version. Add CI to the module repo: fmt, tflint, Checkov, `terraform test`, and a release workflow that tags on merge.

</details>

### Q37. How would you structure multi-account AWS with Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

A management account for Organizations and SCPs, a log archive and audit account, shared services (networking, CI), and workload accounts per environment. Account vending can be automated (Control Tower Account Factory for Terraform, or custom modules). Each account gets a baseline stack (IAM roles, CloudTrail, Config, KMS) applied from a central pipeline that assumes a bootstrap role in the target account.

</details>

### Q38. How do you enforce naming and tagging standards? 🟢

<details>
<summary><b>Answer</b></summary>

Compute names in a shared `locals` or naming module (`"${var.org}-${var.env}-${var.service}"`), set `default_tags` at the provider, enforce mandatory tags with a Checkov or OPA policy on plan JSON, and add AWS tag policies or Config rules as a backstop. Standards enforced only by convention drift within months.

</details>

### Q39. What makes a module a "god module", and how do you avoid it? 🔴

<details>
<summary><b>Answer</b></summary>

One module that creates VPC, cluster, database, DNS and monitoring with 100 variables, where any change risks everything and nobody can reuse a piece. Avoid by composing small modules with narrow interfaces, passing IDs between them, limiting variables to real decisions, and keeping lifecycle-different things (data stores versus stateless compute) in different modules and often different states.

</details>

### Q40. How do you manage provider and module upgrades across many repos? 🟡

<details>
<summary><b>Answer</b></summary>

Pin versions with `~>`, commit the lock file and let Renovate or Dependabot open PRs per update. The PR pipeline runs `plan` against dev; a human reviews the plan for replacements. Group low-risk patch updates, handle major upgrades as planned projects with a runbook, and track versions centrally to find stragglers.

</details>

### Q41. How do you pin the Terraform version across a team? 🟢

<details>
<summary><b>Answer</b></summary>

Use `required_version = "~> 1.9"` in code and a version manager that reads `.terraform-version` or `.tool-versions` (tfenv, mise, asdf). CI installs the same version with `hashicorp/setup-terraform` and `terraform_version`. This avoids the "newer version wrote state" problem.

</details>

### Q42. How do you handle dev being cheaper than prod without forking code? 🟡

<details>
<summary><b>Answer</b></summary>

Make the differences explicit inputs: instance class, node count, multi-AZ, retention, deletion protection. Dev uses smaller defaults; prod overrides in its tfvars. Keep the topology identical (same modules, same dependencies) so what you test in dev is what you ship. Toggle optional components (NAT per AZ, WAF) by boolean variables.

</details>

### Q43. How do you manage hundreds of similar resources from a config file? 🔴

<details>
<summary><b>Answer</b></summary>

Describe them as data (YAML or JSON), decode it and drive `for_each`.
```hcl
locals {
  teams = yamldecode(file("${path.module}/teams.yaml"))
}
module "team" {
  for_each = local.teams
  source   = "./modules/team"
  name     = each.key
  config   = each.value
}
```
Non-Terraform contributors can submit PRs to the YAML. Validate with variable types in the module so bad data fails early, and watch plan time as the count grows.

</details>

### Q44. What are the trade-offs of YAML-driven configuration? 🟡

<details>
<summary><b>Answer</b></summary>

Pros: easy onboarding for non-engineers, a clear data/logic split and simple diffs. Cons: loses Terraform's type checking until it reaches module variables, makes ad-hoc `lookup`/`try` logic common and hides dependencies from `terraform validate`. Add JSON-schema validation in CI and strict `object()` types in the module.

</details>

### Q45. How do you implement feature toggles in a module? 🟢

<details>
<summary><b>Answer</b></summary>

Boolean variables feeding `count` or `for_each`, with outputs using `one()` to return null when off.
```hcl
variable "enable_waf" { type = bool  default = false }
resource "aws_wafv2_web_acl" "this" { count = var.enable_waf ? 1 : 0 /* ... */ }
output "waf_arn" { value = one(aws_wafv2_web_acl.this[*].arn) }
```
Keep toggles few and meaningful; too many produce untestable combinations.

</details>

### Q46. How should a module repo be organised? 🟢

<details>
<summary><b>Answer</b></summary>

Root module files at top level, `examples/basic` and `examples/complete` that actually run in CI, `tests/` for `.tftest.hcl` files, optional `modules/` submodules, README generated by terraform-docs and a CHANGELOG. Examples double as documentation and integration tests.

</details>

### Q47. How do you order applies across dependent stacks? 🟡

<details>
<summary><b>Answer</b></summary>

Make the dependency explicit in the pipeline: a DAG (network, then data, then compute, then app) built from Terragrunt `dependency` blocks, CI job `needs`, or HCP Terraform run triggers. Stacks read each other's outputs through remote state or SSM. A change to the network stack should trigger plans for its dependents so breakage is found early.

</details>

### Q48. How would you orchestrate applies for 80 stacks? 🔴

<details>
<summary><b>Answer</b></summary>

Use an orchestrator rather than hand-scripting: Terragrunt `run-all` with a dependency graph, Spacelift stack dependencies, HCP Terraform run triggers or a CI matrix generated from a dependency file. Plan everything on PR, apply in topological order on merge, limit concurrency and halt on first failure. Observability matters: one dashboard of last-run status per stack.

</details>

### Q49. How do you design a multi-region deployment? 🟡

<details>
<summary><b>Answer</b></summary>

One stack per region that instantiates the same modules with different provider config and inputs, with a global stack for non-regional resources (IAM, Route 53, CloudFront). Avoid one giant stack with multiple provider aliases unless it must exist (a replica relationship); separate states let you roll out region by region and contain failures.

</details>

### Q50. How do you provision per-customer or per-tenant environments? 🟡

<details>
<summary><b>Answer</b></summary>

Model each tenant as data and instantiate a tenant module with `for_each`, or one state per tenant created by a pipeline from a template. Per-tenant states limit blast radius and allow custom upgrades; one state with `for_each` is simpler but plan time grows and every tenant is touched at once. Choose based on tenant count and isolation needs.

</details>

### Q51. How do you build preview (ephemeral) environments? 🟡

<details>
<summary><b>Answer</b></summary>

A pipeline creates a workspace or state key named after the PR (`pr-123`), applies a slim variant of the stack with auto-expire tags, comments the URL on the PR and runs `destroy` on PR close, plus a nightly janitor that deletes anything older than N days. Cost controls and quotas matter or leaks pile up.

</details>

### Q52. How do you bring an existing cloud estate under Terraform? 🔴

<details>
<summary><b>Answer</b></summary>

Inventory first. Prioritise by risk and change rate rather than importing everything. Write modules for new work, import critical existing resources with `import` blocks (and `-generate-config-out` as a draft), review generated code, remove noise, and only then enforce through IAM that changes go through Terraform. Expect to find drift and orphaned resources; do not try a big bang.

</details>

### Q53. Where is the boundary between Terraform and Kubernetes tooling? 🟡

<details>
<summary><b>Answer</b></summary>

Terraform owns the cloud foundation: VPC, cluster, node groups, IAM roles for service accounts, databases. In-cluster resources (Deployments, Services, Helm releases of apps) are better handled by GitOps (Argo CD or Flux) because they change frequently and need cluster-native reconciliation. Terraform may install the GitOps controller itself, then stop.

</details>

### Q54. Should Terraform deploy application releases? 🟡

<details>
<summary><b>Answer</b></summary>

Generally no. Terraform is good at infrastructure shape; application releases change many times a day, need rollbacks, canaries and health checks that a CD tool does better. Terraform may define the ECS service or Lambda, with `ignore_changes` on the image tag while the pipeline updates it.

</details>

### Q55. How do you manage ownership with CODEOWNERS? 🟢

<details>
<summary><b>Answer</b></summary>

Map directories to teams: `/live/prod/iam/ @acme/security`, `/modules/ @acme/platform`. Combined with branch protection, required reviewers and required status checks, it ensures risky areas have the right approvers. Pair it with narrow CI roles so the protection is not only procedural.

</details>

### Q56. Why do we tag resources, and which tags are mandatory? 🟢

<details>
<summary><b>Answer</b></summary>

Tags drive cost allocation, ownership, automation and incident response. Common mandatory set: `Environment`, `Service`, `Owner` (team), `CostCenter`, `ManagedBy = "terraform"` and the repo path. Apply with provider `default_tags` and enforce with policy.

</details>

### Q57. Design a reusable module for a three-tier web service. 🔴

<details>
<summary><b>Answer</b></summary>

Inputs: VPC and subnet IDs, instance or container settings, DB class, domain and certificate ARN. Internals: ALB in public subnets, ASG or ECS service in private, RDS in isolated subnets, security groups chained by reference (alb to app to db), encryption on everything. Outputs: ALB DNS, service security group ID, DB endpoint (sensitive where needed). Add validation, sensible defaults, `moved` blocks for upgrades and examples with tests.

</details>

### Q58. Should you wrap community modules or use them directly? 🟡

<details>
<summary><b>Answer</b></summary>

Use them directly with a pinned version when the interface fits. Wrap only to enforce company policy (forced encryption, mandatory tags, logging) or to simplify a large interface. A wrapper that re-exposes all 80 variables adds maintenance with no benefit, and every upstream change then needs to be passed through.

</details>

### Q59. What do you do when a module has too many variables? 🟡

<details>
<summary><b>Answer</b></summary>

Group related options into objects (`optional()` attributes with defaults), split the module along responsibility lines, hide advanced options behind one `advanced` object, and give strong defaults so callers set only what differs. A module with 80 required inputs is a configuration file in disguise.

</details>

### Q60. How do you build cross-account networking such as Transit Gateway attachments with Terraform? 🔴

<details>
<summary><b>Answer</b></summary>

Use two provider configurations (hub and spoke accounts) passed into a module via `providers`. The spoke creates the attachment request, the hub accepts it (`aws_ec2_transit_gateway_vpc_attachment_accepter`) and adds routes. Keep RAM sharing of the TGW and cross-account IAM in the foundation stack, and apply in the right order since the hub change depends on the spoke's attachment ID.

</details>

## CI/CD and Automation

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q61. Why run Terraform from a pipeline instead of laptops? 🟢

<details>
<summary><b>Answer</b></summary>

A pipeline gives one controlled identity, consistent tool versions, reviewed code, logged plans and approvals, locking discipline and no long-lived credentials on laptops. It also makes "what is deployed" equal "what is on main".

</details>

### Q62. What stages does a good Terraform pull-request pipeline have? 🟢

<details>
<summary><b>Answer</b></summary>

On every PR: `terraform fmt -check -recursive`, `init -backend=false` plus `validate`, tflint, Checkov or Trivy, then `plan -out` with the plan summary posted as a PR comment and cost estimate from Infracost. On merge to main: apply the reviewed plan to dev automatically and to higher environments after approval. Fast checks first, expensive ones later.

</details>

### Q63. How do you make sure the apply runs exactly what was reviewed? 🟡

<details>
<summary><b>Answer</b></summary>

Save the plan in the plan job (`terraform plan -out=tfplan`), upload it as a protected artifact and apply that file in the apply job after approval. Terraform rejects a stale plan if state changed. For the plan file to be valid, apply must use the same code, same providers and the same working directory layout, so run both in one workflow with the same checkout SHA.

</details>

### Q64. How do you serialize applies to the same stack in CI? 🔴

<details>
<summary><b>Answer</b></summary>

Use a concurrency group keyed by stack so only one run per stack executes and later ones queue (do not cancel an in-progress apply).
```yaml
concurrency:
  group: tf-${{ matrix.stack }}
  cancel-in-progress: false
```
Add `-lock-timeout` as a second layer. Never enable cancel-in-progress for apply jobs: killing Terraform mid-apply leaves locks and partial state.

</details>

### Q65. Show a minimal GitHub Actions workflow for plan on PR. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
on: pull_request
permissions: { id-token: write, contents: read, pull-requests: write }
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.9.8 }
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/tf-plan
          aws-region: ap-south-1
      - run: terraform init -input=false
      - run: terraform plan -input=false -out=tfplan
```

</details>

### Q66. How does OIDC authentication from GitHub Actions to AWS work? 🟡

<details>
<summary><b>Answer</b></summary>

You create an IAM OIDC identity provider for `token.actions.githubusercontent.com` and a role whose trust policy allows `sts:AssumeRoleWithWebIdentity` with conditions on `sub` (for example `repo:acme/infra:ref:refs/heads/main` or `repo:acme/infra:environment:prod`) and `aud` (`sts.amazonaws.com`). The workflow requests a short-lived token (`id-token: write`) and exchanges it for credentials. No secrets stored in GitHub.

</details>

### Q67. What is Atlantis and how does the flow work? 🟢

<details>
<summary><b>Answer</b></summary>

Atlantis is a self-hosted server that listens to PR webhooks. Commenting `atlantis plan` runs plan and posts the output; `atlantis apply` applies after approval and locks the directory or workspace for that PR until merge. Configuration lives in `atlantis.yaml`. It is popular because the workflow is entirely in the PR, but you operate and secure the server and its cloud credentials.

</details>

### Q68. Why separate plan and apply identities? 🟡

<details>
<summary><b>Answer</b></summary>

A plan role can be read-only (plus state read and lock), so PRs from branches (potentially untrusted) cannot change infrastructure. The apply role is write-capable and only assumable from the protected main branch or an approved environment. This limits the damage of a malicious PR, since Terraform config can run arbitrary code during plan via `external` data sources or provider config.

</details>

### Q69. How do you gate production applies? 🔴

<details>
<summary><b>Answer</b></summary>

GitHub Environments with required reviewers (or GitLab protected environments), branch protection on main, CODEOWNERS for sensitive paths, and applying only the saved plan that was reviewed. For the highest-risk stacks (IAM, networking) require two approvers and a change ticket reference. Auto-apply on dev; manual on prod.

</details>

### Q70. How do you run only the stacks affected by a change in a monorepo? 🔴

<details>
<summary><b>Answer</b></summary>

Compute changed directories from `git diff --name-only origin/main`, map them to stacks (plus stacks depending on changed modules) and feed them to a job matrix. Tools: GitHub path filters, Atlantis `when_modified`, Terragrunt `--terragrunt-modules-that-include`, Spacelift or env0 stack triggers. Always include a full nightly plan so missed dependencies surface.

</details>

### Q71. How do you set up drift detection in CI? 🟡

<details>
<summary><b>Answer</b></summary>

A scheduled workflow runs `terraform plan -detailed-exitcode -lock=false` for each stack. Exit 2 opens a ticket or posts to Slack with the plan summary; exit 1 pages the owner. `-lock=false` avoids blocking real applies, and a read-only role is enough. HCP Terraform's health assessments or Spacelift's drift detection do the same without writing the pipeline.

</details>

### Q72. Which pre-commit hooks do you recommend? 🟢

<details>
<summary><b>Answer</b></summary>

With `antonbabenko/pre-commit-terraform`: `terraform_fmt`, `terraform_validate`, `terraform_tflint`, `terraform_docs`, `terraform_checkov` or `terraform_trivy`, plus generic `detect-private-key` and `check-merge-conflict`. They give developers feedback in seconds; CI repeats the same checks because hooks can be skipped.

</details>

### Q73. How do you speed up `terraform init` in CI? 🟢

<details>
<summary><b>Answer</b></summary>

Set `TF_PLUGIN_CACHE_DIR` and cache that directory between runs keyed on the lock file hash, or use a provider mirror close to the runners. Avoid re-downloading the 500 MB AWS provider per stack in a matrix job. Make sure the cache is keyed per platform.

</details>

### Q74. Why pin the Terraform version in CI and how? 🟢

<details>
<summary><b>Answer</b></summary>

Different CLI versions produce different plans and may write state in a format that older versions cannot read. Pin `required_version` in code, read the version from `.terraform-version` in the pipeline and upgrade through a PR that runs plans for every stack to confirm no diffs.

</details>

### Q75. Is the saved plan file safe to store as a CI artifact? 🔴

<details>
<summary><b>Answer</b></summary>

No, not casually. Plan files include variable values and resource attributes in plaintext (even those marked sensitive), so treat them like state: private artifact storage, short retention, access limited to the apply job, and never upload to a PR comment. Prefer `terraform show -no-color tfplan` filtered output for humans and keep the binary only in encrypted storage.

</details>

### Q76. Why use `-input=false` and `TF_IN_AUTOMATION`? 🟢

<details>
<summary><b>Answer</b></summary>

`-input=false` makes Terraform fail instead of waiting at a prompt for a missing variable, which would hang the job. `TF_IN_AUTOMATION=1` removes interactive hints like "Run terraform apply" from the output. Both are standard in pipelines.

</details>

### Q77. How do you present plan results on a PR? 🟢

<details>
<summary><b>Answer</b></summary>

Post a summary rather than 4,000 lines: counts per action, list of resources to be replaced or destroyed (highlighted), and a link to the full logs. Tools: tf-summarize, `terraform show -json tfplan | jq`, Atlantis comments, HCP Terraform's PR checks, and Infracost for monthly cost delta.

</details>

### Q78. What do you do when an apply fails halfway in the pipeline? 🔴

<details>
<summary><b>Answer</b></summary>

Do not panic and do not re-run blindly. Read the error and the partial result: Terraform already recorded what it created. Check the lock was released, fix the cause (permissions, quota, name collision, API error) and re-run the same plan or a fresh plan after review. If the lock is stuck, force-unlock after confirming the process died. Never `state rm` to "make it go green".

</details>

### Q79. How does GitLab CI handle Terraform state? 🟢

<details>
<summary><b>Answer</b></summary>

GitLab offers an HTTP backend per project (`https://gitlab.com/api/v4/projects/<id>/terraform/state/<name>`) with locking, versioning and access control, and the `gitlab-terraform` wrapper script configures it. It is convenient for small teams; for multi-account setups S3 or HCP often fits better.

</details>

### Q80. How do you automate provider and module upgrades? 🟡

<details>
<summary><b>Answer</b></summary>

Renovate or Dependabot reads `required_providers` and module `version` and opens PRs. The pipeline runs `terraform init -upgrade` (to update the lock file), `plan` and tests. Auto-merge only if the plan shows no changes and the update is a patch. Majors get a human and a runbook.

</details>

### Q81. How do you roll back a bad Terraform change? 🔴

<details>
<summary><b>Answer</b></summary>

Terraform has no rollback; you roll forward with a code revert. `git revert` the commit, review the plan (it should undo the change), apply. Some changes cannot be undone: deleted databases or replaced stateful resources need restore from snapshot or backup. This is why we gate destructive actions, enable deletion protection and review replacements carefully.

</details>

### Q82. How do GitOps controllers compare with a CI pipeline for Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

Controllers like Flux's tofu-controller reconcile Terraform in-cluster continuously, giving drift correction automatically but needing Kubernetes and careful blast-radius controls. A CI pipeline applies on merge and detects drift on a schedule, which is simpler and easier to approve. Continuous auto-correction of infrastructure is risky for stateful resources, so many teams only auto-apply low-risk stacks.

</details>

### Q83. What is your auto-apply policy? 🟢

<details>
<summary><b>Answer</b></summary>

Auto-apply on dev and sandbox after merge, since the cost of failure is small and speed matters. Staging applies automatically if the plan has no destroys. Production requires approval of the exact plan. Anything touching IAM, networking or data stores needs a second reviewer regardless of environment.

</details>

### Q84. What branching strategy suits Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

Trunk-based: short-lived feature branches, PR with plan, merge to main, apply from main. Main always reflects deployed state. Long-lived environment branches create drift between environments and painful merges.

</details>

### Q85. How do you stop engineers from applying from their laptops? 🔴

<details>
<summary><b>Answer</b></summary>

Remove the permission, not just the habit. Humans get read-only roles in prod; only the CI role can assume the apply role, restricted by OIDC `sub` claim to the protected branch. Backend IAM denies state writes to everyone else. Document a break-glass role with MFA, session recording and alerts.

</details>

## Security, Secrets and Access

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q86. How do you avoid hardcoded credentials in Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

Never put keys in `.tf` files or tfvars. Use IAM roles and OIDC federation in CI, instance profiles on runners, and SSO profiles for humans. Scan with gitleaks or detect-secrets pre-commit and in CI. For application secrets, store them in a secrets manager and pass ARNs.

</details>

### Q87. Is reading a secret with a data source safe? 🟡

<details>
<summary><b>Answer</b></summary>

Reading is fine from an IAM standpoint, but the value lands in state in plaintext (even if marked sensitive) and can show up in plans if used in a resource attribute. Better is to pass the secret's ARN and let the workload read it at runtime. Where Terraform must use the value, prefer ephemeral resources.

</details>

### Q88. How do ephemeral resources and write-only arguments keep secrets out of state? 🔴

<details>
<summary><b>Answer</b></summary>

An `ephemeral` block reads a value (for example from Secrets Manager) only during the run and never writes it to plan or state; write-only arguments such as `password_wo` consume it without persisting it.
```hcl
ephemeral "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db.id
}
resource "aws_db_instance" "main" {
  password_wo         = ephemeral.aws_secretsmanager_secret_version.db.secret_string
  password_wo_version = 1
}
```
Needs Terraform 1.11+ and provider support. Bump `password_wo_version` to apply a new password.

</details>

### Q89. What is the better pattern for RDS master passwords? 🟢

<details>
<summary><b>Answer</b></summary>

Set `manage_master_user_password = true` so RDS generates and rotates the password in Secrets Manager. Terraform never sees it. Applications fetch it by secret ARN (output the ARN), and rotation happens without a Terraform apply. This avoids both `random_password` in state and password drift.

</details>

### Q90. How do you scope IAM for the Terraform deploy role? 🟡

<details>
<summary><b>Answer</b></summary>

Start from a role per stack or per account with only the services that stack manages, add permission boundaries to stop it creating roles more powerful than itself and conditions on region and tags. Use a read-only role for plan. Generate a first draft from CloudTrail (IAM Access Analyzer policy generation), then tighten. Avoid `AdministratorAccess` on shared pipelines, except the break-glass role.

</details>

### Q91. What are dynamic provider credentials in HCP Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

HCP Terraform workspaces authenticate to AWS, Azure or GCP through OIDC with a short-lived token per run, configured by environment variables like `TFC_AWS_PROVIDER_AUTH` and `TFC_AWS_RUN_ROLE_ARN`. No static cloud keys are stored in the workspace, and trust can be limited per project, workspace or run phase.

</details>

### Q92. How do you defend against supply-chain risk in providers and modules? 🔴

<details>
<summary><b>Answer</b></summary>

Commit `.terraform.lock.hcl` so provider hashes are verified, install only from official registries or an internal mirror, pin module versions or commit SHAs, review diffs of upgrades, enable signature verification (providers are signed with GPG) and scan modules with Checkov before adoption. Restrict egress from CI to approved registries.

</details>

### Q93. What is the real threat model of a Terraform pipeline? 🔴

<details>
<summary><b>Answer</b></summary>

Anyone who can change code that gets planned can execute code: `data "external"`, `local-exec` provisioners, a malicious module, even provider configuration can run commands on the runner with its credentials. So: plan jobs run with read-only creds, PRs from forks do not get secrets, CODEOWNERS covers module sources and `.tf` that include executables, runners are ephemeral and network-restricted, and apply happens only from protected branches.

</details>

### Q94. How do you make sure resources are encrypted by default? 🟢

<details>
<summary><b>Answer</b></summary>

Bake it into modules (`server_side_encryption_configuration`, `storage_encrypted = true`, `encrypted = true` on EBS) with a KMS key input, then check with policy: Checkov rules such as `CKV_AWS_19` (S3 encryption) or custom OPA rules on plan JSON. Also set account-level defaults (EBS default encryption, S3 default) in the baseline stack.

</details>

### Q95. How do you stop public S3 buckets? 🟢

<details>
<summary><b>Answer</b></summary>

Defense in depth: the module always creates `aws_s3_bucket_public_access_block` with all four flags true, account-level Block Public Access in the baseline, an SCP denying `s3:PutBucketPublicAccessBlock` changes and a Checkov/OPA check in CI. Exceptions (static websites) are explicit and approved.

</details>

### Q96. Who should be able to read Terraform state? 🟢

<details>
<summary><b>Answer</b></summary>

Only the pipeline roles and a small audited group, because state contains secrets and a full inventory of the environment. If many teams need outputs, publish them through SSM or a read-only `outputs` stack rather than granting state access.

</details>

### Q97. Does `sensitive = true` protect secrets? 🟢

<details>
<summary><b>Answer</b></summary>

It only hides values in CLI output and logs. State and plan files still contain them in plaintext, and anyone with state read access sees them. Combine with a locked-down backend, and use ephemeral values to avoid storing secrets at all.

</details>

### Q98. Plan output in CI logs leaked a secret. What now? 🔴

<details>
<summary><b>Answer</b></summary>

Rotate the secret immediately, purge or restrict the logs, and find the leak path: an attribute printed in the plan (not all attributes are marked sensitive by providers), a `local-exec` echo, or debug logging (`TF_LOG=TRACE` prints request bodies). Fix by marking inputs sensitive, avoiding secrets in resource arguments, enabling log masking in CI and never using TRACE logs in shared pipelines.

</details>

### Q99. How do SCPs and Terraform policy checks complement each other? 🔴

<details>
<summary><b>Answer</b></summary>

Policy-as-code in CI catches issues at PR time, with friendly messages, but can be bypassed by someone with console access. SCPs and Config rules enforce in the cloud regardless of how the call was made but give blunt `AccessDenied` errors. Use both: early feedback plus preventive guardrails.

</details>

### Q100. How do you handle IAM and security-critical resources? 🟡

<details>
<summary><b>Answer</b></summary>

Put them in a separate stack and state with its own pipeline, stricter approvers and fewer humans with access. App teams request roles through a module or a PR to the identity repo. It keeps privilege escalation paths short and visible.

</details>

### Q101. What is your break-glass process for emergency changes? 🟡

<details>
<summary><b>Answer</b></summary>

A role requiring MFA and approval, with session logging and automatic alerts to the security channel. The engineer makes the minimal console or CLI change to stop the incident, then back-ports the change into Terraform within the same day and runs a plan to confirm no drift remains. Post-incident review checks the process was followed.

</details>

### Q102. How do you rotate a secret that Terraform created? 🟡

<details>
<summary><b>Answer</b></summary>

Avoid Terraform-owned rotation. Create the secret shell with Terraform, let Secrets Manager rotation or the service rotate it and add `lifecycle { ignore_changes = [secret_string] }` if Terraform sets the initial value. If you must rotate via Terraform, taint or replace `random_password` (or change its `keepers`) and apply, accepting the new value lands in state.

</details>

### Q103. Should you use the Vault provider to fetch secrets? 🟡

<details>
<summary><b>Answer</b></summary>

It centralises secrets and supports dynamic credentials, but values read through the provider are stored in state like any data source. Use Vault's dynamic credentials for Terraform's own authentication (short-lived cloud creds) and let applications talk to Vault directly for runtime secrets.

</details>

### Q104. What is on your Terraform PR security checklist? 🔴

<details>
<summary><b>Answer</b></summary>

Does it open anything to `0.0.0.0/0`? New IAM actions with `*`? Encryption on? Logging on? Public access? Deletion protection? Any `local-exec`, `external` or new modules and providers (source and version pinned)? Any secret in code, outputs or tfvars? Any `-/+` or `-` on stateful resources? Does the plan match the description? Automate what you can; humans review the rest.

</details>

### Q105. How do you handle KMS keys in Terraform safely? 🟢

<details>
<summary><b>Answer</b></summary>

Set `deletion_window_in_days` to 30, enable rotation, add `prevent_destroy = true` and write explicit key policies (do not rely on defaults) so you cannot lock yourself out. Deleting a key makes everything encrypted with it unreadable, so changes go through the highest review tier.

</details>

### Q106. How do CI runners reach state and APIs privately? 🟡

<details>
<summary><b>Answer</b></summary>

Run self-hosted runners inside the VPC with VPC endpoints for S3, STS and other services, and restrict the bucket policy to `aws:SourceVpce` where practical. Egress is allow-listed to the registry mirror and cloud APIs. It reduces exposure but adds ops work; hosted runners with OIDC are acceptable for many teams.

</details>

### Q107. How do you keep tags and compliance enforced after deployment? 🟡

<details>
<summary><b>Answer</b></summary>

Terraform enforces at deploy time; AWS Config rules, tag policies and Security Hub continuously evaluate what exists, catching console edits and untagged resources from other tools. Violations feed into tickets, and the drift pipeline highlights anything that diverged from code.

</details>

### Q108. How do you protect against a rogue module author or a compromised module repo? 🔴

<details>
<summary><b>Answer</b></summary>

Pin to commit SHAs or signed tags for internal modules, require two reviewers on module repos, run the module test suite and Checkov on every release, keep a private registry as the single source and block direct Git sources by policy. Review upgrade diffs rather than trusting version bumps.

</details>

### Q109. How do you manage tfvars that contain sensitive values? 🟡

<details>
<summary><b>Answer</b></summary>

Avoid it: use secret references. When unavoidable, encrypt files with SOPS (KMS-backed) and decrypt in CI via the `carlpett/sops` provider or a pipeline step, or inject through `TF_VAR_` from the CI secret store or HCP Terraform sensitive variables. Never commit plain `secrets.tfvars`.

</details>

### Q110. How does SOPS fit with Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

The `sops_file` data source (from the community sops provider) decrypts a SOPS-encrypted YAML or JSON using KMS credentials available to the runner, so secrets live encrypted in Git. The decrypted values enter state, so the same state-protection rules apply.

</details>

## Testing and Policy as Code

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q111. How do you test infrastructure code, and what does the testing pyramid look like? 🟢

<details>
<summary><b>Answer</b></summary>

Bottom to top: static checks (`fmt`, `validate`, tflint), security and policy scans (Checkov, Trivy, OPA), plan-level tests (`terraform test` with `command = plan`, mock providers), then real integration tests that apply into a sandbox account (`terraform test` with apply, Terratest) and finally smoke checks after deploy. Many cheap tests, few expensive ones.

</details>

### Q112. What does the static-analysis layer catch? 🟢

<details>
<summary><b>Answer</b></summary>

`fmt -check` enforces style, `validate` catches syntax, type and reference errors, tflint finds provider-specific mistakes (invalid instance types, deprecated syntax, unused declarations), and secret scanners catch leaked keys. All run in seconds without credentials, so they run on every commit.

</details>

### Q113. What does tflint do, and how is it configured? 🟢

<details>
<summary><b>Answer</b></summary>

tflint lints Terraform code for errors `validate` misses, such as `t2.nanoo` as an instance type, deprecated interpolation or unused variables. Enable provider rulesets in `.tflint.hcl` and run `tflint --init` once.
```hcl
plugin "aws" {
  enabled = true
  version = "0.38.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
```

</details>

### Q114. How do Checkov and Trivy differ? 🟡

<details>
<summary><b>Answer</b></summary>

Both scan IaC for misconfigurations. Checkov (Prisma Cloud) has a large policy library, supports custom Python and YAML policies and can scan the plan JSON, which resolves variables and modules. Trivy absorbed tfsec and covers images, filesystems and IaC in one tool. We pick one as the gate, scan plan output for accuracy and manage suppressions in code.
```bash
terraform show -json tfplan > plan.json
checkov -f plan.json --framework terraform_plan
```

</details>

### Q115. How does `terraform test` work? 🟢

<details>
<summary><b>Answer</b></summary>

Create `tests/*.tftest.hcl` with `run` blocks. Each run plans or applies the configuration (default is apply) with optional `variables`, evaluates `assert` blocks and tears down at the end in reverse order.
```hcl
run "invalid_cidr_rejected" {
  command = plan
  variables { vpc_cidr = "not-a-cidr" }
  expect_failures = [var.vpc_cidr]
}
```
Run with `terraform test` or `terraform test -filter=tests/vpc.tftest.hcl -verbose`.

</details>

### Q116. How do mock providers work in `terraform test`? 🔴

<details>
<summary><b>Answer</b></summary>

Since 1.7 you can declare `mock_provider "aws" {}` so tests run without credentials or cloud calls; computed attributes get generated values, and you can set specific ones with `mock_resource`, `mock_data` or `override_resource`.
```hcl
mock_provider "aws" {
  mock_data "aws_availability_zones" {
    defaults = { names = ["ap-south-1a", "ap-south-1b"] }
  }
}
```
They are great for logic tests (conditionals, `for_each` keys, validation) in seconds, but cannot prove the cloud accepts your config.

</details>

### Q117. When do you use Terratest instead of `terraform test`? 🟡

<details>
<summary><b>Answer</b></summary>

Terratest (Go) is for tests that need more than assertions on Terraform values: calling the deployed endpoint over HTTP, SSHing to a host, retries, parallel test cases and reuse of Go libraries. Typical flow: `terraform.InitAndApply`, validate, `defer terraform.Destroy`. For simpler module checks, native `terraform test` has less overhead and no extra language.
```go
defer terraform.Destroy(t, opts)
terraform.InitAndApply(t, opts)
url := terraform.Output(t, opts, "alb_url")
http_helper.HttpGetWithRetry(t, url, nil, 200, "ok", 30, 10*time.Second)
```

</details>

### Q118. How do you control the cost and cleanliness of integration tests? 🟡

<details>
<summary><b>Answer</b></summary>

Run them in a dedicated sandbox account with budgets and SCPs, use unique name suffixes per run, run on merge or nightly rather than per commit, always `defer Destroy`, and run a janitor (cloud-nuke or aws-nuke) for leaked resources. Use small instance sizes and skip slow resources (NAT gateways, RDS Multi-AZ) in test configs.

</details>

### Q119. What is policy as code and where does it run? 🟢

<details>
<summary><b>Answer</b></summary>

Rules about what infrastructure is allowed (encryption, regions, instance types, tags) written as code and evaluated automatically. They run at several points: pre-commit for speed, in CI on the plan JSON, in HCP Terraform between plan and apply (Sentinel or OPA), and in the cloud through SCPs and Config.

</details>

### Q120. How do OPA and Conftest check a Terraform plan? 🟡

<details>
<summary><b>Answer</b></summary>

Export the plan to JSON and evaluate Rego rules against `resource_changes`.
```rego
package terraform
deny[msg] {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket"
  not rc.change.after.server_side_encryption_configuration
  msg := sprintf("%s must be encrypted", [rc.address])
}
```
```bash
terraform show -json tfplan > plan.json && conftest test plan.json -p policy/
```

</details>

### Q121. What are Sentinel enforcement levels? 🟡

<details>
<summary><b>Answer</b></summary>

Sentinel is HashiCorp's policy language for HCP Terraform and Terraform Enterprise. Policies in a policy set are `advisory` (warn only), `soft-mandatory` (block unless an authorised user overrides) or `hard-mandatory` (block always). Start new policies as advisory, measure noise, then escalate.

</details>

### Q122. How do you choose between Sentinel and OPA? 🔴

<details>
<summary><b>Answer</b></summary>

Sentinel is tightly integrated with HCP Terraform, readable and has cost-estimate and run data imports, but is proprietary and works only in HashiCorp's platform. OPA is open source, uses Rego (a steeper learning curve), works everywhere (CI, Kubernetes admission, Atlantis) and HCP Terraform supports it too. If you want one policy language across Terraform and Kubernetes, choose OPA; if you are all-in on HCP Terraform and want simplicity, Sentinel is fine.

</details>

### Q123. How do you write a custom Checkov policy? 🟡

<details>
<summary><b>Answer</b></summary>

A YAML policy (or Python check) defines a resource type and attribute condition.
```yaml
metadata:
  id: "CKV2_ACME_1"
  name: "S3 buckets must have Owner tag"
  category: "GENERAL_SECURITY"
definition:
  cond_type: attribute
  resource_types: ["aws_s3_bucket"]
  attribute: "tags.Owner"
  operator: exists
```
Run with `checkov -d . --external-checks-dir ./policies`. Keep custom policies in one repo, versioned and tested.

</details>

### Q124. How do you test that a module is idempotent? 🟡

<details>
<summary><b>Answer</b></summary>

Apply it, then run plan and assert no changes. In `terraform test`, use two sequential `run` blocks: the second runs `command = plan` against the same state and asserts nothing changes; in Terratest use `InitAndApplyAndIdempotent`. A non-empty second plan means a perpetual diff bug.

</details>

### Q125. What does a contract test for a module look like? 🔴

<details>
<summary><b>Answer</b></summary>

In `tests/`, one run block per documented behaviour: required inputs produce expected outputs, toggles create or skip resources, invalid inputs fail validation with `expect_failures`, and `override_` or mock providers make it fast. Add one `command = apply` test against real AWS that checks outputs. Consumers can then upgrade confidently, because every promised behaviour has a test.

</details>

### Q126. Is LocalStack a good testing target? 🟡

<details>
<summary><b>Answer</b></summary>

Good for fast feedback on basic services (S3, SQS, Lambda, DynamoDB) without cost and good for local development. Weak for IAM semantics, networking and newer services, and some features are paid. We use it for module smoke tests, not as proof a deployment will work in AWS.

</details>

### Q127. How do you test against multiple provider versions? 🟡

<details>
<summary><b>Answer</b></summary>

Run the module's test suite in a CI matrix with the minimum supported provider version (the lower bound in `required_providers`) and the latest, using `terraform init -upgrade` for latest and a lock file for the minimum. It catches the case where you used an argument introduced after your stated lower bound.

</details>

### Q128. How do you verify the deployed result, not only the plan? 🟡

<details>
<summary><b>Answer</b></summary>

Post-apply checks: a `check` block with an HTTP data source, InSpec or Terratest assertions, cloud-native checks (AWS Config conformance packs) and synthetic monitoring. Plan-time assertions prove intent; post-apply checks prove the service actually works.

</details>

### Q129. How would you fail a pipeline when a plan destroys something critical? 🔴

<details>
<summary><b>Answer</b></summary>

Inspect the plan JSON for delete actions on protected resource types.
```bash
terraform show -json tfplan | jq -e '
  [.resource_changes[]
   | select(.change.actions | index("delete"))
   | select(.type | IN("aws_db_instance","aws_s3_bucket","aws_kms_key"))] | length == 0'
```
A non-zero exit fails the job, forcing an explicit override path. Equivalent logic can live in OPA or Sentinel.

</details>

### Q130. Which hooks does pre-commit-terraform provide? 🟢

<details>
<summary><b>Answer</b></summary>

`terraform_fmt`, `terraform_validate`, `terraform_docs`, `terraform_tflint`, `terraform_checkov`, `terraform_trivy`, `terraform_providers_lock` and `terrascan`, configured through `.pre-commit-config.yaml`. Install with `pre-commit install`, and run `pre-commit run -a` in CI.

</details>

### Q131. How do you use Infracost in a PR? 🟡

<details>
<summary><b>Answer</b></summary>

Infracost parses the plan or code and posts a cost diff (monthly delta and top changes) as a PR comment. We add a policy: PRs that raise costs beyond a threshold need FinOps approval. It makes cost visible at the time of change instead of on next month's bill.
```bash
infracost breakdown --path tfplan.json --format json --out-file infracost.json
```

</details>

### Q132. Preconditions or external policy: which to use? 🟡

<details>
<summary><b>Answer</b></summary>

`precondition` and `validation` live with the code, run everywhere (including laptops and tests) and express module-specific assumptions. External policy (OPA, Sentinel, Checkov) expresses organization-wide rules that apply across all repos and cannot be removed by the module author. We use both: module invariants in-code, company rules outside.

</details>

### Q133. How do you handle policy exceptions without losing control? 🔴

<details>
<summary><b>Answer</b></summary>

Allow waivers only in code and with metadata: `#checkov:skip=CKV_AWS_18:Access logs not needed, ticket SEC-421, expires 2026-12-31`. Reviewers (security) are required via CODEOWNERS where a skip appears, a report lists active skips and an expiry check fails the build when a waiver is stale. Exceptions in a spreadsheet get forgotten.

</details>

### Q134. What does Trivy offer for Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

`trivy config .` scans Terraform (and Kubernetes, Dockerfiles) for misconfigurations using the former tfsec rules and Rego-based checks, outputs SARIF for GitHub code scanning and supports `--ignorefile`. It can evaluate variables and modules but, as with any static scan, plan JSON gives the most accurate results.

</details>

### Q135. What do you run on pull requests versus nightly? 🟡

<details>
<summary><b>Answer</b></summary>

Per PR: fmt, validate, tflint, policy scan, plan, mock-provider tests and cost diff, so feedback arrives in a few minutes. Nightly or on merge: real-apply integration tests, drift detection plans for every stack and dependency update checks. Anything slow and expensive stays off the PR path, otherwise people bypass it.

</details>

## Refactoring, Migration and Drift

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q136. What is your workflow for refactoring Terraform safely? 🟡

<details>
<summary><b>Answer</b></summary>

Make the change in a branch, add `moved` blocks for renamed or relocated addresses and run `terraform plan`: the goal is "no changes" (plus move notes). Apply in dev first. If the plan shows destroy and create where you expected a move, stop and fix the addresses. Never combine a refactor with a functional change in one PR.

</details>

### Q137. How do you rename a resource inside a shared module without breaking consumers? 🔴

<details>
<summary><b>Answer</b></summary>

Add a `moved` block inside the module so each consumer's state migrates on its next apply.
```hcl
moved {
  from = aws_security_group.this
  to   = aws_security_group.main
}
```
Moved blocks inside a module apply to every instance of it, including `for_each` keys. Keep them for at least a couple of releases because consumers may skip versions, and document the release as a minor with a migration note.

</details>

### Q138. How do you split one state into two without downtime? 🔴

<details>
<summary><b>Answer</b></summary>

1. Back up both states (`terraform state pull`).
2. In the new stack, write the config for the resources moving out, and use `import` blocks with their real IDs (or `terraform state mv -state-out`).
3. Run plan in the new stack: it must show no changes.
4. In the old stack, replace the resource blocks with `removed { lifecycle { destroy = false } }` and plan: it must show only removals from state.
5. Apply the new stack, then the old one.
No real resource is touched if every plan is clean.

</details>

### Q139. How do you merge two states? 🟡

<details>
<summary><b>Answer</b></summary>

Move resources from stack B into A with `terraform state mv -state=b.tfstate -state-out=a.tfstate ADDR ADDR` (after pulling both and with backups), copy the matching config blocks into A and plan for zero diff before pushing. For many resources, using import blocks in A and `removed` in B is easier to review and safer.

</details>

### Q140. How do you import a large existing environment efficiently? 🟡

<details>
<summary><b>Answer</b></summary>

Use `import` blocks with `terraform plan -generate-config-out=generated.tf` to draft configuration, then clean it up into modules and variables (generated code is verbose and hardcoded). Tools like Terraformer or aws2tf can inventory a region. Import in layers (network first), validate with empty plans and avoid importing resources you plan to retire.

</details>

### Q141. How do you upgrade a major version of the AWS provider? 🔴

<details>
<summary><b>Answer</b></summary>

Read the upgrade guide, pin the new major on a branch, run `init -upgrade` and `plan` to see deprecations and replacements. Fix removed arguments, then apply to dev and run tests. Notable history: in v4 the monolithic `aws_s3_bucket` was split into separate resources (`aws_s3_bucket_versioning`, `..._acl` and others), and v5 and v6 removed or changed several arguments. Upgrade one major at a time and never skip guides.

</details>

### Q142. How do you upgrade the Terraform CLI across many stacks? 🟡

<details>
<summary><b>Answer</b></summary>

Check release notes for breaking changes, run `plan` with the new version in CI across all stacks (read-only) to catch diffs, upgrade dev stacks first, then bump `required_version` and tooling versions. Once any stack applies with the new version, state may no longer be readable by older CLIs, so move fast within a team after the first apply.

</details>

### Q143. How would you migrate from CloudFormation to Terraform? 🔴

<details>
<summary><b>Answer</b></summary>

Per stack, not all at once. Import the real resources into Terraform, set the CloudFormation stack's `DeletionPolicy: Retain` on everything, then delete the CFN stack (or `retain-resources`) so resources survive without a manager. Confirm with an empty Terraform plan before removing the CFN stack. Tools such as `former2` help to draft code, though results need cleanup.

</details>

### Q144. How do you migrate from Terraform to OpenTofu in production? 🔴

<details>
<summary><b>Answer</b></summary>

Pin and match the current Terraform version's feature set, back up all state, install `tofu` in a branch pipeline, run `tofu init` and `tofu plan` for every stack and require empty plans. Roll out stack by stack, since once OpenTofu writes state the Terraform CLI may refuse to read newer formats. Keep a documented rollback (state backup plus Terraform binary) and check any providers or modules sourced only from HCP.

</details>

### Q145. A resource was replaced by mistake and the database is gone. What now, and how do you prevent it? 🔴

<details>
<summary><b>Answer</b></summary>

Restore from the latest snapshot or point-in-time backup into a new instance, import it into state at the right address (or let Terraform create it from `snapshot_identifier`), and repoint applications. Prevention: `deletion_protection = true`, `prevent_destroy`, `skip_final_snapshot = false` with a `final_snapshot_identifier`, plan checks that fail on deletes of stateful types and approvals for replacements.

</details>

### Q146. How do you let autoscaling or other systems own attributes? 🟡

<details>
<summary><b>Answer</b></summary>

Use `lifecycle { ignore_changes = [...] }` for exactly those attributes (`desired_capacity`, `task_definition` image, `tags["..."]`). Document why in a comment and keep the list as narrow as possible, because ignored attributes are blind spots for drift detection.

</details>

### Q147. What do you do about a provider bug causing a perpetual diff? 🟡

<details>
<summary><b>Answer</b></summary>

Reproduce with the minimal config, search the provider's issue tracker, try a newer version, and work around with `ignore_changes` on the attribute or by normalising the value in code. Record the workaround with a link to the issue so it can be removed on upgrade.

</details>

### Q148. How do you diagnose a perpetual diff? 🔴

<details>
<summary><b>Answer</b></summary>

Run `terraform plan` twice and read the diff closely. Common causes: JSON documents with different key ordering or whitespace (use `jsonencode`), lists vs sets with order differences, provider-normalised values (lowercased names, trailing dots in DNS, full ARNs), tags added by AWS or other tools and computed defaults. Use `TF_LOG=DEBUG` to see what the API returns, compare `terraform state show` against config and fix the config to the canonical form.

</details>

### Q149. How do you handle an emergency console change? 🟡

<details>
<summary><b>Answer</b></summary>

Make the change, record it in the incident channel, then immediately mirror it in code and confirm via `terraform plan` that no diff remains. If the change should be reverted, apply Terraform to overwrite it. The point is to close the loop quickly so drift does not accumulate and the next apply does not surprise anyone.

</details>

### Q150. What is your checklist before touching state manually? 🟡

<details>
<summary><b>Answer</b></summary>

Announce a freeze on the stack, take a backup (`terraform state pull > backup.tfstate`) and confirm bucket versioning, take the lock (no one else applying), make the smallest change with `state mv`/`rm`/`import` rather than editing JSON, run `terraform plan` to verify, and record the commands in the ticket.

</details>

### Q151. How do you safely remove a protected resource in production? 🔴

<details>
<summary><b>Answer</b></summary>

In sequence: first apply a change that turns off protection (`deletion_protection = false`, remove `prevent_destroy`) as its own PR, confirm it applied, then remove the resource in a second PR with a plan showing only that deletion, and have a snapshot or backup taken beforehand. Mixing protection removal and deletion in one PR hides the intent.

</details>

### Q152. What do you do when a resource type is deprecated, like inline `acl` on `aws_s3_bucket`? 🟡

<details>
<summary><b>Answer</b></summary>

Read the provider's migration guide, adopt the replacement resources (`aws_s3_bucket_acl`, `aws_s3_bucket_ownership_controls`), import existing settings into them with import blocks if needed and verify an empty plan. Do it per module with a minor-version release, so consumers upgrade at their own pace.

</details>

### Q153. What red flags do you look for when reading a plan? 🟢

<details>
<summary><b>Answer</b></summary>

`-/+` or `-` on databases, volumes, load balancers or KMS keys; `# forces replacement` on names and IDs; unexpected changes in resources you did not touch (drift or provider upgrade); wide `0.0.0.0/0`; IAM `*` actions; "known after apply" on critical arguments; and a plan summary that is bigger than the PR description suggests.

</details>

### Q154. How do you replace an Auto Scaling group's launch template without downtime? 🔴

<details>
<summary><b>Answer</b></summary>

Use `create_before_destroy` with a name that changes per version (or `name_prefix`), and an `instance_refresh` block on the ASG so instances roll gradually with `min_healthy_percentage`.
```hcl
instance_refresh {
  strategy = "Rolling"
  preferences { min_healthy_percentage = 90 }
}
```
Pair with health checks on the load balancer, and test in staging with realistic traffic.

</details>

### Q155. How do you roll out module major upgrades across many consumers? 🟡

<details>
<summary><b>Answer</b></summary>

Publish release notes with a migration section and `moved` blocks, let Renovate open PRs in each repo, require green plans, and roll out by environment (dev, then qa, then prod) and by team. Track adoption on a dashboard, set a deprecation date for the old major and support it with security fixes only until then.

</details>

## HCP Terraform, OpenTofu, Scale and Cost

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q156. How do you design workspaces and projects in HCP Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

One workspace per stack per environment (`network-prod`, `app-dev`), grouped into projects by team or domain, with project-level permissions and variable sets for shared settings (cloud auth, tags). Avoid one workspace per giant monolith. Use naming conventions and tags to filter, and Terraform provider `tfe` to manage workspaces themselves as code.

</details>

### Q157. What execution modes does HCP Terraform offer? 🟢

<details>
<summary><b>Answer</b></summary>

Remote (runs execute on HashiCorp's infrastructure), local (CLI runs locally and only state is stored in HCP) and agent (runs execute on your own `tfc-agent` inside your network). Agents are the answer when Terraform must reach private APIs such as an on-prem vCenter or a private Kubernetes endpoint.

</details>

### Q158. What are the three run workflows in HCP Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

VCS-driven (a push or PR triggers plan and apply), CLI-driven (`terraform plan` from a laptop streams to a remote run via the `cloud` block) and API-driven (CI calls the API to upload a configuration version and create a run). Most teams use VCS or API-driven runs so every change is tied to a commit.

</details>

### Q159. How are variable sets and sensitive variables used? 🟢

<details>
<summary><b>Answer</b></summary>

A variable set holds shared Terraform or environment variables (for instance `AWS_REGION`, tags or credentials config) and attaches to projects or workspaces. Variables marked sensitive are write-only in the UI: they cannot be read back. Combined with dynamic credentials, nothing long-lived needs to be stored there.

</details>

### Q160. How is HCP Terraform priced, and how does it affect design? 🟡

<details>
<summary><b>Answer</b></summary>

Pricing is based on resources under management (RUM) per month in current plans, so thousands of cheap resources like IAM policy attachments or security group rules count. Review the current pricing page before committing: it can push teams toward self-hosted options such as Atlantis, Spacelift, env0, Scalr or plain CI pipelines for cost reasons.

</details>

### Q161. How do you choose between HCP Terraform, Atlantis, Spacelift and plain GitHub Actions? 🔴

<details>
<summary><b>Answer</b></summary>

GitHub Actions: cheapest, fully flexible, you build locking, approvals, drift and policy yourself. Atlantis: open source, PR-centric workflow, you host and secure it. HCP Terraform: managed state, runs, policy, private registry and SSO out of the box, at a price. Spacelift, env0 and Scalr: managed platforms supporting Terraform, OpenTofu and others with strong policy and stack dependency features. Decide by team size, compliance needs and appetite for building internal tooling.

</details>

### Q162. What drives the choice between Terraform and OpenTofu? 🟡

<details>
<summary><b>Answer</b></summary>

Licence (BSL versus MPL under Linux Foundation governance), feature needs (OpenTofu's native state encryption, `-exclude`, provider `for_each`; Terraform's ephemeral resources, HCP integration and Stacks), vendor-support contracts, ecosystem tooling compatibility (most tools support both) and organisational risk appetite. Both run the same providers. Test your CI tools and policies against `tofu` before committing.

</details>

### Q163. How do you configure OpenTofu state encryption? 🔴

<details>
<summary><b>Answer</b></summary>

Define a key provider, a method and the encryption target in the `terraform` block.
```hcl
terraform {
  encryption {
    key_provider "pbkdf2" "main" { passphrase = var.state_passphrase }
    method "aes_gcm" "default" { keys = key_provider.pbkdf2.main }
    state { method = method.aes_gcm.default }
    plan  { method = method.aes_gcm.default }
  }
}
```
Production setups use a KMS key provider (AWS KMS, GCP KMS) instead of a passphrase. Encryption is client-side, so even a bucket reader sees ciphertext. Back up keys: losing them means losing state.

</details>

### Q164. What is Terraform Enterprise and when would you run it? 🟢

<details>
<summary><b>Answer</b></summary>

The self-hosted distribution of HCP Terraform, deployed as a container on your infrastructure. Reasons: data residency, air-gapped or regulated environments and full network control. You take on upgrades, scaling, backup and database operations. Many orgs instead use HCP Terraform with agents to keep execution private.

</details>

### Q165. What are no-code modules and the private registry in HCP Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

The private registry hosts internal modules and providers with version history and docs. No-code provisioning lets users deploy a registry module from a form without writing HCL; a workspace is created with fixed module version. It suits platform teams that offer vetted building blocks to developers.

</details>

### Q166. What are run tasks and notifications? 🟢

<details>
<summary><b>Answer</b></summary>

Run tasks call external services (security scanners, cost tools, approval systems) at pre-plan, post-plan or pre-apply stages and can block or warn. Notifications send run events to Slack, Teams, email or webhooks. Together they integrate the Terraform workflow into existing tooling without custom pipelines.

</details>

### Q167. A plan takes 40 minutes. How do you speed it up? 🔴

<details>
<summary><b>Answer</b></summary>

First find out why: `TF_LOG=DEBUG` timestamps or trace the provider calls. Typical fixes: split the state into smaller stacks (biggest win), disable refresh for data sources that are slow or replace them with variables, reduce `data` lookups in loops, raise `-parallelism` if the API allows, stop using `for_each` over thousands of tiny resources where a single higher-level resource supports lists, cache providers and run on a runner close to the API. `-refresh=false` and `-target` are band-aids, not a strategy.

</details>

### Q168. How do you tune `-parallelism` and deal with API throttling? 🟡

<details>
<summary><b>Answer</b></summary>

Default is 10 concurrent operations. Higher speeds big creates until the API returns `Throttling`/`RequestLimitExceeded`; lower it for rate-sensitive APIs (IAM, Route 53, GitHub). On the AWS provider add `max_retries` and `retry_mode = "adaptive"`. If throttling persists you are asking too much per state: split it.

</details>

### Q169. How does Terraform scale with thousands of `for_each` instances? 🔴

<details>
<summary><b>Answer</b></summary>

Each instance is a graph node with its own refresh call, and the whole plan is held in memory, so CPU, RAM and API calls grow linearly (and some graph steps worse). Symptoms: slow plans and big memory use. Mitigate with splitting by domain, consolidating resources the provider can manage in bulk, and keeping module nesting shallow.

</details>

### Q170. How does the AWS provider handle rate limits and retries? 🟡

<details>
<summary><b>Answer</b></summary>

It retries throttled calls with exponential backoff up to `max_retries` (default 25 in recent versions) and supports `retry_mode`. You can set these in the provider block. If a pipeline needs frequent tuning of retries, the real problem is state size or too many parallel stacks hitting the same account.

</details>

### Q171. How do you run a Terraform cost-control programme? 🔴

<details>
<summary><b>Answer</b></summary>

Visibility: mandatory cost tags, Infracost diffs in PRs and dashboards by tag. Prevention: policy limiting instance families and sizes, budgets and alerts per account, quotas. Waste removal: schedule shutdown or destroy of non-prod environments, TTL tags on ephemeral resources, storage lifecycle rules, rightsizing recommendations fed back into modules. Ownership: each team sees its spend.

</details>

### Q172. How do you shut down non-production environments out of hours? 🟡

<details>
<summary><b>Answer</b></summary>

Options: scheduled pipeline running `terraform destroy` on preview stacks (and apply in the morning), an ASG `aws_autoscaling_schedule` or Instance Scheduler for stateful but idle resources, or TTL tags and a janitor. Destroy/apply cycles are only safe for stacks with no state worth keeping, so keep data layers separate.

</details>

### Q173. How do tags help with cost reporting? 🟢

<details>
<summary><b>Answer</b></summary>

Activate cost allocation tags (`Team`, `Service`, `Environment`) in the billing console, then Cost Explorer and CUR can group spend by them. Set tags through provider `default_tags` and enforce them with policy so spend does not land in "untagged".

</details>

### Q174. How does Terraform compare with Crossplane? 🟡

<details>
<summary><b>Answer</b></summary>

Crossplane runs in Kubernetes and continuously reconciles cloud resources defined as CRDs, which fits platform teams already living in Kubernetes and wanting self-service APIs. Terraform has a bigger provider ecosystem, a plan step and wider adoption outside Kubernetes. Crossplane gives drift correction by default but a more complex debugging model; Terraform is pull-request-driven.

</details>

### Q175. How realistic is "multi-cloud with Terraform"? 🔴

<details>
<summary><b>Answer</b></summary>

Terraform gives one workflow and language, not portable resources: an `aws_instance` and `azurerm_linux_virtual_machine` are different schemas, so you write per-cloud modules. It is realistic for managing multiple clouds and SaaS from one pipeline, and for abstracting at the module interface (a `compute` module with AWS and Azure implementations) at significant cost. True portability is better achieved at the application layer (containers, Kubernetes).

</details>

### Q176. How do you build a self-service platform with Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

Platform team publishes golden modules (service, database, queue) with secure defaults and few inputs; developers request them via a PR to a config repo or a catalog/no-code form; the pipeline plans, runs policy checks and applies. Guardrails (policy, quotas, approvals for expensive resources) replace manual tickets, and ownership tags route cost and alerts to the requesting team.

</details>

### Q177. What guardrails do you give developers in a self-service model? 🟡

<details>
<summary><b>Answer</b></summary>

Narrow module interfaces (they pick size from `small|medium|large`, not arbitrary instance types), validation on inputs, OPA or Sentinel policies on the plan, IAM permission boundaries for roles they create, budget alerts and approval gates for expensive or internet-facing resources. Developers get speed; the platform gets safety.

</details>

### Q178. How would you rebuild an entire region from code after a disaster? 🔴

<details>
<summary><b>Answer</b></summary>

Order matters: foundation (accounts, IAM, KMS, network), then data layers (restore from cross-region backups, since Terraform does not restore data), then compute and apps, then DNS cutover. Things that break the "just re-apply" story: secrets that were never in code, state in the lost region, manually created resources, hardcoded AZs and AMI IDs. We rehearse it with game days and keep state and backups in another region.

</details>

### Q179. How do you observe Terraform runs? 🟡

<details>
<summary><b>Answer</b></summary>

Emit run results to a central place: pipeline duration, success rate, number of stacks with drift, age of last successful apply per stack, and failed locks. HCP Terraform exposes run metrics and notifications; for CI, ship job metadata to your metrics system and alert on stacks not applied in N days or with persistent drift.

</details>

### Q180. How do you keep module documentation up to date? 🟢

<details>
<summary><b>Answer</b></summary>

Run `terraform-docs` in a pre-commit hook and CI check so the README's inputs, outputs and requirements tables are always generated from code.
```bash
terraform-docs markdown table --output-file README.md --output-mode inject .
```

</details>

## Rapid-Fire Commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. Format all files recursively and fail if anything needs formatting? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
terraform fmt -check -recursive -diff
```
Exit code 3 means files need formatting. Drop `-check` to rewrite them in place.

</details>

### Q182. List every resource address under one module? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
terraform state list module.network
```
Filter by prefix. Without arguments it lists everything in the current state.

</details>

### Q183. Show the attributes of one resource in state? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
terraform state show 'aws_instance.web["blue"]'
```
Quote addresses containing brackets, so the shell does not interpret them.

</details>

### Q184. Plan with a specific var file and write the plan? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
terraform plan -var-file=prod.tfvars -out=tfplan -input=false
```

</details>

### Q185. Validate configuration and get machine-readable output? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
terraform init -backend=false && terraform validate -json
```
`-backend=false` lets you validate in CI without credentials.

</details>

### Q186. Apply a saved plan and inspect it as JSON first? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
terraform show -json tfplan | jq '.resource_changes[] | {addr:.address, actions:.change.actions}'
terraform apply tfplan
```
Applying a plan file never prompts.

</details>

### Q187. Force the replacement of a single resource? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
terraform apply -replace='aws_instance.web["blue"]'
```
It shows the replacement in the plan for approval. This supersedes `terraform taint`.

</details>

### Q188. Import an existing resource, two ways? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform import aws_s3_bucket.logs acme-logs
```
or declare it in code (1.5+) and run `terraform plan -generate-config-out=gen.tf`:
```hcl
import { to = aws_s3_bucket.logs  id = "acme-logs" }
```

</details>

### Q189. Move a resource into a module without destroying it? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform state mv aws_s3_bucket.logs module.logging.aws_s3_bucket.this
```
Or add a `moved` block and plan; both only edit state. Verify with a clean plan.

</details>

### Q190. List, create and select workspaces? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform workspace list
terraform workspace new staging
terraform workspace select prod
terraform workspace show
```
Check `workspace show` before every apply when using workspaces.

</details>

### Q191. Release a stuck state lock? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform force-unlock -force 7f6c1b5e-0d3a-4c3b-9f0a-2d1e8b7a9c11
```
Use the ID from the error message and only after confirming no run is active.

</details>

### Q192. Turn on debug logging for a provider issue and keep it in a file? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
TF_LOG=DEBUG TF_LOG_PATH=./tf-debug.log terraform plan
TF_LOG_PROVIDER=TRACE terraform apply   # provider logs only
```
Remember logs can contain secrets; do not share them unredacted.

</details>

### Q193. Detect drift without changing anything and accept it into state? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform plan -refresh-only
terraform apply -refresh-only
```
The first previews drift; the second writes the observed values into state after you confirm.

</details>

### Q194. Make the lock file contain hashes for several platforms? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform providers lock -platform=linux_amd64 -platform=linux_arm64 -platform=darwin_arm64
```
Run it after upgrades so CI on Linux and laptops on macOS all verify providers.

</details>

### Q195. Read a single output for a script? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform output -raw db_endpoint
terraform output -json subnet_ids | jq -r '.[]'
```
`-raw` prints strings without quotes or newline.

</details>

### Q196. List resources the plan will destroy with jq? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform show -json tfplan | jq -r '.resource_changes[] | select(.change.actions|index("delete")) | .address'
```
Handy as a CI gate.

</details>

### Q197. Upgrade provider versions within constraints and refresh the lock file? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform init -upgrade
git diff .terraform.lock.hcl
```
Review the lock diff and run a plan before committing.

</details>

### Q198. Destroy only one resource? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform destroy -target='aws_instance.temp'
# or: terraform apply -destroy -target='aws_instance.temp'
```
Targeted destroy is for exceptions. Prefer removing the resource from code and applying.

</details>

### Q199. Run one test file with verbose output, or filter tests? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform test -verbose
terraform test -filter=tests/vpc.tftest.hcl
terraform test -var="env=test"
```
Test files live in the module root or `tests/` and need `terraform init` first.

</details>

### Q200. Show the dependency graph as an image and the provider requirements tree? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform graph | dot -Tpng > graph.png
terraform providers
```
`terraform providers` shows which modules require which providers, helpful when debugging version conflicts.

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
