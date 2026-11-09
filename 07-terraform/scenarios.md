# Terraform — Scenario-Based Questions

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

> Real-world "what would you do" situations on Terraform and IaC, from first deploys to multi-account platforms · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Terraform](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Workflow and Providers](#workflow-and-providers) — Q1–Q25
- [Resources Variables and Expressions](#resources-variables-and-expressions) — Q26–Q50
- [Modules and Code Structure](#modules-and-code-structure) — Q51–Q75
- [State Backends and Workspaces](#state-backends-and-workspaces) — Q76–Q100
- [Import Refactoring Drift and Lifecycle](#import-refactoring-drift-and-lifecycle) — Q101–Q125
- [Secrets Provisioners and Policy as Code](#secrets-provisioners-and-policy-as-code) — Q126–Q150
- [Pipelines Terraform Cloud and OpenTofu](#pipelines-terraform-cloud-and-opentofu) — Q151–Q175
- [Testing Upgrades and Large-Scale Operations](#testing-upgrades-and-large-scale-operations) — Q176–Q200

---

## Workflow and Providers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. A new teammate is about to run Terraform for the first time against a shared dev account. Which commands do you walk them through, and in what order? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Someone with no muscle memory for the workflow, a shared account, and a real chance of an accidental change.

**What I'd do →**
1. `terraform init` to download providers and configure the backend.
2. `terraform fmt -recursive` and `terraform validate` to catch syntax problems early.
3. `terraform plan -out=tfplan` and read every line of the diff.
4. `terraform apply tfplan` so only the reviewed plan is applied.

**Why →** Plan-then-apply with a saved plan guarantees what was reviewed is what runs, and the teammate learns that `plan` is free while `apply` is not.

**Interview tip →** Say "apply a saved plan file", not just "run apply"; it signals you think about review and staleness.

</details>

### Q2. Your manager asks why we bother with `terraform plan` when `apply` shows the same diff before asking for confirmation. What do you say? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A fair question; interactive `apply` does print a plan, but it re-computes it at that moment.

**What I'd do →**
1. Run `terraform plan -out=tfplan` in the PR pipeline and attach the human-readable output to the PR.
2. After approval, run `terraform apply tfplan` which applies exactly that artifact.

**Why →** A separate plan lets reviewers approve a concrete diff, and a saved plan fails if state changed in between (stale plan). Interactive apply re-plans, so the diff you approved could differ from what runs.

**Interview tip →** Mention that plan files can contain secrets, so treat the artifact as sensitive.

</details>

### Q3. Two engineers get different provider versions and see different plans for the same code. How do you stop this? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Provider versions are floating and nobody is committing the dependency lock file.

**What I'd do →**
1. Declare constraints in `required_providers`.
2. Commit `.terraform.lock.hcl` to Git.
3. Upgrade deliberately with `terraform init -upgrade` in a dedicated PR.

```hcl
terraform {
  required_version = ">= 1.9.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.80"
    }
  }
}
```

**Why →** Constraints define the allowed range; the lock file pins the exact version and its checksums, so every machine and CI run uses identical providers.

**Interview tip →** "Constraint says what is allowed, lock file says what is used."

</details>

### Q4. Someone committed the `.terraform/` directory to the repo. What should be in version control and what should not? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic repo hygiene problem; `.terraform/` holds downloaded provider binaries and backend cache.

**What I'd do →**
1. Remove it from Git with `git rm -r --cached .terraform`.
2. Add a `.gitignore`.
3. Keep `.terraform.lock.hcl` committed.

```gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfplan
override.tf
*.auto.tfvars   # only if they hold secrets
```

**Why →** Provider binaries are large and platform-specific, and state or plan files can contain secrets. The lock file is the one generated file that must be shared.

**Interview tip →** If a state file was ever committed, treat every secret in it as leaked and rotate.

</details>

### Q5. The same configuration must create a VPC in `us-east-1` and a DR copy of a bucket in `eu-west-1`. How do you do it with one root module? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One root module, two regions: that's the textbook use for provider aliases.

**What I'd do →**
1. Define a default provider and an aliased one.
2. Point specific resources at the alias with `provider =`.

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "eu-west-1"
}

resource "aws_s3_bucket" "dr" {
  provider = aws.dr
  bucket   = "acme-logs-dr"
}
```

**Why →** Each provider configuration is bound to one region/account; aliases let one run manage both without separate stacks.

**Interview tip →** Mention that if modules need the alias, you pass it explicitly via the `providers` map.

</details>

### Q6. You must deploy the same stack into 12 AWS accounts. How do you handle credentials and provider configuration? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many accounts means one identity assuming a role per target account, not 12 sets of keys.

**What I'd do →**
1. Authenticate once (SSO or OIDC) into a tooling account.
2. Configure `assume_role` in the provider, with the account ID as a variable.
3. Run one state per account (separate backend key or workspace per account), driven by a pipeline matrix.

```hcl
provider "aws" {
  region = var.region
  assume_role {
    role_arn     = "arn:aws:iam::${var.account_id}:role/terraform-deploy"
    session_name = "tf-${var.account_id}"
  }
}
```

**Why →** Short-lived role sessions mean no long-lived keys, and per-account state keeps blast radius small.

**Interview tip →** Avoid `for_each` over providers in one state; provider aliases can't be dynamic in standard Terraform, so use one state per account.

</details>

### Q7. How do you make sure nobody runs your code with a Terraform version it wasn't tested against? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Different CLI versions can rewrite state in incompatible ways.

**What I'd do →**
1. Set `required_version` in the `terraform` block.
2. Pin the CLI in CI with `hashicorp/setup-terraform` or a `.terraform-version` file for `tfenv`/`mise`.

```hcl
terraform {
  required_version = "~> 1.9.0"
}
```

**Why →** Once a newer Terraform writes state, older versions refuse to read it. Pinning avoids a surprise "state was created by a newer version" for the rest of the team.

**Interview tip →** Upgrade the CLI as a deliberate change, one environment at a time.

</details>

### Q8. Developers use macOS laptops while CI runs on Linux, and `terraform init` in CI complains about missing checksums. What's wrong? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The lock file only has hashes for the platform that generated it.

**What I'd do →**
1. Regenerate hashes for every platform in use.
2. Commit the updated lock file.

```bash
terraform providers lock \
  -platform=linux_amd64 -platform=linux_arm64 \
  -platform=darwin_arm64 -platform=darwin_amd64
```

**Why →** `init` verifies downloaded provider archives against `h1:`/`zh:` hashes. A lock file generated on a Mac lacks the Linux archive hash, so verification fails or warns.

**Interview tip →** Add this to the contributing guide so provider bumps always include all platforms.

</details>

### Q9. The AWS provider has a new minor release you need. How do you upgrade without breaking production? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A provider upgrade can change defaults and cause diffs even with unchanged code.

**What I'd do →**
1. Read the provider CHANGELOG for the version range.
2. Bump the constraint in a branch and run `terraform init -upgrade`.
3. Run `terraform plan` in dev, then stage, expecting zero or understood diffs.
4. Merge the updated `.terraform.lock.hcl` and promote through environments.

**Why →** The plan is the safety net: an upgrade that shows unexpected replacements is stopped before any apply.

**Interview tip →** For a major version upgrade, do it alone in its own PR, never mixed with feature work.

</details>

### Q10. Before tearing down a short-lived test environment, how do you see exactly what will be destroyed? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Destroy is irreversible, so preview it.

**What I'd do →**
1. `terraform plan -destroy -out=destroy.tfplan`
2. Review the list; check for shared or `prevent_destroy` resources.
3. `terraform apply destroy.tfplan`

**Why →** `terraform destroy` is just a plan with destroy mode plus an apply. Running it as two steps gives a review point, and the same pattern works in CI.

**Interview tip →** Never use `-auto-approve` on destroy against anything resembling production.

</details>

### Q11. How would you design the plan/apply split in CI so the apply uses exactly what a reviewer approved? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Review must be tied to an immutable artifact.

**What I'd do →**
1. In the PR job: `terraform plan -out=tfplan`, then `terraform show -no-color tfplan` for the PR comment and `terraform show -json tfplan > plan.json` for policy checks.
2. Upload `tfplan` as an encrypted artifact.
3. After approval, a gated job downloads it and runs `terraform apply tfplan`.

**Why →** The apply refuses a plan whose base state serial has changed ("Saved plan is stale"), so a drifted world fails safe instead of applying something nobody reviewed.

**Interview tip →** Pin the same Terraform and provider versions in both jobs, since a plan file is version-specific.

</details>

### Q12. A scheduled job should alert when infrastructure differs from code, without changing anything. What do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want a read-only drift check with machine-readable results.

**What I'd do →**
1. Run `terraform plan -detailed-exitcode -lock=false -input=false` on a schedule with a read-only role.
2. Exit code `0` means no change, `1` means error, `2` means changes pending.
3. On `2`, post the plan summary to Slack or open a ticket.

```bash
terraform plan -detailed-exitcode -input=false -lock=false
case $? in 2) echo "DRIFT";; 1) echo "ERROR"; exit 1;; esac
```

**Why →** The exit code makes drift detection scriptable without parsing text.

**Interview tip →** Note that pending code changes also produce exit 2, so run drift jobs against the main branch only.

</details>

### Q13. You have dev, stage and prod using the same code but different sizes and counts. How do you feed per-environment values? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same code, different inputs.

**What I'd do →**
1. Keep one `variables.tf` with sensible defaults.
2. Create `envs/dev.tfvars`, `envs/stage.tfvars`, `envs/prod.tfvars`.
3. Run with `terraform plan -var-file=envs/prod.tfvars`, combined with a separate backend key per environment.

**Why →** Values are data, not code; the code path is identical across environments, which is what makes promotion trustworthy.

**Interview tip →** Pair the tfvars file with its own state, otherwise a `-var-file` mix-up can plan prod values into dev state.

</details>

### Q14. The EKS cluster and the Kubernetes/Helm providers are in the same root module, and first apply fails with "connection refused" on localhost. How do you fix the design? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Provider configuration depends on resource attributes that don't exist at plan time, so the Kubernetes provider falls back to defaults.

**What I'd do →**
1. Split into two stacks: `cluster` creates EKS, `cluster-addons` configures the Kubernetes/Helm providers from a data source.
2. In the second stack, read the cluster with `data "aws_eks_cluster"` and use an exec-based token.

```hcl
provider "kubernetes" {
  host                   = data.aws_eks_cluster.this.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.this.certificate_authority[0].data)
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", var.cluster_name]
  }
}
```

**Why →** Provider blocks are evaluated before resources are applied; splitting by lifecycle removes the chicken-and-egg.

**Interview tip →** Mention that deleting the cluster and its in-cluster resources in the same state is the other reason to split.

</details>

### Q15. How do you keep formatting and basic quality consistent across a team without arguing in code review? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Style and trivial errors should be automated, not reviewed.

**What I'd do →**
1. Add `pre-commit` with `terraform_fmt`, `terraform_validate`, `tflint` and `terraform_docs` hooks.
2. Repeat the same checks in CI as the enforcement point.
3. Use `terraform fmt -check -recursive` as a failing CI step.

**Why →** Local hooks give fast feedback; CI is the source of truth because hooks can be skipped.

**Interview tip →** TFLint's AWS ruleset catches invalid instance types and deprecated syntax that `validate` can't.

</details>

### Q16. A production network has no internet access. How do you make `terraform init` work? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Providers are downloaded from the registry by default, which an air-gapped runner can't reach.

**What I'd do →**
1. On a connected machine run `terraform providers mirror ./mirror`.
2. Host that directory internally (filesystem or HTTPS network mirror).
3. Point the CLI at it in `~/.terraformrc`.

```hcl
provider_installation {
  filesystem_mirror {
    path    = "/opt/terraform/providers"
    include = ["registry.terraform.io/*/*"]
  }
}
```

**Why →** A mirror gives reproducible, offline installs, and also protects you from a registry outage or a withdrawn release.

**Interview tip →** Mention that modules need a mirror too (internal Git or private registry), since `terraform init` fetches those separately.

</details>

### Q17. CI runs `terraform init` for 30 stacks and downloads the 400 MB AWS provider every time. How do you speed it up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repeated downloads waste time and bandwidth.

**What I'd do →**
1. Set `TF_PLUGIN_CACHE_DIR` to a persistent directory on the runner.
2. Cache that directory between jobs (for example `actions/cache` keyed on the lock file hash).

```bash
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
mkdir -p "$TF_PLUGIN_CACHE_DIR"
terraform init -input=false
```

**Why →** Terraform symlinks cached provider binaries instead of redownloading, so 30 inits share one copy.

**Interview tip →** The lock file still verifies checksums, so caching doesn't weaken integrity.

</details>

### Q18. How should an engineer authenticate Terraform to AWS from a laptop? Someone suggests hardcoding keys in the provider block. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hardcoded keys end up in Git and in state-adjacent files.

**What I'd do →**
1. Reject the hardcoded keys.
2. Use AWS IAM Identity Center: `aws sso login --profile dev`, then `export AWS_PROFILE=dev`.
3. Leave the provider block credential-free and let the default credential chain resolve identity.

```hcl
provider "aws" {
  region = "ap-south-1"
}
```

**Why →** Short-lived SSO credentials expire on their own and are tied to a person, which also helps CloudTrail attribution.

**Interview tip →** Say "the provider block should contain no secrets, ever."

</details>

### Q19. A plan on a 2,000-resource state takes 25 minutes. What do you look at? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Time goes to refreshing every resource via cloud APIs, often throttled.

**What I'd do →**
1. Confirm with `TF_LOG=DEBUG` or timing which provider calls dominate.
2. Short term: `terraform plan -refresh=false` for quick local iteration, and raise `-parallelism` carefully (default 10).
3. Medium term: split the state by lifecycle and ownership (network, data, apps).
4. Check for API throttling and tune provider `max_retries`.

**Why →** Plan time is roughly linear in state size, so the structural fix is smaller states, not flags.

**Interview tip →** `-target` is not a performance tool; it hides drift and leaves partial applies.

</details>

### Q20. The pipeline fails with "Saved plan is stale" after approval. Why did it happen and is it a bug? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Not a bug; it's the protection working.

**What I'd do →**
1. Check what changed state between plan and apply: another run, a manual `state` edit, or a refresh by a second pipeline.
2. Re-run the plan job on the current commit and get a new approval.
3. Prevent recurrence with a CI concurrency group so only one run per stack executes.

```yaml
concurrency:
  group: tf-prod-network
  cancel-in-progress: false
```

**Why →** A plan records the state serial and lineage it was built from. If state moved, applying it could undo someone else's change.

**Interview tip →** Short gaps between plan and apply reduce this; long-lived approvals make it worse.

</details>

### Q21. A resource needs a provider whose configuration comes from another resource's output in the same apply (for example a DB provider pointed at a new RDS endpoint). How do you handle it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same chicken-and-egg as Kubernetes, but for a database or Vault provider.

**What I'd do →**
1. Separate the stacks: stack A creates the RDS instance and outputs the endpoint.
2. Stack B reads it with `terraform_remote_state` or an SSM parameter and configures the provider.
3. For a one-off bootstrap, use two-phase apply with `-target` and document it as an exception.

**Why →** Provider configuration must be known when the provider initializes. Unknown values at that moment produce errors or silently wrong defaults.

**Interview tip →** "Providers configured from resource attributes are a code smell; split by lifecycle."

</details>

### Q22. When is `-target` acceptable, and when is it a red flag? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An engineer wants to apply one resource to "save time".

**What I'd do →**
1. Allow it for recovery, such as fixing a broken dependency or getting past an error to unblock the graph.
2. Follow every targeted apply with a full `terraform plan` that must converge to no changes.
3. Refuse it as a routine workflow or in pipelines.

```bash
terraform apply -target=aws_iam_role.deploy
terraform plan   # must be clean afterwards
```

**Why →** Terraform itself warns that targeting is for exceptional situations; it skips parts of the graph, so outputs and dependent resources can go stale.

**Interview tip →** Offer the real fix for slow plans: smaller states.

</details>

### Q23. You want to manage Datadog monitors with Terraform. How do you add a third-party provider correctly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Non-HashiCorp providers need an explicit source address.

**What I'd do →**
1. Declare it in `required_providers` with the namespace.
2. Configure credentials via environment variables.
3. Run `terraform init`.

```hcl
terraform {
  required_providers {
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.0"
    }
  }
}
provider "datadog" {}   # reads DD_API_KEY and DD_APP_KEY
```

**Why →** Since 0.13, providers are namespaced (`registry.terraform.io/DataDog/datadog`); without a `source`, Terraform assumes `hashicorp/`.

**Interview tip →** Mention keeping API keys in env vars or a secrets store, never in `.tf` files.

</details>

### Q24. After enabling `default_tags` on the AWS provider, every plan shows tag changes on Auto Scaling groups or some resources. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Some resources manage tags in a different way (inline `tag` blocks, or tags applied by another system), and `default_tags` overlaps with them.

**What I'd do →**
1. Look at the plan to see whether the diff is on `tags` or `tags_all`.
2. For ASGs, use `tag` blocks with `propagate_at_launch` or exclude the key from defaults.
3. Use the provider's `ignore_tags` for tags owned by other tooling.

```hcl
provider "aws" {
  default_tags { tags = { Team = "platform", ManagedBy = "terraform" } }
  ignore_tags  { key_prefixes = ["kubernetes.io/cluster/"] }
}
```

**Why →** Two writers of the same tag key produce a permanent diff; define one owner per key.

**Interview tip →** Same-key conflict between `default_tags` and resource `tags` can cause errors; keep keys disjoint.

</details>

### Q25. A new SaaS feature has no Terraform provider resource yet. What are your options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Providers lag behind APIs; you still want it in code.

**What I'd do →**
1. Check whether the newer `awscc` provider (Cloud Control API) already supports it.
2. For AWS, wrap a CloudFormation template with `aws_cloudformation_stack`.
3. As a last resort use `terraform_data` with `local-exec` calling the CLI/API, making it idempotent.
4. Open an issue or PR with the provider, and leave a TODO to migrate.

**Why →** You keep the lifecycle and dependency graph in Terraform while waiting for first-class support.

**Interview tip →** Explain that `local-exec` hacks don't track drift, so label them clearly.

</details>

## Resources Variables and Expressions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. You need three identical web servers. How do you declare them, and what is the catch with `count`? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Simple repetition, but index-based identity is fragile.

**What I'd do →**
1. Use `count = 3` and reference instances as `aws_instance.web[0]`.
2. Prefer `for_each` if instances need individual identities.

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.micro"
  tags          = { Name = "web-${count.index}" }
}
```

**Why →** With `count`, removing item `1` from a list shifts later indexes, so Terraform wants to destroy and recreate resources that didn't really change.

**Interview tip →** "Count for N identical things, for_each for named things."

</details>

### Q27. You manage subnets from a list of CIDRs. Someone removes the second CIDR and Terraform plans to recreate the third subnet. Why, and what do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `count` keyed by list index; the list shifted.

**What I'd do →**
1. Switch to `for_each` over a map keyed by a stable name.
2. Add `moved` blocks to avoid recreating existing subnets.

```hcl
variable "subnets" {
  type = map(string)   # { a = "10.0.1.0/24", b = "10.0.2.0/24" }
}
resource "aws_subnet" "this" {
  for_each   = var.subnets
  vpc_id     = aws_vpc.main.id
  cidr_block = each.value
}
```

**Why →** Keys are identities. Removing key `b` only affects `b`.

**Interview tip →** Choose keys that never change (names, not IPs).

</details>

### Q28. How do you stop someone passing `environment = "production"` when only `dev`, `stage` and `prod` are valid? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want to fail early with a clear message.

**What I'd do →** Add a `validation` block to the variable.

```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "stage", "prod"], var.environment)
    error_message = "environment must be one of: dev, stage, prod."
  }
}
```

**Why →** Validation runs at plan time before any API call, so bad input never reaches the cloud.

**Interview tip →** Since Terraform 1.9, validation conditions can reference other variables and data sources, not just the variable itself.

</details>

### Q29. A module takes a database password and exposes the connection string as an output. How do you stop them leaking into plan output? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Values are printed in plan/apply logs by default.

**What I'd do →**
1. Mark the variable `sensitive = true`.
2. Mark any output derived from it `sensitive = true` (Terraform will require it).
3. Read it explicitly with `terraform output -raw db_url` when needed.

**Why →** Sensitive values are redacted in the CLI output, but they are still stored in plain text in the state file. This is redaction, not encryption.

**Interview tip →** Always follow up with "so state itself must be encrypted and access-controlled".

</details>

### Q30. A monitoring stack should only exist in prod. How do you create a resource conditionally? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** On/off switch for a resource or module.

**What I'd do →**
1. Use `count` with a conditional.
2. Reference safely with `one()` or `try()`.

```hcl
resource "aws_cloudwatch_dashboard" "main" {
  count          = var.environment == "prod" ? 1 : 0
  dashboard_name = "app"
  dashboard_body = local.dashboard_json
}

output "dashboard_arn" {
  value = one(aws_cloudwatch_dashboard.main[*].dashboard_arn)
}
```

**Why →** `one()` returns `null` for an empty list and the element for a single item, which avoids `[0]` index errors when count is 0.

**Interview tip →** Prefer a boolean `enable_x` variable over environment-name checks inside modules.

</details>

### Q31. You have subnets defined as `{ web = {...}, db = {...} }` per VPC, nested two levels deep. How do you create them with `for_each`? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `for_each` needs a flat map, but data is nested.

**What I'd do →** Flatten with a `for` expression and build composite keys.

```hcl
locals {
  subnet_list = flatten([
    for vpc_name, vpc in var.vpcs : [
      for sn_name, sn in vpc.subnets : {
        key  = "${vpc_name}/${sn_name}"
        vpc  = vpc_name
        cidr = sn.cidr
      }
    ]
  ])
  subnet_map = { for s in local.subnet_list : s.key => s }
}

resource "aws_subnet" "this" {
  for_each   = local.subnet_map
  vpc_id     = aws_vpc.this[each.value.vpc].id
  cidr_block = each.value.cidr
}
```

**Why →** Composite string keys are stable, human-readable in state (`aws_subnet.this["prod/web"]`), and make diffs predictable.

**Interview tip →** Avoid keys that include values known only after apply.

</details>

### Q32. A security group needs 15 ingress rules read from a variable. How do you avoid 15 copy-pasted blocks? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repeated nested blocks: use `dynamic`.

**What I'd do →**

```hcl
resource "aws_security_group" "app" {
  name   = "app"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules   # map(object({port=number, cidrs=list(string)}))
    content {
      description = ingress.key
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidrs
    }
  }
}
```

**Why →** `dynamic` generates blocks from a collection. For larger rule sets, I prefer separate `aws_vpc_security_group_ingress_rule` resources so rules have their own state addresses.

**Interview tip →** Overusing `dynamic` hurts readability; use it only where the structure is genuinely variable.

</details>

### Q33. A teammate puts everything in `variables.tf`, including values computed from other variables. What's the right place for computed values? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Confusion between inputs and internal values.

**What I'd do →** Keep caller-supplied inputs in `variable` blocks and derived values in `locals`.

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = merge(var.tags, { Project = var.project })
}
```

**Why →** Variables form the module's public interface and can be overridden; locals are private, can't be set from outside, and remove repeated expressions.

**Interview tip →** A new reader should learn the interface from `variables.tf` and `outputs.tf` alone.

</details>

### Q34. Every resource needs a base set of tags plus its own. How do you combine them cleanly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tag drift and inconsistent tagging.

**What I'd do →**
1. Use provider `default_tags` for organisation-wide tags.
2. Use `merge()` for resource-specific additions.

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs"
  tags   = merge(local.common_tags, { DataClass = "internal" })
}
```

**Why →** `default_tags` applies uniformly with no per-resource code; `merge` handles the cases that need extras. Later maps override earlier keys.

**Interview tip →** Not every resource supports tags, so I validate coverage with a policy check rather than trusting convention.

</details>

### Q35. A script needs the load balancer DNS name after apply. How do you expose it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Outputs are the interface between Terraform and the outside world.

**What I'd do →**

```hcl
output "alb_dns_name" {
  value       = aws_lb.main.dns_name
  description = "Public DNS name of the ALB"
}
```

```bash
terraform output -raw alb_dns_name
terraform output -json | jq .
```

**Why →** `-raw` prints a bare string suitable for shell use; `-json` is for structured data.

**Interview tip →** Outputs from the root module are also what other stacks read through `terraform_remote_state`, so treat them as a contract.

</details>

### Q36. A YAML file lists 40 IAM users with groups. How do you create users and memberships without hand-writing 40 blocks? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data-driven infrastructure from a file.

**What I'd do →**

```hcl
locals {
  users = yamldecode(file("${path.module}/users.yaml"))   # list of {name, groups}
  memberships = flatten([
    for u in local.users : [for g in u.groups : { user = u.name, group = g }]
  ])
}

resource "aws_iam_user" "this" {
  for_each = { for u in local.users : u.name => u }
  name     = each.key
}

resource "aws_iam_user_group_membership" "this" {
  for_each = { for m in local.memberships : "${m.user}/${m.group}" => m }
  user     = aws_iam_user.this[each.value.user].name
  groups   = [each.value.group]
}
```

**Why →** Onboarding becomes a one-line YAML PR, reviewed and audited like code.

**Interview tip →** Validate the YAML with a variable `validation` or `precondition` so typos fail early.

</details>

### Q37. When should you add `depends_on`, and what can go wrong with it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Terraform infers most dependencies from references; explicit ones are for hidden dependencies.

**What I'd do →**
1. Use it only when the dependency isn't visible in an attribute reference, such as an IAM policy that must exist before an instance starts.
2. Avoid `depends_on` on modules or data sources unless essential.

```hcl
resource "aws_instance" "app" {
  # ...
  depends_on = [aws_iam_role_policy.app]
}
```

**Why →** `depends_on` on a data source defers its read to apply time, so values become unknown at plan time and downstream diffs get noisy. On modules it makes everything in the module wait.

**Interview tip →** Prefer expressing the dependency through a real reference.

</details>

### Q38. Every AMI refresh recreates the fleet, and you want a pinned and a "latest" option. How do you look up AMIs? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A `most_recent` data source changes results over time.

**What I'd do →**
1. Use a data source with explicit filters, or an SSM public parameter.
2. Offer an `ami_id` variable that overrides the lookup for prod.

```hcl
data "aws_ssm_parameter" "al2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}
resource "aws_instance" "app" {
  ami = coalesce(var.ami_id, data.aws_ssm_parameter.al2023.value)
}
```

**Why →** Dev follows latest, prod is pinned and promoted explicitly, so an AMI release never surprises a production plan.

**Interview tip →** Better still: bake AMIs with Packer and pass the ID in.

</details>

### Q39. User data scripts are long inline strings. How do you manage them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unreadable heredocs inside HCL.

**What I'd do →** Move the script into a template file and render it.

```hcl
resource "aws_instance" "app" {
  user_data = templatefile("${path.module}/templates/boot.sh.tftpl", {
    app_version = var.app_version
    region      = var.region
  })
  user_data_replace_on_change = true
}
```

**Why →** The script becomes lintable shell, and `user_data_replace_on_change` makes the change visible in the plan as a replacement.

**Interview tip →** Escape literal `${` as `$${` in template files.

</details>

### Q40. A module input has many optional settings with defaults. How do you type it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A flat list of 20 variables is painful; an untyped map is dangerous.

**What I'd do →** Use an `object` with `optional()` attributes and defaults.

```hcl
variable "database" {
  type = object({
    engine         = string
    instance_class = optional(string, "db.t4g.medium")
    multi_az       = optional(bool, false)
    backup_days    = optional(number, 7)
  })
}
```

**Why →** Callers set only what differs, typos are rejected at plan, and defaults are visible in one place.

**Interview tip →** `optional()` with a default has been stable since Terraform 1.3.

</details>

### Q41. Replacing a database should be forced when a config file changes, but the file is not an attribute of the resource. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need a trigger that doesn't map to a resource argument.

**What I'd do →**
1. Track the hash in a `terraform_data` resource.
2. Use `replace_triggered_by` to link it to the target.

```hcl
resource "terraform_data" "config_version" {
  input = filesha256("${path.module}/app.conf")
}
resource "aws_instance" "app" {
  ami           = var.ami
  instance_type = "t3.small"
  lifecycle {
    replace_triggered_by = [terraform_data.config_version]
  }
}
```

**Why →** `terraform_data` is the built-in replacement for `null_resource`, and `replace_triggered_by` (1.2+) expresses the intent declaratively.

**Interview tip →** Use it sparingly; every forced replacement is downtime for a stateful thing.

</details>

### Q42. You need to split a /16 into subnets across 3 AZs without hand-computing CIDRs. What function do you use? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Network math is error-prone by hand.

**What I'd do →**

```hcl
locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
  private_subnets = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i)]
  public_subnets  = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i + 100)]
}
```

Test it first with `terraform console`: `cidrsubnet("10.0.0.0/16", 8, 1)` gives `10.0.1.0/24`.

**Why →** `cidrsubnet(prefix, newbits, netnum)` is deterministic, so subnets never overlap and never change between runs.

**Interview tip →** Leave gaps in the numbering for future tiers.

</details>

### Q43. Which value wins when a variable is set in several places? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A teammate's `-var` is being silently overridden.

**What I'd do →** Recall the order, lowest to highest precedence:
1. Defaults in the variable block.
2. `TF_VAR_name` environment variables.
3. `terraform.tfvars`, then `terraform.tfvars.json`.
4. `*.auto.tfvars` files in alphabetical order.
5. `-var` and `-var-file` on the command line, in the order given.

**Why →** Later sources override earlier ones, and the CLI always wins. Knowing this explains "why isn't my tfvars value used?" in seconds.

**Interview tip →** Check `terraform console` and `var.name` to see the resolved value.

</details>

### Q44. A nested attribute may or may not exist, and `lookup` or index errors break the plan. How do you write resilient expressions? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Optional data causes "Invalid index" or "attribute not found" errors.

**What I'd do →**
1. Use `try()` for fallbacks, `can()` for validations, `coalesce()` for first non-null and `lookup()` for map keys.
2. Prefer typed optional object attributes over defensive expressions.

```hcl
locals {
  port     = try(var.app.port, 8080)
  log_days = lookup(var.retention, var.environment, 30)
  region   = coalesce(var.region_override, data.aws_region.current.name)
}
```

**Why →** These make intent explicit, but `try()` can mask real typos, so I limit it to genuinely optional values.

**Interview tip →** If every line has `try()`, the variable's type is too loose.

</details>

### Q45. Your AMI must be x86_64 and the plan should fail otherwise. How do you enforce that inside Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Catch wrong assumptions at plan/apply time rather than at boot.

**What I'd do →** Add a `precondition` or `postcondition`.

```hcl
data "aws_ami" "app" {
  owners = ["self"]
  filter {
    name   = "name"
    values = ["app-*"]
  }
  most_recent = true

  lifecycle {
    postcondition {
      condition     = self.architecture == "x86_64"
      error_message = "AMI ${self.id} is ${self.architecture}, expected x86_64."
    }
  }
}
```

**Why →** Conditions encode assumptions where the data is read, with a message that tells the operator exactly what is wrong.

**Interview tip →** Contrast with `check` blocks (1.5+), which warn but don't block.

</details>

### Q46. `for_each` fails because the map keys depend on a resource ID not known until apply. What's the fix? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `for_each` keys must be known at plan time.

**What I'd do →**
1. Key by static values such as names, not generated IDs.
2. Put the unknown part in the value instead.

```hcl
# bad:  for_each = toset(aws_subnet.this[*].id)
# good: keys are static names, IDs are values
for_each = { for k, v in aws_subnet.this : k => v.id }
```

**Why →** Terraform has to know how many instances exist and their addresses before applying; values can be unknown, keys cannot.

**Interview tip →** If you hit this, the root cause is usually using the wrong thing as the identity.

</details>

### Q47. You need to prove an expression works before editing a large config. What do you use? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You don't want to apply to learn what `cidrsubnet` or a `for` expression returns.

**What I'd do →** Use `terraform console`.

```bash
$ terraform console
> cidrsubnet("10.0.0.0/16", 8, 3)
"10.0.3.0/24"
> { for k, v in var.subnets : k => upper(v) }
> exit
```

**Why →** It evaluates expressions against your real variables and, with state, resource attributes, without changing anything.

**Interview tip →** Pipe expressions in non-interactively: `echo 'length(var.azs)' | terraform console`.

</details>

### Q48. Resource names must be unique globally, such as S3 buckets, across many environments and accounts. How do you generate them safely? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hardcoded names collide.

**What I'd do →**
1. Build names from project, environment and account ID.
2. Add a `random_id` suffix only where uniqueness can't be derived.

```hcl
data "aws_caller_identity" "current" {}
resource "aws_s3_bucket" "logs" {
  bucket = "${var.project}-${var.environment}-logs-${data.aws_caller_identity.current.account_id}"
}
```

**Why →** Derived names are predictable and discoverable, while random suffixes make things hard to find. Random values are stored in state, so they stay stable.

**Interview tip →** Use `name_prefix` for resources that must be replaced create-before-destroy.

</details>

### Q49. A list of resources was created with `count`, and now the team wants `for_each` with no downtime or recreation. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Changing the repetition mechanism changes state addresses from `[0]` to `["key"]`.

**What I'd do →**
1. Change the code to `for_each`.
2. Declare a `moved` block for each old index.
3. Run `terraform plan` and confirm it shows moves, not destroy/create.

```hcl
moved {
  from = aws_instance.web[0]
  to   = aws_instance.web["api"]
}
moved {
  from = aws_instance.web[1]
  to   = aws_instance.web["worker"]
}
```

**Why →** `moved` records the rename in code, so every environment performs the same state migration safely when it applies.

**Interview tip →** `terraform state mv` does the same but is manual and unreviewable.

</details>

### Q50. A reviewer asks why you use `locals` with `for` expressions instead of `count` and `element()`. How do you answer? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A style discussion about readability and safety.

**What I'd do →**
1. Show the `for` version with named keys.
2. Compare with `element(var.list, count.index)` patterns.

```hcl
locals {
  buckets = { for env in var.envs : env => "${var.project}-${env}" }
}
resource "aws_s3_bucket" "this" {
  for_each = local.buckets
  bucket   = each.value
}
```

**Why →** `for` expressions are explicit and keyed, so changes are local and plans are readable. Index arithmetic hides intent and breaks when order changes.

**Interview tip →** Prefer data shapes that match how people refer to things (names), not positions.

</details>

## Modules and Code Structure

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. When do you turn a block of resources into a module, and when do you leave it alone? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Modules add indirection; they should earn their keep.

**What I'd do →**
1. Create a module when the same pattern is used in 2–3 places, or when a team needs a safe, opinionated way to build something (VPC, ECS service, RDS).
2. Keep one-off resources in the root module.
3. Give the module a small, typed interface and meaningful outputs.

**Why →** A module with 40 pass-through variables is just a worse version of the resource. A good module hides complexity and enforces standards.

**Interview tip →** "A module should model a concept, not wrap a single resource."

</details>

### Q52. How do you consume a community module like `terraform-aws-modules/vpc/aws` responsibly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Public modules save weeks, but they are third-party code running with your credentials.

**What I'd do →**
1. Pin the version with a constraint.
2. Read the module source and its resources before adopting.
3. Review the plan on every version bump.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name    = "prod"
  cidr    = "10.0.0.0/16"
}
```

**Why →** Unpinned modules change under you. Pinning plus plan review treats module upgrades like any other dependency.

**Interview tip →** Many companies wrap popular modules in an internal one to enforce tags and encryption.

</details>

### Q53. An ECS service module needs the VPC and subnets created by a network module. How do you wire them together? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Modules communicate through outputs and inputs, never by reaching inside.

**What I'd do →**

```hcl
module "network" {
  source = "./modules/network"
  cidr   = "10.0.0.0/16"
}

module "service" {
  source     = "./modules/ecs-service"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
}
```

**Why →** Passing `module.network.*` outputs creates an implicit dependency, so ordering is automatic, and each module stays testable on its own.

**Interview tip →** If a module needs 10+ outputs from another, they probably belong together.

</details>

### Q54. How would you lay out a repo that deploys the same stack to dev, stage and prod? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need isolation between environments and shared logic.

**What I'd do →**
1. Put reusable code under `modules/`.
2. Create `envs/dev`, `envs/stage`, `envs/prod`, each a small root module that calls the shared modules with its own variables and backend key.
3. Version modules with Git tags so prod can lag dev.

```text
modules/network  modules/app  modules/database
envs/dev/main.tf  envs/dev/backend.tf  envs/dev/terraform.tfvars
envs/prod/main.tf envs/prod/backend.tf envs/prod/terraform.tfvars
```

**Why →** Separate directories mean separate state and a prod apply can never be triggered by editing dev.

**Interview tip →** I avoid workspaces for prod separation; directories make the target environment obvious.

</details>

### Q55. How do you release a shared module so consumers can upgrade at their own pace? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many stacks depend on one module, and a breaking change must not hit them all at once.

**What I'd do →**
1. Keep modules in their own repo (or path with tags) and release with SemVer tags: `v1.4.0`.
2. Consumers pin: `source = "git::https://github.com/acme/tf-modules.git//vpc?ref=v1.4.0"`.
3. Maintain a CHANGELOG and flag breaking changes with a major bump and migration notes (and `moved` blocks).

**Why →** Pinned versions mean consumers upgrade when ready, and `moved` blocks shipped inside the module make renames painless.

**Interview tip →** Branch references like `ref=main` are a production incident waiting to happen.

</details>

### Q56. A module needs to deploy to a second region. Why does putting a provider block inside the module cause trouble? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Provider blocks inside child modules make the module impossible to use with `for_each`/`count` and hard to remove.

**What I'd do →**
1. Remove provider blocks from the module.
2. Declare the expected provider names with `configuration_aliases`.
3. Pass providers from the root.

```hcl
# in module
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", configuration_aliases = [aws.replica] }
  }
}

# in root
module "bucket" {
  source    = "./modules/replicated-bucket"
  providers = { aws = aws, aws.replica = aws.dr }
}
```

**Why →** If you delete a module that contains its own provider, Terraform can't destroy its resources because the provider config vanished with it.

**Interview tip →** "Providers belong in the root."

</details>

### Q57. Teams complain a "VPC module" has 90 variables and still doesn't do what they need. How do you fix the design? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A God module that tries to cover every case.

**What I'd do →**
1. Split by concern: network core, NAT, endpoints, flow logs.
2. Offer opinionated defaults and a few feature flags.
3. Group optional settings in typed objects.
4. Add `examples/` showing the three common shapes.

**Why →** Smaller modules compose; each has a small interface, and consumers swap parts without forking.

**Interview tip →** If a module needs a `count` for every resource, it's trying to do too much.

</details>

### Q58. You must refactor loose resources into a module in a live prod state without recreating anything. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The resources already exist; only their addresses change.

**What I'd do →**
1. Create the module with the same resource definitions.
2. Add `moved` blocks from the root addresses to `module.x.…`.
3. Run `terraform plan`; it must show only "moved", no create/destroy.
4. Apply, and keep the `moved` blocks for a release or two.

```hcl
moved {
  from = aws_security_group.app
  to   = module.app.aws_security_group.this
}
```

**Why →** `moved` is declarative, reviewable and applies the same way in every environment.

**Interview tip →** Mention that plan output literally says "has moved to" for each, which is your proof.

</details>

### Q59. Where should you document modules so people actually use them correctly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Undocumented modules get copied and forked.

**What I'd do →**
1. Add a README per module with purpose, an example call and inputs/outputs.
2. Generate the tables with `terraform-docs` via pre-commit so they never go stale.
3. Include an `examples/basic` directory that CI actually plans.

```bash
terraform-docs markdown table --output-file README.md --output-mode inject ./modules/vpc
```

**Why →** Generated docs match the code; tested examples prove the README works.

**Interview tip →** Also write descriptions on every variable and output.

</details>

### Q60. Platform team wants each module tested before release. What's your approach? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Modules are shared dependencies and need a test gate.

**What I'd do →**
1. Static: `fmt`, `validate`, `tflint`, `checkov` on every PR.
2. Plan-level tests with `terraform test` using `command = plan` and mock providers.
3. Integration tests in a sandbox account for critical modules, applying and destroying.
4. Run the examples folder in CI.

**Why →** Cheap checks run on every commit; expensive real-cloud tests run before release.

**Interview tip →** Mention cleanup: tag test resources and run a sweeper so failed runs don't leak cost.

</details>

### Q61. A module should optionally create a KMS key, or use an existing one. How do you design that interface? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** "Create or bring your own" is a very common module requirement.

**What I'd do →**
1. Accept `kms_key_arn` (nullable) as input.
2. Create the key only if it is null.
3. Expose the effective ARN as a local.

```hcl
variable "kms_key_arn" {
  type    = string
  default = null
}
resource "aws_kms_key" "this" {
  count               = var.kms_key_arn == null ? 1 : 0
  enable_key_rotation = true
}
locals {
  kms_key_arn = coalesce(var.kms_key_arn, one(aws_kms_key.this[*].arn))
}
```

**Why →** The caller expresses intent with one optional value, and the rest of the module uses `local.kms_key_arn` with no branching.

**Interview tip →** The condition must depend on a value known at plan time, not on another resource's ID.

</details>

### Q62. A team wants to tweak a resource inside your module that you didn't expose. What are the options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pressure to fork or hack a shared module.

**What I'd do →**
1. Ask whether it's a legitimate, general need; if yes, add a variable and release a minor version.
2. If it's team-specific, create the extra resource outside the module using the module's outputs.
3. Do not use `override.tf` files or copy the module.

**Why →** Forks drift away from upstream and miss fixes; extending through outputs keeps one source of truth.

**Interview tip →** Good modules output IDs and ARNs of their key resources for exactly this reason.

</details>

### Q63. Why should the AWS provider be configured in the root module and not in a reusable child module? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A junior added `provider "aws"` with a hardcoded region inside a module.

**What I'd do →** Remove it; let the module inherit the root's default provider, and declare only `required_providers`.

```hcl
# module
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = ">= 5.0" }
  }
}
```

**Why →** Region and credentials are deployment decisions, not module decisions. Modules with embedded providers can't be removed cleanly and break reuse across regions.

**Interview tip →** In modules, set minimum provider versions with `>=`; the root pins the exact range.

</details>

### Q64. Compare Terragrunt with plain Terraform for a company with 30 stacks and 5 accounts. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The pain is repeated backend and provider boilerplate and ordering between stacks.

**What I'd do →**
1. Quantify what hurts today: copied `backend.tf`, `-var-file` wrappers, dependency ordering.
2. If it's mostly boilerplate, Terraform plus a generated backend via `-backend-config` and a small CI matrix is enough.
3. If dependency graphs across many stacks is the real pain, Terragrunt (`dependency`, `run-all`) pays off.

**Why →** Every extra tool is something to learn, upgrade and debug; adopt it for a measured problem.

**Interview tip →** Neither is wrong; show the trade-off, not a preference.

</details>

### Q65. A monorepo has 40 stacks. How do you structure layers and blast radius? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One giant state means slow plans and a huge blast radius.

**What I'd do →**
1. Layer by change frequency and ownership: `foundation` (accounts, IAM, DNS), `network`, `data`, `platform` (EKS), `apps`.
2. One state per layer per environment.
3. Pass values downward through SSM parameters or remote state outputs.
4. Gate each layer with CODEOWNERS and separate apply roles.

**Why →** Slow-changing, high-impact layers get strict review; fast-changing app layers deploy often and can't break the network.

**Interview tip →** Say how apps discover infrastructure: data sources and tags, not hardcoded IDs.

</details>

### Q66. Stack B needs the VPC ID created by stack A. Do you use `terraform_remote_state`, SSM parameters or a data source? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-stack data sharing, with different coupling levels.

**What I'd do →**
1. Prefer data sources (`aws_vpc` by tag) when the thing is discoverable.
2. Use SSM parameters or Secrets Manager as an explicit published contract between teams.
3. Use `terraform_remote_state` only inside one team where read access to the whole state is acceptable.

**Why →** `terraform_remote_state` exposes the entire state, including secrets, to the reader, and couples stacks to output names and the backend.

**Interview tip →** "Publish what you intend to share; don't share the whole state."

</details>

### Q67. Variables in a module have no defaults and callers keep forgetting some. How do you set required versus optional correctly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Interface hygiene.

**What I'd do →**
1. No default for values the caller must decide, such as `name` and `vpc_id`.
2. A safe default for everything else.
3. Add `description`, `type` and `validation` to each.

**Why →** A missing required variable fails fast with "No value for required variable", while a risky default silently deploys the wrong thing. Never default something like `publicly_accessible` to `true`.

**Interview tip →** Secure defaults, explicit opt-in for risk.

</details>

### Q68. You need one module instance per team, and adding or removing a team must not touch others. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repeated module instances keyed by a stable identity.

**What I'd do →** Use `for_each` on the module call with team names as keys.

```hcl
module "team_sandbox" {
  source   = "./modules/team-sandbox"
  for_each = var.teams            # map keyed by team name
  name     = each.key
  budget   = each.value.budget
}
```

**Why →** State addresses become `module.team_sandbox["payments"]...`, so removing one key destroys only that team's resources. Keys must be known at plan time.

**Interview tip →** `for_each` on modules requires providers passed from the root, not defined inside.

</details>

### Q69. Two modules require different version ranges of the same provider and `init` fails. What now? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Version constraints from all modules must intersect.

**What I'd do →**
1. Run `terraform providers` to see which module requires what.
2. Update or fork the module with the overly strict constraint (`~> 4.0` while others need `>= 5`).
3. In your own modules use `>=` lower bounds and let the root set the ceiling.

**Why →** Terraform installs one version per provider per root module, so module constraints must be compatible.

**Interview tip →** Strict upper bounds in shared modules are the usual culprit.

</details>

### Q70. How do you enforce encryption and tagging standards on every team's infrastructure? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standards should be built in, not remembered.

**What I'd do →**
1. Publish golden modules (encrypted S3, tagged RDS) as the default way to build.
2. Enforce with policy as code in CI (Checkov, OPA/Conftest, or Sentinel) on the plan JSON.
3. Add org guardrails (SCPs, AWS Config rules) as the backstop for anything outside Terraform.

**Why →** Modules make the right thing easy, policy makes the wrong thing hard, and SCPs catch the rest.

**Interview tip →** Mention both shift-left and runtime controls.

</details>

### Q71. Someone proposes `depends_on = [module.network]` on every other module "to be safe". Your take? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Over-constraining the graph.

**What I'd do →**
1. Remove blanket `depends_on`.
2. Pass real outputs (`module.network.vpc_id`) so dependencies are implicit and precise.
3. Keep explicit `depends_on` only for hidden dependencies, with a comment explaining why.

**Why →** Module-level `depends_on` makes every resource wait for every network resource, reduces parallelism and makes data sources inside defer to apply time.

**Interview tip →** Use references for ordering; use `depends_on` as a documented exception.

</details>

### Q72. You inherit a module nested five levels deep and nobody can follow it. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Over-abstraction.

**What I'd do →**
1. Map the real resources with `terraform state list` and `terraform graph`.
2. Flatten to at most two levels: root calling modules calling resources.
3. Use `moved` blocks while collapsing wrappers so state stays untouched.
4. Verify with `terraform plan` showing no changes.

**Why →** Deep wrappers pass variables through layers without adding value, and every layer is another place for a bug.

**Interview tip →** Refactor behind a no-diff plan; that's your safety net.

</details>

### Q73. Where do you put provider-agnostic helper logic, such as naming and tagging conventions, shared by all stacks? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Naming logic is copy-pasted in every stack.

**What I'd do →** Create a tiny "context" or "naming" module with no resources, just locals and outputs.

```hcl
module "ctx" {
  source      = "git::https://github.com/acme/tf-modules.git//context?ref=v1.2.0"
  project     = "payments"
  environment = var.environment
}
resource "aws_s3_bucket" "x" {
  bucket = "${module.ctx.prefix}-reports"
  tags   = module.ctx.tags
}
```

**Why →** One definition of "how we name things" means a convention change is a single release.

**Interview tip →** Keep it dumb and stable; it's imported everywhere.

</details>

### Q74. A new engineer asks why modules have `outputs.tf` full of IDs they never use. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** They see unused outputs as clutter.

**What I'd do →** Explain that outputs are the public interface of the module.

1. Other modules and stacks consume them (`module.vpc.vpc_id`).
2. Outputs are the only way to reach values inside a child module.
3. Keep them: removing one is a breaking change for consumers.

**Why →** Child module internals are not accessible from the caller; outputs define what is supported.

**Interview tip →** Output IDs and ARNs of key resources, with descriptions.

</details>

### Q75. When would you choose one root module with many modules over many small root modules (stacks)? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Granularity of state is the real decision.

**What I'd do →**
1. Single root when components are tightly coupled, deployed together and owned by one team.
2. Separate roots when lifecycles, owners or blast radius differ, such as network vs app.
3. Between roots, share through published outputs, not shared state files.

**Why →** One state gives atomic changes but slow plans and a large blast radius; many states give independence but need orchestration and contracts.

**Interview tip →** Frame it as lifecycle and ownership boundaries, not a file count.

</details>

## State Backends and Workspaces

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. Two engineers keep overwriting each other's changes because state is local. What's the fix? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Local `terraform.tfstate` on laptops means each person has a different truth.

**What I'd do →**
1. Move state to a remote backend (S3, Azure Blob, GCS, or HCP Terraform).
2. Enable locking so only one run mutates state at a time.
3. Migrate with `terraform init -migrate-state`.

**Why →** Remote state gives a single source of truth, access control, versioning and locking.

**Interview tip →** Say "state is shared, sensitive and must be locked".

</details>

### Q77. Set up a production-grade S3 backend. What do you configure for locking? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** S3 backend needs encryption, versioning and a lock mechanism.

**What I'd do →**

```hcl
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-prod"
    key          = "network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true   # native S3 locking, Terraform 1.10+
  }
}
```

**Why →** `use_lockfile` writes a `.tflock` object next to the state using S3 conditional writes, so no DynamoDB table is needed. The older `dynamodb_table` argument still works on older versions but is deprecated.

**Interview tip →** Mention both approaches and say which Terraform version you're on.

</details>

### Q78. How do you harden the S3 bucket that holds Terraform state? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** State holds secrets and is the keys to the kingdom.

**What I'd do →**
1. Turn on versioning so you can roll back a bad write.
2. Enable default SSE-KMS encryption.
3. Block all public access.
4. Add a bucket policy that denies non-TLS requests and restricts access to the CI role and a break-glass role.
5. Enable access logging or CloudTrail data events.

**Why →** Versioning is your undo; IAM scoping and encryption protect the secrets inside.

**Interview tip →** Put the bucket in a dedicated tooling account.

</details>

### Q79. Who creates the state bucket itself, given Terraform needs it to exist before `init`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bootstrap problem.

**What I'd do →**
1. Create the bucket once with a tiny bootstrap stack that uses local state, or with the CLI.
2. Then optionally migrate that stack's state into the new bucket.
3. Keep the bootstrap code in Git with a README.

```bash
aws s3api create-bucket --bucket acme-tfstate-prod --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
aws s3api put-bucket-versioning --bucket acme-tfstate-prod \
  --versioning-configuration Status=Enabled
```

**Why →** There's always one thing created outside the system; make it tiny, documented and rarely touched.

**Interview tip →** Account vending pipelines usually create it automatically.

</details>

### Q80. A laptop apply and a CI apply start at the same time. What stops state corruption? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Concurrent writers are the classic cause of state corruption.

**What I'd do →**
1. Rely on backend locking: the second run fails with "Error acquiring the state lock".
2. Don't use `-lock=false` for applies.
3. Restrict who can apply prod to the CI role only.

**Why →** The lock holds an ID and owner info; the second run waits (`-lock-timeout=5m`) or fails instead of interleaving writes.

**Interview tip →** The cleanest control is making CI the only identity allowed to write prod state.

</details>

### Q81. When are Terraform workspaces appropriate? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A teammate wants to use workspaces for dev/stage/prod.

**What I'd do →**
1. Use CLI workspaces for short-lived copies of the same stack, such as a feature-branch environment or PR preview.
2. Use separate directories or backends for long-lived environments with different access rules.

```bash
terraform workspace new pr-421
terraform apply -var="name_suffix=pr-421"
terraform workspace select default
```

**Why →** CLI workspaces share the same backend and code, so credentials and blast radius are shared, and it's easy to apply to the wrong one.

**Interview tip →** HCP Terraform "workspaces" are a different thing: independent state, variables and permissions.

</details>

### Q82. Compare directories, workspaces and branches for environment separation. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A design-review question.

**What I'd do →**
1. Directories per environment: explicit, separate state and backend, best for prod isolation, some duplication.
2. CLI workspaces: minimal duplication, same backend and credentials, easy mistakes.
3. Branch per environment: drift between branches and painful merges; avoid.

**Why →** Production safety comes from separate credentials and state, and directories give you both.

**Interview tip →** Recommend directories plus shared modules, with workspaces only for ephemeral copies.

</details>

### Q83. Move from local state to S3 for an existing project without losing anything. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Existing resources, existing local state.

**What I'd do →**
1. Back up `terraform.tfstate`.
2. Add the `backend "s3"` block.
3. Run `terraform init -migrate-state` and answer `yes` to copy.
4. Run `terraform plan`; it should be clean.
5. Delete the local state file after verifying the remote object.

**Why →** `-migrate-state` copies state with its lineage and serial, so nothing is re-imported.

**Interview tip →** Check the plan is empty before telling anyone it worked.

</details>

### Q84. A giant state with 1,500 resources must be split into network and app states. How do you do it safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You must move resources between states without destroying anything.

**What I'd do →**
1. Back up the state (`terraform state pull > backup.tfstate`).
2. Write the new root module, with the same resource code.
3. Use `terraform state mv -state-out=new.tfstate ...` for each resource, or `import` blocks in the new stack and `removed` blocks in the old one.
4. Push the new state to the new backend and run `terraform plan` in both: both must be clean.
5. Do it in a maintenance window with locks held.

**Why →** Each step is reversible via the backup, and clean plans prove the split is correct.

**Interview tip →** Prefer import and removed blocks for reviewability; `state mv` is imperative.

</details>

### Q85. How do you inspect what Terraform thinks it manages? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want to see what's in state.

**What I'd do →**
```bash
terraform state list
terraform state show aws_instance.web
terraform show -json | jq '.values.root_module.resources[].address'
terraform state pull > snapshot.json
```

**Why →** `state list` gives addresses, `state show` gives attributes, and `pull` writes the raw JSON for offline inspection.

**Interview tip →** State is JSON but never edit it by hand; use `state mv`, `rm` or `import`.

</details>

### Q86. You want Terraform to stop managing a resource but leave it running. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Handing off ownership.

**What I'd do →**
1. Use a `removed` block (Terraform 1.7+) with `destroy = false`, then delete the resource block.
2. Run `terraform apply`.

```hcl
removed {
  from = aws_s3_bucket.legacy
  lifecycle {
    destroy = false
  }
}
```

**Why →** It's declarative and shows up in review, versus `terraform state rm aws_s3_bucket.legacy` which is an unreviewed manual edit.

**Interview tip →** Whichever you use, plan afterwards to confirm nothing will be destroyed.

</details>

### Q87. The state file in S3 is deleted by mistake. How do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Infrastructure is fine, Terraform has amnesia.

**What I'd do →**
1. With versioning on, restore the previous version: list with `aws s3api list-object-versions`, then copy the last good version back.
2. Without versions, rebuild state with `import` blocks for each resource, and check plan.
3. Don't run `apply` with an empty state; it would try to create duplicates.

**Why →** Versioning turns a disaster into a one-minute fix; without it, recovery is slow and error-prone.

**Interview tip →** Add MFA delete or object lock for prod state buckets.

</details>

### Q88. Give a pipeline in Account A access to state in a bucket in Account B. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-account backend access.

**What I'd do →**
1. Create an IAM role in the state account with `s3:GetObject`, `PutObject`, `DeleteObject` (for the lock file), `ListBucket` on the prefix.
2. Allow the pipeline role to assume it.
3. Configure the backend with `assume_role`.

```hcl
backend "s3" {
  bucket = "acme-tfstate"
  key    = "app/terraform.tfstate"
  region = "ap-south-1"
  assume_role = { role_arn = "arn:aws:iam::111111111111:role/tf-state-access" }
}
```

**Why →** The backend's identity is separate from the provider's, which keeps state access centralised and auditable.

**Interview tip →** Scope the role to the key prefix per stack.

</details>

### Q89. Security asks, "who can see our secrets in Terraform state?" What do you answer? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** State stores resource attributes in plain text, including DB passwords and private keys.

**What I'd do →**
1. Treat state read access as equal to secret access and restrict it to CI and a few admins.
2. Encrypt at rest with KMS and enforce TLS in transit.
3. Reduce secrets in state: RDS `manage_master_user_password`, ephemeral values and write-only arguments on Terraform 1.10/1.11+, secrets fetched at runtime by apps.
4. Audit access with CloudTrail.

**Why →** `sensitive = true` only hides values in CLI output; it never removes them from state.

**Interview tip →** This question tests whether you know "sensitive" is redaction, not protection.

</details>

### Q90. How do you rename a resource in code, from `aws_instance.web` to `aws_instance.app`, without replacement? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A rename without a state change looks like destroy plus create.

**What I'd do →** Add a `moved` block next to the renamed resource.

```hcl
moved {
  from = aws_instance.web
  to   = aws_instance.app
}
```

Plan should say `aws_instance.web has moved to aws_instance.app`.

**Why →** The move is tracked in code, so every environment migrates identically on its next apply.

**Interview tip →** `terraform state mv` achieves the same for one state but isn't reviewable.

</details>

### Q91. What is `terraform refresh` and what replaced it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** An old doc says to run `refresh` before plan.

**What I'd do →**
1. Know that plan and apply already refresh state by default.
2. Use `terraform plan -refresh-only` to review drift and `terraform apply -refresh-only` to accept it into state.
3. Avoid the standalone `terraform refresh`, which writes state without showing a diff.

**Why →** Refresh-only mode adds a review step, whereas the bare command can silently write bad data into state, for example after a credentials mistake.

**Interview tip →** "refresh-only mode is the safe version".

</details>

### Q92. You have dev, stage and prod backends that differ only by bucket and key. How do you avoid copy-pasting `backend.tf`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Backend blocks can't use variables.

**What I'd do →** Use partial configuration: leave the backend block mostly empty and pass the rest at init.

```hcl
terraform {
  backend "s3" {}
}
```
```bash
terraform init -reconfigure -backend-config=envs/prod.backend.hcl
```
```hcl
# envs/prod.backend.hcl
bucket = "acme-tfstate-prod"
key    = "app/terraform.tfstate"
region = "ap-south-1"
```

**Why →** One code path, per-environment backend config files that are easy to review.

**Interview tip →** Use `-reconfigure` when switching between environments locally, and watch for the wrong-file mistake.

</details>

### Q93. A CI job crashed and the lock remains. When is `force-unlock` acceptable? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A stale lock from a killed runner blocks everyone.

**What I'd do →**
1. Confirm via the lock info (ID, who, when) and CI that no run is active.
2. Check the state isn't partially written: compare the latest S3 version and look at logs.
3. Run `terraform force-unlock <LOCK_ID>`.
4. Run `plan` to verify state and reality agree.

**Why →** Unlocking while a real apply is running can corrupt state. Verify, then unlock, then re-plan.

**Interview tip →** With `use_lockfile`, the lock is an S3 object, which you can also inspect and delete.

</details>

### Q94. Many teams read each other's outputs via `terraform_remote_state`. What problems appear, and how do you fix them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hidden coupling.

**What I'd do →**
1. List who reads what; ensure readers use a read-only role.
2. Treat outputs as a versioned contract and don't rename without a deprecation window.
3. Move high-value shared values to SSM or a service catalog, with explicit ownership.

**Why →** Anyone with remote_state access can read the entire state, including secrets, and renames silently break downstream plans.

**Interview tip →** Mention "contract" and "least privilege".

</details>

### Q95. State is 40 MB and operations are slow. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Large state means slow refresh, slow locking and a large blast radius.

**What I'd do →**
1. Identify heavy resource types with `terraform state list | cut -d. -f1-2 | sort | uniq -c | sort -rn`.
2. Split by lifecycle into multiple states.
3. Remove noisy resources such as thousands of IAM policy attachments from a hot path by moving them to their own stack.
4. Use `-refresh=false` only for local iteration.

**Why →** Plan time scales with resource count; smaller states are faster and safer.

**Interview tip →** Splitting is a design fix, flags are a patch.

</details>

### Q96. How do you protect state for disaster recovery? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** State is a critical asset; regional loss shouldn't block recovery.

**What I'd do →**
1. Enable versioning, and optionally Object Lock for compliance.
2. Replicate the bucket cross-region (CRR) to a DR bucket in a separate account.
3. Practice a restore, including pointing `-backend-config` at the DR bucket.
4. Keep bootstrap code and runbooks outside the same region and account.

**Why →** Versioning covers mistakes, replication covers regional failure, and rehearsal proves it works.

**Interview tip →** You can't fix prod with Terraform in an outage if the state bucket is in the failing region.

</details>

### Q97. Should you use HCP Terraform or an S3 backend for state? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A build-versus-buy decision for a small team.

**What I'd do →**
1. S3 (or GCS, Azure Blob): cheap, simple, and you own the pipeline.
2. HCP Terraform: remote runs, state locking, RBAC, policy, private registry and SSO out of the box, at a per-resource price.
3. Pick HCP if you'd otherwise build runs, approvals and policy yourself.

**Why →** It's about operational effort and governance needs rather than technical capability.

**Interview tip →** Also mention OpenTofu with an S3 backend as a licence-free path.

</details>

### Q98. In HCP Terraform, how do you manage 40 workspaces that need the same AWS credentials and tags? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repeated variables across workspaces.

**What I'd do →**
1. Create a variable set with the shared environment variables (preferably dynamic provider credentials via OIDC, not static keys).
2. Apply the set to a project or to all workspaces.
3. Keep workspace-specific values on the workspace.

**Why →** One place to rotate or change, and no keys copied by hand.

**Interview tip →** Dynamic credentials give per-run short-lived tokens.

</details>

### Q99. Two repos accidentally manage the same S3 bucket. What happens, and how do you resolve it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dual ownership: each state thinks it owns the resource, and they fight.

**What I'd do →**
1. Decide the owner, ideally the team that owns the lifecycle.
2. In the other repo, add a `removed` block with `destroy = false` (or `state rm`) and apply.
3. Consumers of the bucket use a data source instead.
4. Add CODEOWNERS or tagging (`ManagedBy`, `Stack`) to make ownership visible.

**Why →** Two states reconciling one resource produce perpetual diffs, and a destroy in either repo deletes it for both.

**Interview tip →** Ownership should be answerable from a tag.

</details>

### Q100. What is the difference between `terraform state rm`, `removed` and `terraform destroy -target` for getting a resource out of your config? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Three tools with very different outcomes.

**What I'd do →**
1. `state rm` and `removed { destroy = false }`: forget the resource; the real one stays.
2. Deleting the resource block normally: destroys the real one on next apply.
3. `destroy -target`: destroys now, outside normal flow; use only in recovery.

**Why →** Pick by intent: forget, delete via code, or emergency delete.

**Interview tip →** Prefer the code-based options so there's a commit trail.

</details>

## Import Refactoring Drift and Lifecycle

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. An S3 bucket was created by hand in the console and now must be managed by Terraform. How do you bring it in? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Existing resource, no code, no state entry.

**What I'd do →**
1. Write the resource block that matches the bucket.
2. Add an `import` block (Terraform 1.5+).
3. Run `terraform plan`, adjust the code until the diff is empty, then apply.

```hcl
import {
  to = aws_s3_bucket.assets
  id = "acme-assets-prod"
}
resource "aws_s3_bucket" "assets" {
  bucket = "acme-assets-prod"
}
```

**Why →** Import blocks are reviewable and plannable, unlike the one-shot `terraform import` command.

**Interview tip →** Import only writes state; you must write the code that matches.

</details>

### Q102. You must adopt 200 existing resources and don't want to write all the HCL by hand. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Large brownfield adoption.

**What I'd do →**
1. List the resources by service and tag; script the `import` blocks (ID discovery with the AWS CLI).
2. Run `terraform plan -generate-config-out=generated.tf` to draft the HCL.
3. Clean the generated code: drop computed attributes, replace hardcoded IDs with references, extract variables.
4. Iterate until `plan` shows no changes, then refactor into modules.

**Why →** Generated config is a starting point and is often noisy; the empty plan is the acceptance test.

**Interview tip →** Do it in small batches grouped by dependency, not all 200 at once.

</details>

### Q103. How do you find out that someone changed infrastructure outside Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Console hotfix; code and reality have diverged.

**What I'd do →**
1. Run `terraform plan`; drifted attributes appear as changes to revert.
2. Use `terraform plan -refresh-only` to see only drift with no code changes mixed in.
3. Automate a scheduled plan that alerts on exit code 2.

**Why →** Refresh reads real attributes from the cloud API and compares with state and code.

**Interview tip →** Drift detection is only useful if somebody owns the alert.

</details>

### Q104. Someone opened port 22 on a security group in the console. Terraform wants to remove it. What's your process? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Revert or codify? Find out why it happened first.

**What I'd do →**
1. Ask who changed it and why; check CloudTrail (`AuthorizeSecurityGroupIngress`).
2. If it was an unauthorised or temporary change, run `terraform apply` to revert.
3. If it's a legitimate need, add it to code through a PR, then apply.

**Why →** Terraform's job is to enforce the desired state, but humans decide what that state should be.

**Interview tip →** Mention follow-up: restrict console write access so drift becomes rare.

</details>

### Q105. An Auto Scaling group's desired capacity is changed by autoscaling, and Terraform keeps resetting it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Another system legitimately owns this attribute.

**What I'd do →** Tell Terraform to ignore it.

```hcl
resource "aws_autoscaling_group" "web" {
  min_size         = 2
  max_size         = 20
  desired_capacity = 2
  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

**Why →** `ignore_changes` removes that attribute from diff computation; the initial value is still set at creation.

**Interview tip →** Use it for attributes owned by other controllers (HPA, ASG, image pipelines), not to hide mistakes.

</details>

### Q106. How do you protect a production database from an accidental `terraform destroy`? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A plan that deletes the DB must fail loudly.

**What I'd do →**
1. Add `prevent_destroy = true`.
2. Also set `deletion_protection = true` on the resource itself.
3. Enable final snapshots (`skip_final_snapshot = false`).

```hcl
resource "aws_db_instance" "main" {
  deletion_protection       = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "main-final"
  lifecycle { prevent_destroy = true }
}
```

**Why →** `prevent_destroy` stops Terraform; `deletion_protection` stops the API even if someone bypasses Terraform.

**Interview tip →** `prevent_destroy` only works while the block is in code; deleting the block deletes the protection.

</details>

### Q107. A load balancer's TLS certificate must be rotated with no outage. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The default order is destroy then create, which would break the listener.

**What I'd do →** Use `create_before_destroy`.

```hcl
resource "aws_acm_certificate" "app" {
  domain_name       = "app.example.com"
  validation_method = "DNS"
  lifecycle { create_before_destroy = true }
}
```

**Why →** The new cert exists and is attached before the old one is removed, so the listener always has a valid certificate.

**Interview tip →** `create_before_destroy` propagates to dependencies, and names must not collide.

</details>

### Q108. Using `create_before_destroy` fails because the new resource's name conflicts with the old one. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both exist briefly and share a unique name.

**What I'd do →**
1. Use `name_prefix` so Terraform generates a unique suffix.
2. Where the resource has no prefix option, include a changing component such as a random or version suffix.

```hcl
resource "aws_security_group" "app" {
  name_prefix = "app-"
  vpc_id      = var.vpc_id
  lifecycle { create_before_destroy = true }
}
```

**Why →** Names must be unique while both versions coexist; a prefix makes that automatic.

**Interview tip →** Applies to launch templates, security groups and IAM roles.

</details>

### Q109. The plan says an RDS instance "must be replaced" after a small code change. What do you do before approving? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Some attribute changes force replacement, which for a database means data loss.

**What I'd do →**
1. Find the line marked `# forces replacement` in the plan.
2. Stop. Take a manual snapshot.
3. Check whether the change can be made in place (`modify` vs recreate), such as a parameter group change or `apply_immediately`.
4. If replacement is unavoidable, restore from snapshot through `snapshot_identifier` or plan a migration.

**Why →** The plan tells you which attribute is immutable; the fix is a design change, not a flag.

**Interview tip →** Recognising "forces replacement" in plans is a core skill.

</details>

### Q110. A resource was deleted from the console. What does Terraform do on the next plan? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** State says it exists, the cloud says it doesn't.

**What I'd do →**
1. Run `terraform plan`; refresh notices it's gone and plans to create it again.
2. If the deletion was intended, remove the code too.
3. Otherwise apply to recreate.

**Why →** Terraform drives toward the code; missing means "create".

**Interview tip →** For data-bearing resources recreation does not restore data; use backups.

</details>

### Q111. An instance is unhealthy and needs rebuilding. How do you force replacement of one resource? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want a recreate without changing code.

**What I'd do →**
```bash
terraform apply -replace="aws_instance.web[1]"
```

**Why →** `-replace` supersedes the older `terraform taint`: it appears in the plan, so reviewers can see it, and nothing is left tainted in state if the run is cancelled.

**Interview tip →** Prefer `-replace` to `taint`; the docs deprecate taint.

</details>

### Q112. A new AMI release makes Terraform want to replace every EC2 instance in prod. How do you control this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `most_recent` AMI lookups change the AMI attribute, which forces replacement.

**What I'd do →**
1. Pin the AMI ID per environment via a variable.
2. Promote new AMIs through dev and stage first.
3. For fleets, use a launch template plus an ASG with `instance_refresh` so rollout is rolling instead of all-at-once.

**Why →** You choose when a new image reaches prod, and rollouts replace instances gradually.

**Interview tip →** `ignore_changes = [ami]` hides the problem; use it only where an external pipeline owns the AMI.

</details>

### Q113. How do you roll out a new launch template to an ASG without downtime? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A launch template change doesn't touch running instances by default.

**What I'd do →** Enable instance refresh and reference the latest template version.

```hcl
resource "aws_autoscaling_group" "web" {
  launch_template {
    id      = aws_launch_template.web.id
    version = aws_launch_template.web.latest_version
  }
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 90
      instance_warmup        = 120
    }
  }
}
```

**Why →** Terraform detects the new version and the ASG replaces instances in batches while keeping capacity.

**Interview tip →** Combine with health checks and a target group so bad AMIs are caught and the refresh stops.

</details>

### Q114. Import 30 similar resources with different IDs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many imports of the same type.

**What I'd do →** Use `for_each` in the import block (Terraform 1.7+).

```hcl
locals {
  buckets = { logs = "acme-logs", assets = "acme-assets" }
}
import {
  for_each = local.buckets
  to       = aws_s3_bucket.this[each.key]
  id       = each.value
}
resource "aws_s3_bucket" "this" {
  for_each = local.buckets
  bucket   = each.value
}
```

**Why →** One block replaces dozens, and the plan shows exactly what will be imported.

**Interview tip →** Remove import blocks after the first successful apply, or leave them; they're no-ops once imported.

</details>

### Q115. An IAM role with five policy attachments is imported, and the plan wants to change policies. What did you miss? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Importing the role does not import its attachments or inline policies.

**What I'd do →**
1. Import each attachment separately (`aws_iam_role_policy_attachment` uses `role-name/policy-arn`).
2. Use exclusive resources deliberately: `aws_iam_role_policy_attachments_exclusive` if Terraform should be the only manager.
3. Re-plan until clean.

**Why →** In the AWS provider each attachment is its own resource; mixing inline `managed_policy_arns` and attachments produces fights.

**Interview tip →** Know whether a resource is "authoritative" (manages the whole set) or "additive".

</details>

### Q116. A CloudFormation stack owns resources that you want in Terraform. How do you migrate? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two tools claim the same resources.

**What I'd do →**
1. Update the CFN template to set `DeletionPolicy: Retain` on every resource and deploy it.
2. Import the resources into Terraform and get a clean plan.
3. Delete the stack; resources stay because of Retain.

**Why →** Retain makes stack deletion non-destructive, so ownership moves safely.

**Interview tip →** Do it in order; deleting the stack first would delete the infrastructure.

</details>

### Q117. How do you accept a harmless drift into state without changing infrastructure? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The cloud changed an attribute you're happy with, such as a tag added by another system.

**What I'd do →**
1. `terraform plan -refresh-only` to review what changed.
2. `terraform apply -refresh-only` to record it in state.
3. Update the code to match so the next normal plan is clean.

**Why →** Refresh-only changes state only, never infrastructure.

**Interview tip →** Without step 3 the next plan will try to revert it.

</details>

### Q118. You need to move a resource from one module call to another within the same state. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Address change across module boundaries.

**What I'd do →** Use a `moved` block with full addresses.

```hcl
moved {
  from = module.old_app.aws_lb.this
  to   = module.new_app.aws_lb.this
}
```

For moves between different states, `moved` doesn't work; use `import` in the target and `removed { destroy = false }` in the source.

**Why →** `moved` only rewrites addresses in the same state. Cross-state moves require forgetting in one and importing in the other.

**Interview tip →** Apply the import side first, then the removal, to avoid an unmanaged gap.

</details>

### Q119. A teammate runs `terraform state mv` ad hoc in prod. What do you recommend instead? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Imperative state surgery with no review.

**What I'd do →**
1. Replace it with a `moved` block committed in a PR.
2. Keep `state mv` for cross-state moves or emergencies, with a state backup first.

**Why →** `moved` blocks apply the same migration in every environment and leave a history in Git.

**Interview tip →** Always `terraform state pull > backup.json` before any manual state command.

</details>

### Q120. How would you run drift detection across 80 workspaces? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One-off checks don't scale.

**What I'd do →**
1. In HCP Terraform, enable health assessments (drift detection) on workspaces; or in CI run a nightly matrix of `plan -detailed-exitcode -refresh-only` with a read-only role.
2. Aggregate results: Slack alerts for drift, tickets with an owner by workspace tag.
3. Track a drift metric over time.
4. Decide a policy: auto-revert for non-prod, ticket for prod.

**Why →** The goal is mean time to detect drift, not just detection.

**Interview tip →** Say who acts on the alert.

</details>

### Q121. AWS adds its own tags to resources, producing constant tag diffs. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tags starting with `aws:` or added by services (for example `aws:cloudformation:*`) differ from your config.

**What I'd do →**
1. Check the plan for which keys differ.
2. Use provider `ignore_tags` with key prefixes for system tags.
3. Or `lifecycle { ignore_changes = [tags["Owner"]] }` for a specific key.

**Why →** You exclude tags you don't own while still managing yours.

**Interview tip →** Avoid blanket `ignore_changes = [tags]`; it stops managing your own tags too.

</details>

### Q122. One change to a VPC output makes the plan replace 30 resources. How do you find out why, and reduce the damage? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A cascading replacement through dependencies.

**What I'd do →**
1. Read the plan top-down: find the first resource marked `forces replacement`.
2. Trace the attribute that forced it with `terraform graph | dot -Tsvg`.
3. Revert or restructure: avoid changing immutable inputs, or use `create_before_destroy`.
4. Do not apply until the root cause is understood.

**Why →** Replacing a parent (subnet, VPC, role) forces replacement of everything that has it as an immutable argument.

**Interview tip →** A blast-radius estimate in the PR description shows maturity.

</details>

### Q123. When is `ignore_changes = all` appropriate? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A teammate wants it to silence a noisy plan.

**What I'd do →**
1. Refuse as a default; list the specific attributes instead.
2. Accept it only for resources that are intentionally "create once, then someone else manages it", with a comment explaining who.

**Why →** `all` turns Terraform off for that resource: drift, upgrades and security fixes never apply.

**Interview tip →** If you need `all`, ask if Terraform should manage it at all.

</details>

### Q124. How do you test a refactor so that you're sure nothing will be destroyed? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A refactor should have a no-op plan.

**What I'd do →**
1. Run `terraform plan -out=refactor.tfplan` against a copy of production state or a staging replica.
2. Inspect with `terraform show -json refactor.tfplan | jq '.resource_changes[] | select(.change.actions != ["no-op"]) | .address'`.
3. Gate the PR on that command printing only intended moves.

**Why →** The plan is machine-readable truth about what will change.

**Interview tip →** Moves show as `no-op` with `previous_address` set.

</details>

### Q125. Resources are shared between two environments, such as a DNS zone. How do you avoid accidental deletion by either stack? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared dependency owned by neither environment.

**What I'd do →**
1. Put shared resources in their own stack (`foundation/dns`) with its own state and owner.
2. Environment stacks reference them with data sources.
3. Add `prevent_destroy` and service-level deletion protection where available.

**Why →** Resources that outlive environments belong to a lifecycle of their own.

**Interview tip →** "Lifecycle boundaries define state boundaries."

</details>

## Secrets Provisioners and Policy as Code

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. A developer hardcodes a DB password in `terraform.tfvars` and commits it. What do you do right now, and what's the long-term fix? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A credential in Git history is compromised, even if you delete the file.

**What I'd do →**
1. Rotate the password immediately.
2. Remove it from history (`git filter-repo`) and scan with `gitleaks`.
3. Move the value to a secret manager and read at runtime, or use `manage_master_user_password = true` on RDS.
4. Add a pre-commit secret scanner and ignore `*.tfvars` containing secrets.

**Why →** Deleting a file doesn't remove it from history or forks; rotation is the only real fix.

**Interview tip →** "Rotate first, clean later."

</details>

### Q127. How do you avoid Terraform persisting a secret in state when creating a secret in AWS Secrets Manager? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Normal resource attributes are always stored in state.

**What I'd do →**
1. On Terraform 1.10+, use ephemeral values and ephemeral resources for secrets that are only needed during the run.
2. On 1.11+, use write-only arguments where the provider supports them (for example `secret_string_wo` with `secret_string_wo_version` on `aws_secretsmanager_secret_version`).
3. Check the provider docs for support per resource, and fall back to creating the secret container in Terraform while an out-of-band process sets the value.

**Why →** Write-only arguments are sent to the provider but never written to plan or state.

**Interview tip →** Know the pattern, and know that support depends on provider version and resource.

</details>

### Q128. Someone wants a `remote-exec` provisioner to configure servers after creation. What's your advice? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Provisioners are a last resort.

**What I'd do →**
1. Prefer `user_data`/cloud-init for first boot and baked AMIs (Packer) for software.
2. Use Ansible or SSM State Manager for configuration management.
3. If you must use a provisioner, make it idempotent and set `on_failure`.

**Why →** Provisioners don't appear in plan, aren't tracked by state, need network access to the host and fail halfway, leaving tainted resources. HashiCorp documents them as a last resort.

**Interview tip →** Know `local-exec` vs `remote-exec` and why neither is declarative.

</details>

### Q129. You must run a one-off command after a resource is created, such as registering with an external system. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No provider support; a hook is needed.

**What I'd do →** Use `terraform_data` with `local-exec` and triggers.

```hcl
resource "terraform_data" "register" {
  triggers_replace = [aws_instance.app.id]
  provisioner "local-exec" {
    command = "./scripts/register.sh ${aws_instance.app.private_ip}"
  }
  provisioner "local-exec" {
    when    = destroy
    command = "./scripts/deregister.sh"
  }
}
```

**Why →** `triggers_replace` re-runs it when the instance changes, and destroy-time provisioners handle cleanup. Destroy-time provisioners can only reference `self`, so pass values via `triggers`/`input`.

**Interview tip →** Say you'd replace it with a proper provider if one exists.

</details>

### Q130. How do you generate and manage the master password for a new RDS instance? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You don't want a human choosing or seeing the password.

**What I'd do →**
1. Prefer AWS-managed master credentials: `manage_master_user_password = true` stores and rotates it in Secrets Manager.
2. Applications read it from Secrets Manager at runtime via IAM.

```hcl
resource "aws_db_instance" "main" {
  engine                      = "postgres"
  username                    = "app_admin"
  manage_master_user_password = true
}
```

**Why →** No password in code, tfvars or state, and rotation is automatic.

**Interview tip →** The alternative `random_password` puts the value in state.

</details>

### Q131. Compare Sentinel, OPA/Conftest and Checkov for policy as code. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need guardrails before apply.

**What I'd do →**
1. Checkov/Trivy: static scanning of HCL with hundreds of built-in rules; fast adoption.
2. OPA with Conftest: custom Rego policies evaluated against `terraform show -json` plan output; vendor-neutral.
3. Sentinel: HashiCorp's policy language, integrated into HCP Terraform with enforcement levels.
4. Pair a built-in scanner with custom OPA or Sentinel for company-specific rules.

**Why →** Scanners cover generic best practice; plan-based policies can reason about final values and cost.

**Interview tip →** Plan-based checks catch values that only exist after expression evaluation.

</details>

### Q132. How do you prevent public S3 buckets from ever reaching production via Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Defence in depth.

**What I'd do →**
1. Module default: `aws_s3_bucket_public_access_block` with all four flags `true`.
2. CI scan: `checkov -d . --framework terraform` fails the PR on public ACLs.
3. Account-level S3 Block Public Access and an SCP as backstop.

**Why →** If a resource slips through code review, the account control still blocks it.

**Interview tip →** Layer the checks: module, pipeline, org.

</details>

### Q133. Write a policy that denies any plan creating an unencrypted EBS volume. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Custom rule on plan JSON.

**What I'd do →**
```bash
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
conftest test plan.json -p policy/
```
```rego
package main

deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_ebs_volume"
  rc.change.actions[_] == "create"
  not rc.change.after.encrypted
  msg := sprintf("%s must set encrypted = true", [rc.address])
}
```

**Why →** The plan JSON contains the final computed values, so the policy sees what will actually be built.

**Interview tip →** Rego `deny contains msg if` is the current v1 syntax.

</details>

### Q134. Which checks belong in pre-commit, and which only in CI? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fast feedback locally, authority in CI.

**What I'd do →**
1. Pre-commit: `terraform fmt`, `terraform validate`, `tflint`, `terraform-docs`, secret scanning.
2. CI: everything above plus `checkov`/`trivy config`, plan, policy checks.

**Why →** Developers skip slow hooks; CI can't be skipped.

**Interview tip →** CI is the control, hooks are the convenience.

</details>

### Q135. Design least-privilege IAM for a Terraform pipeline. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pipelines often run as `AdministratorAccess`.

**What I'd do →**
1. Two roles: `plan` (read-only plus state read, lock write) and `apply` (write permissions scoped to the services in that stack).
2. PR pipelines assume `plan`; only the protected main-branch job assumes `apply`.
3. Add permission boundaries and use IAM Access Analyzer's policy generation from CloudTrail to tighten.
4. Restrict by trust policy to the exact repo and branch.

**Why →** A compromised PR can run arbitrary code in plan, so plan must not be able to change anything.

**Interview tip →** Mention that `terraform plan` runs provider and data source code from your PR.

</details>

### Q136. Pipeline needs AWS access without storing access keys in GitHub. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long-lived keys in CI secrets are a liability.

**What I'd do →**
1. Create an IAM OIDC identity provider for `token.actions.githubusercontent.com`.
2. Create a role whose trust policy restricts `sub` to `repo:acme/infra:ref:refs/heads/main`.
3. Use `aws-actions/configure-aws-credentials` with `role-to-assume`.

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/gha-terraform-apply
      aws-region: ap-south-1
```

**Why →** Tokens are short-lived and bound to repo and branch; nothing to leak or rotate.

**Interview tip →** Always constrain the `sub` claim, or any repo could assume the role.

</details>

### Q137. A secret leaked in a Terraform plan log in CI. What now? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Log retention makes it widely visible.

**What I'd do →**
1. Rotate the secret.
2. Delete or restrict the log and artifacts.
3. Find why it printed: missing `sensitive = true`, a `local-exec` echoing it, or `TF_LOG=DEBUG`.
4. Add masking in CI and a check forbidding debug logs in shared runs.

**Why →** Treat logs as a leak channel; sensitive marking and masking are the controls.

**Interview tip →** Plan files and `terraform show` output can also contain secrets.

</details>

### Q138. Compare using SOPS or Vault with plain Secrets Manager for providing secrets to Terraform. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Choose a source of truth for secrets.

**What I'd do →**
1. Secrets Manager or SSM Parameter Store: native, IAM-controlled, easiest on AWS.
2. Vault: dynamic, short-lived credentials and a policy engine; more to operate.
3. SOPS: encrypted files in Git, good for small teams but secrets still land in state.

**Why →** The question is "where does the value live and who rotates it?"; Terraform should reference, not own, secret values where possible.

**Interview tip →** Whichever is chosen, keep the secret value out of state when you can.

</details>

### Q139. Why is `sensitive = true` not enough for a private key resource output? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A `tls_private_key` output is marked sensitive and the team feels safe.

**What I'd do →**
1. Explain that sensitive hides the value in CLI output only.
2. Point out that `terraform output -json` and the state file still hold it in plain text.
3. Generate keys outside Terraform where possible, or restrict state access and encrypt.

**Why →** Redaction is a display feature, not a security boundary.

**Interview tip →** Be specific: "state is plain JSON".

</details>

### Q140. An application needs a secret at boot. A teammate puts it in `user_data`. Why is that wrong, and what's the alternative? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** User data is readable from the instance metadata and visible in console and state.

**What I'd do →**
1. Remove the secret from user data.
2. Give the instance an IAM role.
3. Have the app or boot script fetch it from Secrets Manager or SSM Parameter Store at runtime.
4. Enforce IMDSv2 with `metadata_options { http_tokens = "required" }`.

**Why →** Secrets stay out of code, state and metadata, and access is auditable and revocable.

**Interview tip →** Mention rotation: runtime fetch makes it painless.

</details>

### Q141. How do you stop engineers from bypassing policy checks in a pipeline? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A policy check that can be skipped is a suggestion.

**What I'd do →**
1. Enforce policy in the system that applies (HCP Terraform policy sets with hard-mandatory enforcement), not in a script in the repo that PR authors can edit.
2. Protect the workflow files with CODEOWNERS and branch protection; use required workflows.
3. Remove direct apply credentials from humans.
4. Log and review override requests (soft-mandatory).

**Why →** If the PR author controls the check, they control the result.

**Interview tip →** Know advisory, soft-mandatory and hard-mandatory.

</details>

### Q142. How do you verify that third-party providers and modules haven't been tampered with? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Supply-chain risk: providers run with your cloud credentials.

**What I'd do →**
1. Commit the lock file; `init` verifies checksums, and the registry's signing keys are verified for HashiCorp-signed providers.
2. Pin module versions or Git commit SHAs rather than branches.
3. Use an internal mirror and allowlist namespaces.
4. Review upgrades with a plan and read release notes.

**Why →** Pinned, verified, mirrored dependencies reduce what can change without review.

**Interview tip →** Pin Git modules by tag or SHA, not by branch.

</details>

### Q143. Terraform needs temporary admin-like credentials for AWS. How can Vault help? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static keys are risky; dynamic ones expire.

**What I'd do →**
1. Configure Vault's AWS secrets engine with a role mapped to the needed policy.
2. The pipeline authenticates to Vault (OIDC/JWT) and requests credentials.
3. Export them as `AWS_*` environment variables for the run; they expire in minutes.

**Why →** There's no long-lived credential to steal; every run's access is logged in Vault.

**Interview tip →** Mention that HCP Terraform dynamic credentials give the same pattern natively.

</details>

### Q144. What do you keep as evidence for an auditor that policy was enforced on every production change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compliance wants proof, not promises.

**What I'd do →**
1. Retain plan JSON, policy results, approver identity and the commit SHA for each apply.
2. Store them in an immutable bucket or HCP Terraform run history.
3. Link to CloudTrail events from the apply role.

**Why →** Plan, policy and approval together tell the full story of a change.

**Interview tip →** Plan files contain secrets, so encrypt and restrict access.

</details>

### Q145. Why should you replace `null_resource` with `terraform_data`? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old modules use the `null` provider for hooks.

**What I'd do →**
1. Switch to the built-in `terraform_data` resource (Terraform 1.4+).
2. Use `triggers_replace` instead of `triggers`.
3. Migrate with a `moved` block where supported, or recreate since it has no real infrastructure.

**Why →** It needs no extra provider download and can store input and output values.

**Interview tip →** It's a small thing that shows you keep up with the language.

</details>

### Q146. You must pass a Kubernetes auth token to a provider without writing it to state. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Short-lived credentials shouldn't persist.

**What I'd do →**
1. Use the provider's `exec` plugin (as in `aws eks get-token`) so the token is fetched at run time and never stored.
2. On Terraform 1.10+, use ephemeral resources/values where the provider supports them.

**Why →** Values obtained by exec or marked ephemeral aren't saved in plan or state files.

**Interview tip →** Avoid `data "aws_eks_cluster_auth"` tokens for long runs; they expire after about 15 minutes.

</details>

### Q147. A teammate wants a `remote-exec` over SSH through a bastion for every new instance. What do you suggest instead? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SSH from Terraform couples infrastructure creation to network reachability and key handling.

**What I'd do →**
1. Bake the setup into an AMI with Packer, or use cloud-init.
2. For ongoing config, use Ansible over SSM Session Manager, or SSM Run Command/Association.
3. Close port 22 entirely.

**Why →** No keys, no bastion, and the configuration is repeatable and testable outside Terraform.

**Interview tip →** "Terraform provisions, configuration management configures."

</details>

### Q148. Tagging compliance: how do you guarantee every resource has `Owner` and `CostCenter`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Missing tags break cost allocation.

**What I'd do →**
1. Set `default_tags` in the provider, fed from required variables with validation.
2. A policy check on plan JSON verifying `tags_all` contains both keys.
3. AWS Config or tag policies at the org level as the backstop.

**Why →** Provider defaults make it automatic; policy makes it enforced.

**Interview tip →** Check `tags_all`, not `tags`, in policies.

</details>

### Q149. How do you share a database password between a Terraform-created RDS and an app deployed by another tool? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both sides need the secret, neither should be hand-copied.

**What I'd do →**
1. Let RDS manage the master secret (`manage_master_user_password`); output only the secret ARN.
2. Grant the app's IAM role `secretsmanager:GetSecretValue` on that ARN.
3. The app reads at runtime (External Secrets Operator on Kubernetes works well).

**Why →** The value never passes through Terraform outputs, pipelines or Git.

**Interview tip →** Share ARNs and names, not values.

</details>

### Q150. How do you review a PR that adds a `local-exec` provisioner? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Arbitrary code runs on the CI runner with deployment credentials.

**What I'd do →**
1. Ask why no provider or data source can do it.
2. Read the script: idempotent? quoting safe? no secret in command line (visible in process list and logs)?
3. Check failure behaviour (`on_failure`) and that destroy-time behaviour is handled.
4. Require a ticket to retire it when native support exists.

**Why →** Provisioners run outside Terraform's model, so the review must cover the script as if it were production code.

**Interview tip →** Mention injection risk if interpolating untrusted variables into commands.

</details>

## Pipelines Terraform Cloud and OpenTofu

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. Build a basic GitHub Actions pipeline: plan on pull request, apply on merge to main. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The standard GitOps-for-infrastructure loop.

**What I'd do →**
1. Trigger `plan` on `pull_request`, `apply` on `push` to `main`.
2. Authenticate with OIDC, pin the Terraform version.
3. Protect `main` and require the plan job to pass.

```yaml
on:
  pull_request:
  push: { branches: [main] }
jobs:
  terraform:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read, pull-requests: write }
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.9.8 }
      - run: terraform init -input=false
      - run: terraform fmt -check -recursive
      - run: terraform plan -input=false -out=tfplan
      - if: github.ref == 'refs/heads/main'
        run: terraform apply -input=false tfplan
```

**Why →** Code review gates every change, and merge is the only path to production.

**Interview tip →** Mention approval environments and concurrency for prod.

</details>

### Q152. Reviewers can't read raw plan logs. How do you present the plan in the PR? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Review quality drops when diffs are hard to find.

**What I'd do →**
1. Run `terraform show -no-color tfplan` and post it as a PR comment (collapsed, truncated).
2. Add a short summary from `terraform show -json` (counts of create/update/delete).
3. Use Atlantis, Spacelift, HCP Terraform or an action that updates one sticky comment per PR.

**Why →** Reviewers approve what they can see; a destroy count in the summary is the number they care about.

**Interview tip →** Redact sensitive values and avoid posting secrets in PR comments.

</details>

### Q153. Two PRs merge close together and both apply to the same stack. What prevents trouble? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Concurrent applies to one state.

**What I'd do →**
1. Backend locking guarantees only one writer at a time.
2. A CI concurrency group per stack serialises runs without cancelling an in-flight apply.
3. Plans are re-created on the merged commit, so a stale PR plan is never applied.

**Why →** Locking prevents corruption; serialisation prevents confusing failures.

**Interview tip →** `cancel-in-progress: false` for applies. Cancelling an apply can leave partial state.

</details>

### Q154. How do you add a manual approval before prod applies? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Prod changes need a human gate.

**What I'd do →**
1. Split the workflow: `plan` job produces the artifact; `apply` job targets a GitHub `environment: production`.
2. Configure that environment with required reviewers and a deployment branch rule.
3. Apply the saved plan from the artifact.

```yaml
apply:
  needs: plan
  environment: production
  steps:
    - uses: actions/download-artifact@v4
      with: { name: tfplan }
    - run: terraform apply -input=false tfplan
```

**Why →** The approver sees the exact plan; the apply is bound to it.

**Interview tip →** Same shape in GitLab with `when: manual` and protected environments.

</details>

### Q155. Compare Atlantis, HCP Terraform and a hand-built GitHub Actions pipeline. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Choosing the tooling for a growing team.

**What I'd do →**
1. GitHub Actions or GitLab CI: flexible, free to start, but you build locking, comments, policy and drift detection yourself.
2. Atlantis: self-hosted PR automation (`atlantis plan`, `atlantis apply`) with locking per directory.
3. HCP Terraform: managed runs, state, RBAC, policy, drift detection and private registry.

**Why →** The trade-off is build effort and control versus subscription cost.

**Interview tip →** Say what you'd choose for a 5-person team and for a 200-person platform.

</details>

### Q156. A saved plan was approved yesterday and apply fails because main changed. How do you avoid dead approvals? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long gaps between plan and apply make stale plans likely.

**What I'd do →**
1. Plan on the merge commit in the apply pipeline, then approve and apply right after.
2. Set an expiry on approvals and re-plan if older than N hours.
3. Use merge queues so each change is planned against current main.

**Why →** The shorter the plan-to-apply window, the less drift can intervene; the stale-plan check is the safety net, not the workflow.

**Interview tip →** Distinguish "PR plan" (informational) from "apply plan" (binding).

</details>

### Q157. Explain HCP Terraform workflows: VCS-driven, CLI-driven and API-driven. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A teammate is new to HCP Terraform workspaces.

**What I'd do →**
1. VCS-driven: connect a repo; pushes trigger plans, merges trigger applies, PRs get speculative plans.
2. CLI-driven: `terraform plan` runs remotely via the `cloud` block, with state and runs in HCP.
3. API-driven: pipelines upload configuration versions and create runs via API for custom flows.

```hcl
terraform {
  cloud {
    organization = "acme"
    workspaces { name = "network-prod" }
  }
}
```

**Why →** Pick by where you want the trigger: Git events, developers, or your own automation.

**Interview tip →** A workspace is state plus variables plus runs plus permissions.

</details>

### Q158. A change in the network workspace should trigger the app workspace automatically. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Chained workspaces.

**What I'd do →**
1. Configure a run trigger on the app workspace with the network workspace as source.
2. Keep the data hand-off explicit through outputs (via `tfe_outputs` or SSM).

**Why →** Downstream plans run after upstream applies, catching impact quickly.

**Interview tip →** Run triggers queue a plan, so you still decide on apply.

</details>

### Q159. Private resources (RDS in a VPC) can't be reached by HCP Terraform's public runners. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Providers like PostgreSQL or Vault need network access.

**What I'd do →**
1. Deploy HCP Terraform agents inside the VPC, outbound-only to HCP.
2. Create an agent pool and set the workspace's execution mode to "Agent".
3. Run the agent as a container with a least-privilege role.

**Why →** Agents poll HCP over HTTPS, so you don't open inbound ports.

**Interview tip →** In GitHub Actions, the equivalent is a self-hosted runner in the VPC.

</details>

### Q160. Migrate existing S3-backend stacks into HCP Terraform with no downtime. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** State move under live infrastructure.

**What I'd do →**
1. Freeze applies on the stack (disable the pipeline, announce a window).
2. Replace the backend with the `cloud` block and run `terraform init -migrate-state`.
3. Configure variables and credentials in the workspace; run a speculative plan: must be empty.
4. Update CI to use HCP runs, archive the old state object.

**Why →** The state is copied, not recreated; an empty plan proves the move.

**Interview tip →** Migrate one low-risk workspace first as a pilot.

</details>

### Q161. Why would a company evaluate OpenTofu instead of Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Licence and governance questions after HashiCorp moved Terraform to the BUSL licence in 2023.

**What I'd do →**
1. OpenTofu is an open-source fork under the Linux Foundation, MPL-2.0 licensed, compatible with Terraform configuration and providers up to the fork point.
2. Check legal requirements: BUSL restricts competing commercial offerings, which matters for vendors, less for most internal users.
3. Evaluate features: native state encryption and other OpenTofu-only features.

**Why →** For most users it's a governance and licence decision, and technically a small switch.

**Interview tip →** Stay neutral and factual; know both exist and why.

</details>

### Q162. How do you migrate a repo from Terraform 1.6 to OpenTofu? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Drop-in replacement, but verify.

**What I'd do →**
1. Back up state; confirm Terraform version is within the compatible range from the OpenTofu migration guide.
2. Install `tofu`, run `tofu init` (it uses the OpenTofu registry) and regenerate the lock file.
3. Run `tofu plan`; expect an empty plan.
4. Update CI, Docker images and docs.

```bash
tofu init -upgrade
tofu plan
```

**Why →** State format is compatible within supported version ranges; an empty plan is the test.

**Interview tip →** Moving back isn't guaranteed after newer OpenTofu features write state.

</details>

### Q163. When would you use OpenTofu's native state encryption? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** State holds secrets; backend encryption protects at rest, but anyone with read access sees plaintext.

**What I'd do →**
1. Configure an `encryption` block with a key provider (KMS, PBKDF2 passphrase) and method (AES-GCM).
2. Apply to state and plan files.
3. Roll out with a fallback block so existing plaintext state migrates smoothly.

**Why →** Client-side encryption means the backend operator, or someone who can read the bucket, can't read secrets without the key.

**Interview tip →** Key loss means state loss; back up key material and test recovery.

</details>

### Q164. Name differences between OpenTofu and Terraform that could affect an adoption decision. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A feature-parity question.

**What I'd do →**
1. OpenTofu adds native state and plan encryption (1.7), and `for_each` on providers (1.9) and an `-exclude` flag (1.9).
2. Terraform ships its own newer features, such as ephemeral resources and write-only arguments, which OpenTofu may add later on its own timeline.
3. Check your providers, tooling (Terragrunt, Atlantis, Checkov) and HCP-specific features.

**Why →** The two projects have diverged; decide using your required features and tooling.

**Interview tip →** Verify versions against current release notes before answering in detail.

</details>

### Q165. How do you add a nightly drift detection job in CI? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scheduled, read-only comparison.

**What I'd do →**
```yaml
on:
  schedule: [{ cron: "0 2 * * *" }]
jobs:
  drift:
    steps:
      - run: terraform init -input=false
      - run: |
          set +e
          terraform plan -detailed-exitcode -lock=false -input=false
          rc=$?
          [ $rc -eq 2 ] && ./notify-slack.sh "Drift detected in ${{ github.workflow }}"
          [ $rc -eq 1 ] && exit 1
          exit 0
```

**Why →** Exit code 2 signals pending changes without parsing text; a read-only role makes it safe.

**Interview tip →** Mention `-lock=false` is acceptable only for read-only plans.

</details>

### Q166. Finance wants cost visibility on infrastructure PRs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cost surprise comes from unreviewed changes.

**What I'd do →**
1. Add Infracost to the PR workflow: `infracost breakdown --path . --format json` and `infracost diff` against the base branch.
2. Post the monthly cost delta as a comment.
3. Add a policy that blocks PRs exceeding a threshold without approval.

**Why →** A "+$4,200/month" line changes review behaviour.

**Interview tip →** Cost estimates for usage-based resources are approximations.

</details>

### Q167. How do you promote the same Terraform code from dev to stage to prod? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Promotion needs to move through environments with the same module versions.

**What I'd do →**
1. Tag module releases; environments pin versions in their own directory.
2. Pipeline: merge → apply dev → automated checks → apply stage → manual approval → apply prod, each with its own plan.
3. Keep environment differences in tfvars only.

**Why →** What was proven in dev is the same code in prod; only inputs differ.

**Interview tip →** Plans are per-environment; never promote a plan file.

</details>

### Q168. Which CI gates would you require on every infrastructure PR? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A baseline.

**What I'd do →**
1. `terraform fmt -check -recursive`
2. `terraform validate`
3. `tflint --recursive`
4. `checkov -d .` or `trivy config .`
5. `terraform plan` with output on the PR
6. Policy checks on the plan

**Why →** Each gate is cheap and each catches a different class of mistake.

**Interview tip →** Fail fast: run cheap checks first.

</details>

### Q169. In a monorepo with 40 stacks, how do you plan only the stacks that changed? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Planning everything on every PR is slow and noisy.

**What I'd do →**
1. Detect changed paths (`git diff --name-only origin/main...HEAD`) or use `paths:` filters.
2. Map changed module directories to the stacks that use them.
3. Generate a job matrix of affected stacks and run plans in parallel.
4. Apply in dependency order (network before apps).

**Why →** Affected-only plans scale; a shared module change correctly fans out to all consumers.

**Interview tip →** Shared module changes are the case naive path filters miss.

</details>

### Q170. How do you keep the Terraform version consistent between laptops and CI? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Version skew produces confusing plan differences.

**What I'd do →**
1. Commit a `.terraform-version` (tfenv) or `.tool-versions` (mise/asdf) file.
2. In CI, read it: `terraform_version_file` in `setup-terraform`.
3. Set `required_version` in code as the hard stop.

**Why →** One source of truth for the version and a guard in the code.

**Interview tip →** Bump the version in one PR with a clean plan in every environment.

</details>

### Q171. Your HCP Terraform runs can't reach an internal Git server. What are your options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** VCS or private module sources aren't reachable from the managed runners.

**What I'd do →**
1. Publish modules to the HCP private registry instead of private Git URLs.
2. Use agents in your network for runs that need internal access.
3. Or run CLI-driven, with your CI uploading the configuration.

**Why →** Moving the dependency to where the runner can reach it is easier than opening network paths.

**Interview tip →** Note that agents handle runs, but VCS webhooks still need to reach HCP.

</details>

### Q172. An emergency requires a manual apply to prod outside the pipeline. How do you handle it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pipeline is down or too slow for an incident.

**What I'd do →**
1. Use a break-glass role that requires approval, MFA and is alerted on assumption.
2. Apply the smallest possible change, with a saved plan reviewed by a second person.
3. Immediately commit the same change to Git so code and state agree.
4. Post-incident, review whether the pipeline path could have been used.

**Why →** Emergencies happen, but they must be audited and reconciled to avoid drift.

**Interview tip →** "Break-glass is allowed, silent break-glass isn't."

</details>

### Q173. How do you handle secrets as environment variables in CI for Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Providers like Datadog or Cloudflare need API tokens.

**What I'd do →**
1. Store them in the CI secret store, exposed only to the apply job and protected branches.
2. Pass them as provider environment variables (`DD_API_KEY`) or `TF_VAR_*` marked `sensitive`.
3. Prefer OIDC or dynamic credentials where the provider supports them.

**Why →** Secrets never appear in code, and PR jobs from forks never see them.

**Interview tip →** Never `echo` env in debug steps.

</details>

### Q174. Where does Terraform sit relative to GitOps tools like Argo CD? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Teams blur the boundary between infrastructure and application delivery.

**What I'd do →**
1. Terraform: cloud infrastructure (VPC, EKS, IAM, databases, DNS) and cluster bootstrap including installing Argo CD.
2. Argo CD: everything that runs inside the cluster, reconciled continuously from Git.
3. Share values through outputs, SSM or annotations on service accounts.

**Why →** Terraform is plan/apply on demand and weak at continuous reconciliation; GitOps controllers are strong at it.

**Interview tip →** Avoid managing app Helm releases in Terraform for fast-moving services.

</details>

### Q175. Leadership wants developers to self-serve infrastructure safely. How do you design the platform? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tickets to the platform team don't scale.

**What I'd do →**
1. Publish versioned golden modules in a private registry with examples.
2. Give teams their own repos and workspaces created from a template, with guardrails: policy as code, cost checks, least-privilege roles.
3. Offer a small form or Backstage template that generates a PR with the module call.
4. Platform team owns modules and policies; app teams own their usage.

**Why →** Paved roads let teams move fast, and guardrails keep them safe.

**Interview tip →** Measure it: time to first environment and number of exceptions.

</details>

## Testing Upgrades and Large-Scale Operations

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. What does each of `terraform validate`, `plan` and `terraform test` actually check? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A team thinks `validate` means the config works.

**What I'd do →**
1. `validate`: syntax, types and references only; no cloud calls, no provider credentials.
2. `plan`: computes the real diff against state and the cloud; catches invalid values the API rejects at plan time.
3. `terraform test`: runs `.tftest.hcl` files with assertions against plan or apply results.

**Why →** They're layers: structure, behaviour against reality, and behaviour you assert.

**Interview tip →** Many API errors only appear on apply (quotas, IAM), so plan isn't proof.

</details>

### Q177. Write a native test for a module that creates an encrypted S3 bucket. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use `terraform test` at plan level for fast feedback.

**What I'd do →**
```hcl
# tests/bucket.tftest.hcl
variables {
  name = "acme-test-bucket"
}

run "bucket_is_encrypted" {
  command = plan

  assert {
    condition     = aws_s3_bucket_server_side_encryption_configuration.this.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm == "aws:kms"
    error_message = "Bucket must use KMS encryption."
  }
}
```
Run with `terraform test`.

**Why →** Plan-mode tests don't create anything, so they run on every PR without cost.

**Interview tip →** Use `mock_provider` (1.7+) to avoid needing credentials.

</details>

### Q178. Compare Terratest with `terraform test`. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Picking an integration test tool.

**What I'd do →**
1. `terraform test`: native HCL, no extra language, supports plan and apply, mocks; ideal for module contracts.
2. Terratest: Go tests with full programming power, such as calling the deployed endpoint, retrying, SSH checks.
3. Use native tests by default, and Terratest for end-to-end behaviour.

**Why →** Smaller toolchain wins unless you need to test the running system.

**Interview tip →** Mention cost and cleanup for either.

</details>

### Q179. Real-cloud integration tests leave orphaned resources when they fail midway. How do you handle that? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cost leak and quota exhaustion in the test account.

**What I'd do →**
1. Run tests in a dedicated sandbox account with budgets and alarms.
2. Tag everything with `test-run-id` and `ttl`.
3. Run a nightly sweeper (`cloud-nuke` or a custom script) deleting expired items.
4. Always run `destroy` in a `defer`/`always()` step.

**Why →** Failures are normal; cleanup must not depend on tests succeeding.

**Interview tip →** Account-level isolation limits blast radius.

</details>

### Q180. How do you upgrade from Terraform 1.5 to 1.9 safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A multi-version jump across many stacks.

**What I'd do →**
1. Read the changelogs for each minor version.
2. Upgrade the CLI in a branch with dev state; run `plan`: expect no changes.
3. Roll through stage then prod; keep prod on the old CLI until the others are clean.
4. Bump `required_version` last.

**Why →** After a newer version writes state, older versions can't read it; rollout order matters.

**Interview tip →** Take a state backup before the first apply with the new CLI.

</details>

### Q181. Major AWS provider upgrade from v4 to v5 (or v5 to v6) across many stacks. How do you plan it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Major versions change defaults and remove deprecated arguments.

**What I'd do →**
1. Read the upgrade guide; list breaking changes that touch your resources.
2. Upgrade to the latest minor of the old major first to see deprecation warnings.
3. Bump a single low-risk stack, fix code, and capture the pattern.
4. Roll out stack by stack with plans reviewed for unexpected diffs.
5. Pin to the new major in a shared module constraint.

**Why →** The provider is where most upgrade breakage lives, so move slowly and learn from the pilot.

**Interview tip →** Cite a concrete change, such as the v4 S3 bucket argument split into separate resources.

</details>

### Q182. A stack is still on Terraform 0.12 syntax and old providers. What upgrade path do you follow? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Skipping versions isn't supported.

**What I'd do →**
1. Step through 0.13, 0.14, 0.15 and 1.0 one at a time, using the matching upgrade guide.
2. At 0.13 run `terraform 0.13upgrade` and add `required_providers` sources.
3. Run `plan` after each step: expect no diffs.
4. Then proceed to current 1.x.

**Why →** State format upgrades are one-way and each step has a guided migration.

**Interview tip →** Do it on a copy of state first.

</details>

### Q183. Why do you review `.terraform.lock.hcl` diffs in PRs? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A lock file change shows up in a PR.

**What I'd do →**
1. Check which provider version and hashes changed.
2. Confirm it matches an intended constraint bump.
3. Ensure all platform hashes are present.

**Why →** The lock file defines what code runs with your credentials, so unexpected changes are supply-chain red flags.

**Interview tip →** Unexplained lock changes are worth a question.

</details>

### Q184. Add a runtime health assertion to infrastructure without blocking applies. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want post-apply validation that warns instead of failing.

**What I'd do →** Use a `check` block (Terraform 1.5+).

```hcl
check "app_is_healthy" {
  data "http" "health" {
    url = "https://${aws_lb.app.dns_name}/health"
  }
  assert {
    condition     = data.http.health.status_code == 200
    error_message = "Health endpoint did not return 200."
  }
}
```

**Why →** Check failures show as warnings on every plan/apply, which is useful for continuous validation.

**Interview tip →** Contrast with `precondition`/`postcondition`, which block.

</details>

### Q185. You need to push a change across 100 workspaces. How do you do it safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fleet-wide rollout, such as a module version bump.

**What I'd do →**
1. Group workspaces into rings: canary (2), non-prod, small prod, rest.
2. Automate with a script or the HCP Terraform API (`tfe` provider or CLI) that opens PRs or triggers runs per ring.
3. Require clean or expected plans before each ring; pause on failures.
4. Track progress in a dashboard.

**Why →** Rings limit blast radius and give early signal.

**Interview tip →** Resist "big bang" changes for shared infrastructure.

</details>

### Q186. API rate limits cause "Throttling" errors during large applies. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hundreds of parallel API calls hit account limits.

**What I'd do →**
1. Lower `-parallelism` (for example `-parallelism=5`).
2. Raise the provider's retry settings (`max_retries`, and retry mode `adaptive` for AWS).
3. Split the state so fewer resources are touched at once.

**Why →** Terraform retries with backoff but needs headroom; less concurrency means fewer throttles.

**Interview tip →** Check if other tools hit the same account concurrently.

</details>

### Q187. Developers leave dev environments running over weekends, burning money. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Idle cost.

**What I'd do →**
1. Add a scheduled pipeline running `terraform destroy` on dev stacks at night (or scale to zero with schedule-based ASG actions).
2. Tag `ttl` on ephemeral environments and sweep.
3. Make re-creation a single command.

**Why →** If environments rebuild reliably, destroying them is free.

**Interview tip →** This only works with fully-codified, tested environments.

</details>

### Q188. A failed apply left half the resources created. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Partial apply: Terraform saves progress to state as it goes.

**What I'd do →**
1. Read the error; fix the cause (permissions, quota, name collision).
2. Re-run `terraform plan`; it plans only what's missing.
3. If a resource was created but not recorded (timeout), `import` it before applying.
4. Apply again.

**Why →** Terraform is convergent: re-applying finishes the job.

**Interview tip →** Check for tainted resources in the plan.

</details>

### Q189. Someone ran `terraform destroy` against production. What's your recovery plan? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Worst-case incident.

**What I'd do →**
1. Stop further changes and revoke the identity's access.
2. Check what survived: deletion protection and `prevent_destroy` may have saved databases.
3. Restore stateful data from snapshots and backups first (RDS, EBS, S3 versions).
4. Re-apply the code (state is empty or reflects destroyed items) to recreate stateless layers.
5. Run a blameless review: remove human prod credentials, require approval, enable protection.

**Why →** Data recovery comes first; infrastructure is reproducible from code.

**Interview tip →** This is why DR drills and deletion protection exist.

</details>

### Q190. What do you look for when reviewing a teammate's Terraform PR? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A code review for infrastructure.

**What I'd do →**
1. Read the plan first: creates, updates, destroys, replacements; question every `forces replacement`.
2. Security: IAM wildcards, open CIDRs, unencrypted storage, public access.
3. Blast radius: shared resources, state boundaries.
4. Quality: variable validation, module pinning, naming, tags, no hardcoded IDs.

**Why →** The plan shows effects; the code shows intent; both matter.

**Interview tip →** "I review the plan before I review the code."

</details>

### Q191. Design a multi-account AWS landing zone managed with Terraform. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dozens of accounts, consistent guardrails.

**What I'd do →**
1. Use AWS Organizations with OUs; manage them and SCPs in a `management` stack.
2. An account factory (AWS Control Tower Account Factory for Terraform, or custom) creates accounts from a request file.
3. Baseline modules per account: IAM roles, CloudTrail, Config, VPC, budgets.
4. Layered states, a central state bucket in a tooling account, and OIDC roles per account.

**Why →** Accounts are created and baselined by code, so each looks identical and is auditable.

**Interview tip →** Describe the account-vending flow end to end.

</details>

### Q192. New AWS feature, but provider support hasn't landed. Options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Resource missing in `hashicorp/aws`.

**What I'd do →**
1. Check the `awscc` provider (generated from Cloud Control API).
2. Use `aws_cloudformation_stack` wrapping a minimal template.
3. Use `terraform_data` + AWS CLI as a stopgap, documented as technical debt.

**Why →** Pick the option that still gives drift detection and dependency tracking.

**Interview tip →** Also open a provider issue.

</details>

### Q193. How do you write unit tests for policy rules without deploying? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Policies need tests like code.

**What I'd do →**
1. Keep a folder of sample plan JSONs (good and bad cases).
2. Run `conftest verify` for Rego unit tests, or `opa test policy/`.
3. Generate sample plans from small fixtures using `terraform show -json`.
4. Run policy tests in CI whenever policies change.

**Why →** Untested policies either block everything or nothing.

**Interview tip →** Test both the deny and the allow path.

</details>

### Q194. Where do you draw the line between Terraform and Helm/Kubernetes providers for cluster add-ons? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Platform bootstrapping.

**What I'd do →**
1. Terraform: cluster, node groups, IAM roles for service accounts, and a minimal set of bootstrap add-ons (CNI, CoreDNS, Argo CD).
2. GitOps tool: everything else, such as ingress controllers, monitoring and apps.
3. Separate state for cluster and for bootstrap resources.

**Why →** Slow-moving infrastructure benefits from plan/apply; fast-moving workloads benefit from continuous reconciliation.

**Interview tip →** Avoid long `helm_release` lists in one state.

</details>

### Q195. A new hire needs to start contributing to Terraform safely on day one. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Onboarding with a safety net.

**What I'd do →**
1. Give a read-only role so `terraform plan` works locally.
2. Pair on a small PR in a dev stack; CI does the apply.
3. Point them at the module README, the style guide and pre-commit config.

**Why →** They learn the workflow without the ability to damage prod.

**Interview tip →** Safe environments make fast onboarding possible.

</details>

### Q196. Does Terraform have rollback? How do you undo a bad change? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A change made things worse.

**What I'd do →**
1. Revert the commit and apply (roll forward with the old code).
2. For replaced resources, recreate from the previous config; stateful data comes from backups.
3. If state is wrong, restore the previous state version from S3 versioning, then plan.

**Why →** Terraform has no transactional rollback; "revert in Git and apply" works for stateless resources only.

**Interview tip →** Distinguish stateless (easy) from stateful (needs backups).

</details>

### Q197. List a short checklist of Terraform best practices you'd enforce in a team. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A standard answer to "your top practices".

**What I'd do →**
1. Remote state with locking and versioning.
2. Pinned providers and committed lock file.
3. Modules with typed, validated variables.
4. Plan in PR, apply from CI only.
5. Secrets never in code; least-privilege roles.
6. Policy checks and drift detection.

**Why →** Each point maps to a real failure mode: lost state, drift, version skew, leaks.

**Interview tip →** Tie each one to an incident it prevents.

</details>

### Q198. Terraform vs Crossplane vs Pulumi vs CDK: how do you advise a team? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tool selection.

**What I'd do →**
1. Terraform/OpenTofu: largest provider ecosystem, declarative HCL, plan/apply workflow, easiest hiring.
2. Pulumi / AWS CDK: general-purpose languages, good for developer-heavy teams, more abstraction power and more ways to write unmaintainable code.
3. Crossplane: Kubernetes-native control plane, continuous reconciliation, fits platform teams built around Kubernetes APIs.
4. Choose by team skills, existing platform and ecosystem fit.

**Why →** Tool religion is less important than consistent practice, and migration cost is high.

**Interview tip →** Give a recommendation and the condition under which you'd change it.

</details>

### Q199. A plan shows a diff on a resource nobody touched, after a provider upgrade. How do you decide whether to apply? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A new provider version computes attributes differently, or a default changed.

**What I'd do →**
1. Read the diff attribute by attribute; check the provider changelog.
2. If the change is a new default, either accept it or pin the old behaviour explicitly in code.
3. If it's a bug, pin the previous provider version and file an issue.

**Why →** No-op upgrades are the goal; unexpected diffs deserve explanation before apply.

**Interview tip →** Explicit config beats relying on defaults that can change.

</details>

### Q200. After a year of growth, the team has 60 stacks with inconsistent practices. How do you standardise without stopping delivery? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Organic sprawl.

**What I'd do →**
1. Inventory stacks: Terraform versions, provider versions, backend types, owners.
2. Publish a baseline template repo (backend, pre-commit, CI, policy) and a migration guide.
3. Migrate by risk and value: top 10 critical stacks first, with no-diff plans.
4. Track compliance in a scorecard; make new stacks use the template by default.

**Why →** Incremental, measured migration beats a freeze; the template makes the right way the easy way.

**Interview tip →** Show a metric, such as the percentage of stacks on the baseline.

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
