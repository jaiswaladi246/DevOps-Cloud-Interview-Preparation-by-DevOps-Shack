# Terraform & IaC — Cheatsheet

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

> Dense, scannable reference for daily Terraform work and interview prep: CLI, HCL patterns, state, modules, backends, quality gates and CI.

[⬅ Back to Terraform & IaC](./README.md) · [🏠 Home](../README.md)

## Table of contents

- [Core workflow](#core-workflow)
- [Plan and apply flags](#plan-and-apply-flags)
- [State commands](#state-commands)
- [HCL building blocks](#hcl-building-blocks)
- [Expressions and functions](#expressions-and-functions)
- [Meta-arguments and lifecycle](#meta-arguments-and-lifecycle)
- [Modules](#modules)
- [Backends and locking](#backends-and-locking)
- [Import, move and remove](#import-move-and-remove)
- [Environment variables](#environment-variables)
- [Quality gates](#quality-gates)
- [Testing](#testing)
- [GitHub Actions snippets](#github-actions-snippets)
- [Debugging](#debugging)
- [Version constraints](#version-constraints)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## Core workflow

| Command | What it does |
|---|---|
| `terraform init` | Downloads providers and modules, configures the backend, writes `.terraform.lock.hcl` |
| `terraform init -upgrade` | Re-resolves providers and modules to the newest versions allowed by constraints |
| `terraform init -backend=false` | Initialise without touching the backend (good for `validate` in CI) |
| `terraform init -backend-config="bucket=x"` | Supply backend settings that cannot live in code |
| `terraform init -reconfigure` | Ignore the saved backend config and start fresh |
| `terraform init -migrate-state` | Copy state from the old backend to the new one |
| `terraform fmt -recursive` | Rewrite files to canonical style |
| `terraform fmt -check -diff` | CI mode: fail if anything needs formatting |
| `terraform validate` | Check syntax and internal consistency (no cloud calls) |
| `terraform plan` | Show what would change |
| `terraform apply` | Apply changes after confirmation |
| `terraform apply tfplan` | Apply a saved plan exactly as reviewed |
| `terraform destroy` | Destroy everything in this state (same as `apply -destroy`) |
| `terraform output -raw name` | Print one output without quotes (scripting) |
| `terraform output -json` | All outputs as JSON |
| `terraform show tfplan` | Human-readable view of a saved plan |
| `terraform show -json tfplan` | Plan as JSON for policy tools |
| `terraform graph \| dot -Tpng > g.png` | Dependency graph |
| `terraform providers` | Show providers required by config and modules |
| `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64` | Add hashes for several platforms to the lock file |
| `terraform version` | Terraform and provider versions |
| `terraform console` | Evaluate expressions interactively against current state |
| `terraform workspace list/new/select/show/delete` | Manage workspaces |
| `terraform force-unlock <ID>` | Release a stuck lock (only if you are certain) |

## Plan and apply flags

| Flag | Use |
|---|---|
| `-out=tfplan` | Save the plan to apply later |
| `-var 'name=value'` / `-var-file=prod.tfvars` | Set inputs |
| `-target=aws_instance.web` | Limit to one resource and its dependencies (use for break-glass only) |
| `-replace=aws_instance.web` | Force recreation (replaces `taint`) |
| `-refresh-only` | Update state to match reality, change nothing |
| `-refresh=false` | Skip the refresh step (faster, riskier) |
| `-destroy` | Plan a destroy |
| `-detailed-exitcode` | Exit 0 no changes, 1 error, 2 changes |
| `-lock-timeout=120s` | Wait for a lock instead of failing at once |
| `-lock=false` | Do not lock (almost never) |
| `-parallelism=20` | Concurrent operations, default 10 |
| `-input=false` | Never prompt (CI) |
| `-no-color` | Clean logs |
| `-auto-approve` | Skip the prompt (CI after a reviewed plan, or labs) |
| `-generate-config-out=gen.tf` | With `import` blocks, write resource config for you |
| `-compact-warnings` | Shorter warnings |

Plan symbols: `+` create, `-` destroy, `~` update in place, `-/+` replace (destroy then create), `+/-` replace with `create_before_destroy`, `<=` data read.

## State commands

```bash
terraform state list                              # all addresses
terraform state list module.vpc                   # filter
terraform state show aws_instance.web             # attributes of one resource
terraform state mv aws_instance.a aws_instance.b  # rename in state
terraform state mv -state-out=other.tfstate aws_instance.a aws_instance.a   # move between states
terraform state rm aws_instance.web               # forget it, do not destroy it
terraform state pull > backup.tfstate             # download current state
terraform state push backup.tfstate               # upload (dangerous, check serial)
terraform state replace-provider registry.terraform.io/-/aws hashicorp/aws
terraform show -json | jq '.values.root_module.resources[].address'
```

| Rule | Why |
|---|---|
| Never edit `terraform.tfstate` by hand | Lineage and serial checks, easy to corrupt |
| Back up before surgery | `state pull`, plus S3 bucket versioning |
| State holds secrets in plain text | Encrypt the bucket, restrict access, never commit |
| Prefer `moved`, `import` and `removed` blocks over CLI surgery | Reviewable and repeatable |

## HCL building blocks

```hcl
terraform {
  required_version = ">= 1.11"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
}

provider "aws" {
  region = var.region
  default_tags { tags = { Project = "demo", ManagedBy = "terraform" } }
}

provider "aws" {                       # second region
  alias  = "dr"
  region = "ap-southeast-1"
}

variable "instance_count" {
  description = "How many instances."
  type        = number
  default     = 2
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_count must be between 1 and 10."
  }
}

variable "db_password" {
  type      = string
  sensitive = true
}

variable "rules" {
  type = map(object({
    port  = number
    cidrs = optional(list(string), ["10.0.0.0/8"])
  }))
}

locals {
  name = "${var.project}-${var.env}"
  tags = merge(var.tags, { Name = local.name })
}

data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "this" {
  bucket   = "${local.name}-${data.aws_caller_identity.current.account_id}"
  provider = aws.dr                    # pick a non-default provider
}

output "bucket_arn" {
  description = "ARN of the bucket."
  value       = aws_s3_bucket.this.arn
  sensitive   = false
}
```

| Block | Purpose |
|---|---|
| `terraform` | Version and provider requirements, backend |
| `provider` | Configure a plugin (region, auth, default tags) |
| `resource` | Create and manage an object |
| `data` | Read an existing object |
| `variable` | Module input |
| `locals` | Named expressions |
| `output` | Module return value |
| `module` | Call a child module |
| `moved` / `import` / `removed` | Refactor and adopt resources declaratively |
| `check` | Non-blocking assertions that warn during plan and apply |

Common variable types: `string`, `number`, `bool`, `list(string)`, `set(string)`, `map(string)`, `object({...})`, `tuple([...])`, `any`.

## Expressions and functions

```hcl
# conditional
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"

# for expressions
upper_names = [for n in var.names : upper(n)]
by_name     = { for s in aws_subnet.private : s.tags["Name"] => s.id }
filtered    = { for k, v in var.buckets : k => v if v.versioning }
grouped     = { for r in var.rules : r.port => r.name... }     # group by key

# splat
ids = aws_instance.web[*].id

# dynamic blocks
dynamic "ingress" {
  for_each = var.ingress_rules
  content {
    from_port   = ingress.value.port
    to_port     = ingress.value.port
    protocol    = "tcp"
    cidr_blocks = ingress.value.cidrs
  }
}

# optional values and defaults
name = coalesce(var.name, "default")
cfg  = try(var.config.timeout, 30)
one_id = one(aws_s3_bucket.optional[*].id)
```

| Function | Example | Result |
|---|---|---|
| `cidrsubnet(prefix, newbits, num)` | `cidrsubnet("10.0.0.0/16", 8, 2)` | `10.0.2.0/24` |
| `merge(a, b)` | `merge({a=1},{b=2})` | `{a=1,b=2}` |
| `lookup(map, key, default)` | `lookup(var.sizes, "dev", "small")` | value or default |
| `element(list, i)` | `element(["a","b"], 3)` | wraps around: `"b"` |
| `slice(list, from, to)` | `slice(azs, 0, 2)` | first two |
| `flatten([[1],[2,3]])` | | `[1,2,3]` |
| `toset(list)` / `tolist(set)` | `toset(["a","a"])` | `["a"]` |
| `format("%s-%03d", "web", 7)` | | `web-007` |
| `join(",", list)` / `split(",", str)` | | string or list |
| `templatefile(path, vars)` | `templatefile("ud.tftpl", { env = "dev" })` | rendered text |
| `jsonencode(obj)` / `jsondecode(str)` | policy documents | JSON |
| `file(path)` / `filebase64(path)` | | file contents |
| `base64encode(str)` | user data | base64 |
| `try(a, b)` / `can(expr)` | | fallback / boolean |
| `length`, `keys`, `values`, `contains`, `distinct`, `concat`, `zipmap`, `setproduct` | | collection helpers |
| `abspath`, `path.module`, `path.root`, `path.cwd` | | paths |
| `timestamp()` | | changes every run, avoid in resources |

## Meta-arguments and lifecycle

```hcl
resource "aws_instance" "web" {
  for_each = toset(["a", "b"])        # or count = 3, never both
  depends_on = [aws_iam_role_policy.x]
  provider   = aws.dr

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags["LastScan"], desired_capacity]
    replace_triggered_by  = [aws_launch_template.this.latest_version]

    precondition {
      condition     = data.aws_ami.x.architecture == "x86_64"
      error_message = "AMI must be x86_64."
    }
    postcondition {
      condition     = self.public_ip == null
      error_message = "Instance must not have a public IP."
    }
  }
}
```

| Choice | Prefer | Because |
|---|---|---|
| Identical copies, on/off toggle | `count` | Simple and numeric |
| Distinct named items | `for_each` | Stable addresses when items are added or removed |
| Order not inferable from references | `depends_on` | Explicit edge in the graph |
| Zero-downtime replacement | `create_before_destroy` | Needs unique names (`name_prefix`) |
| Autoscaler owns a field | `ignore_changes` | Stop plan noise |
| Critical data stores | `prevent_destroy` | Plan fails instead of deleting |

## Modules

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"     # registry
  version = "~> 6.0"
  name    = "main"
  cidr    = "10.0.0.0/16"
}

module "web" {
  source = "../../modules/web"                  # local path
  count  = 2                                    # modules support count, for_each, depends_on
  vpc_id = module.vpc.vpc_id
}

module "net" {
  source = "git::https://github.com/org/tf-vpc.git?ref=v1.2.0"   # Git, pin a tag
}
```

| Convention | Detail |
|---|---|
| Files | `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, `README.md` |
| Providers | `required_providers` in modules, `provider` blocks only in roots |
| Interface | Every variable and output has a `description`, variables have `type` |
| Versioning | Semantic version tags, pin consumers, upgrade dev first |
| Referencing outputs | `module.<name>.<output>` |
| Update modules | `terraform init -upgrade` or `terraform get -update` |

Registry source format is `<namespace>/<name>/<provider>`. Registry modules take `version`, Git modules take `?ref=`.

## Backends and locking

```hcl
terraform {
  backend "s3" {
    bucket       = "tfstate-123456789012-ap-south-1"
    key          = "web-platform/prod/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true            # native S3 locking, Terraform 1.11+
    # kms_key_id = "alias/tfstate"
    # dynamodb_table = "tf-locks"  # legacy locking, deprecated in favour of use_lockfile
  }
}
```

| Item | Detail |
|---|---|
| Lock object | `<key>.tflock` next to the state file |
| Needed S3 permissions | `s3:ListBucket`, `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` (the lock file is written and deleted) |
| Bucket hardening | Versioning on, SSE on, public access block, TLS-only policy |
| Cross-account access | Assume a role: `assume_role = { role_arn = "..." }` inside the backend block |
| Reading another stack | `data "terraform_remote_state"` (exposes all outputs) or SSM parameters |
| Backends cannot use variables | Use `-backend-config` or a `backend.hcl` file |

```bash
terraform init -backend-config=backend.hcl
terraform init -migrate-state          # move local state to the new backend
```

## Import, move and remove

```hcl
import {
  to = aws_s3_bucket.legacy
  id = "my-existing-bucket"
}

import {
  for_each = var.subnet_ids
  to       = aws_subnet.imported[each.key]
  id       = each.value
}

moved {
  from = aws_instance.web
  to   = module.web.aws_instance.this
}

removed {
  from = aws_instance.old
  lifecycle { destroy = false }       # drop from state, keep in AWS
}
```

```bash
terraform plan -generate-config-out=generated.tf     # draft code for import blocks
terraform import aws_s3_bucket.legacy my-existing-bucket   # legacy CLI form
terraform apply -replace=aws_instance.web            # recreate one resource
```

## Environment variables

| Variable | Meaning |
|---|---|
| `TF_VAR_name=value` | Set input variable `name` |
| `TF_LOG=TRACE\|DEBUG\|INFO\|WARN\|ERROR` | Enable logging |
| `TF_LOG_PATH=./tf.log` | Write logs to a file |
| `TF_IN_AUTOMATION=true` | Trim "next steps" hints in output |
| `TF_INPUT=false` | Same as `-input=false` |
| `TF_CLI_ARGS_plan="-lock-timeout=60s"` | Default args for a subcommand |
| `TF_PLUGIN_CACHE_DIR=$HOME/.terraform.d/plugin-cache` | Share provider downloads between projects |
| `TF_WORKSPACE=prod` | Select a workspace |
| `AWS_PROFILE`, `AWS_REGION` | AWS credentials and region |

Variable precedence, lowest to highest: defaults, `TF_VAR_*`, `terraform.tfvars`, `*.auto.tfvars` (alphabetical), then `-var` and `-var-file` in command-line order.

## Quality gates

```bash
terraform fmt -check -recursive -diff
terraform validate
tflint --init && tflint --recursive --config "$(pwd)/.tflint.hcl"
checkov -d . --framework terraform --compact --quiet
checkov -d . -c CKV_AWS_79,CKV_AWS_8          # run selected checks
checkov -d . --skip-check CKV_AWS_91          # skip with a reason in review
trivy config .                                 # alternative IaC scanner
terraform-docs markdown table . > README.md    # generate module docs
infracost breakdown --path .                   # cost estimate
```

| Tool | Catches |
|---|---|
| `fmt` | Style drift |
| `validate` | Syntax, wrong argument names, bad references |
| `tflint` | Invalid instance types, deprecated syntax, unused declarations, naming |
| `checkov` / `trivy` | Insecure settings: open security groups, unencrypted storage, IMDSv1 |
| `conftest` (OPA) / Sentinel | Organisation rules evaluated on the plan JSON |
| `infracost` | Cost change per PR |
| `pre-commit` | Runs the above before the commit lands |

Inline skip for one resource: `#checkov:skip=CKV_AWS_260:reason`.

## Testing

```hcl
# tests/basic.tftest.hcl
mock_provider "aws" {}

variables {
  name = "demo"
}

run "encryption_enabled" {
  command = plan
  assert {
    condition     = aws_s3_bucket_server_side_encryption_configuration.this.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm == "AES256"
    error_message = "Bucket must be encrypted."
  }
}

run "rejects_bad_name" {
  command = plan
  variables { name = "UPPER" }
  expect_failures = [var.name]
}
```

```bash
terraform test                         # all tests
terraform test -filter=tests/basic.tftest.hcl
terraform test -verbose
```

## GitHub Actions snippets

```yaml
permissions:
  id-token: write       # required for OIDC
  contents: read

steps:
  - uses: actions/checkout@v4
  - uses: hashicorp/setup-terraform@v3
    with: { terraform_version: "1.13.x", terraform_wrapper: false }
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: ${{ vars.AWS_PLAN_ROLE_ARN }}
      aws-region: ap-south-1
  - run: terraform init -input=false -backend-config="bucket=${{ vars.TF_STATE_BUCKET }}"
  - run: terraform plan -input=false -no-color -detailed-exitcode -out=tfplan
```

OIDC trust `sub` values: `repo:ORG/REPO:pull_request`, `repo:ORG/REPO:ref:refs/heads/main`, `repo:ORG/REPO:environment:prod`. Gate prod with a GitHub environment that has required reviewers, and serialise runs with `concurrency: { group: tf-prod, cancel-in-progress: false }`.

## Debugging

| Symptom | Command or action |
|---|---|
| Need provider API detail | `TF_LOG=DEBUG terraform plan 2> debug.log` |
| Unsure what an expression returns | `terraform console` then type the expression |
| Plan wants to replace something unexpectedly | Read the line marked `# forces replacement`; check `ignore_changes` or a renamed attribute |
| `Error acquiring the state lock` | Check who holds it in the message, then `terraform force-unlock <ID>` only if dead |
| `Saved plan is stale` | Create a new plan, state changed after it was saved |
| Resource exists in AWS but not in state | `import` block |
| Resource in state but deleted in AWS | `terraform plan` shows re-create; or `state rm` if intentionally gone |
| Provider hash mismatch | `terraform providers lock -platform=...` and commit the lock file |
| Slow plans | Split state, raise `-parallelism`, use `-target` only to diagnose |
| Cycle error | Break the circular reference, often by using separate rule resources |

## Version constraints

| Constraint | Meaning |
|---|---|
| `= 6.1.0` or `6.1.0` | Exactly this version |
| `>= 6.0` | This or newer (includes next major, risky) |
| `~> 6.0` | `>= 6.0, < 7.0` (allows minor and patch) |
| `~> 6.1.0` | `>= 6.1.0, < 6.2.0` (patch only) |
| `>= 6.0, < 7.0` | Explicit range |

Rule of thumb: roots use `~>` on providers and commit the lock file, modules use `>=` lower bounds so consumers can choose.

## Top 15 interview one-liners

1. **What is Terraform?** A declarative, provider-based tool that builds a dependency graph from your config, diffs it against state, and calls cloud APIs to make reality match.
2. **Why state?** It maps code addresses to real resource IDs and caches attributes, so Terraform knows what it owns and what to change.
3. **Why remote state with locking?** Teams need one shared source of truth and a lock so two applies cannot corrupt it. On S3 use versioning, encryption and `use_lockfile`.
4. **`plan` versus `apply`?** `plan` is a read-only diff. `apply` executes it. In CI we save the plan with `-out` and apply that exact file.
5. **`count` versus `for_each`?** `count` addresses by index and shifts when items are removed. `for_each` uses stable keys, so we default to it.
6. **How do you handle secrets?** Keep them out of code and Git, mark inputs `sensitive`, fetch from Secrets Manager or SSM at runtime, and remember they can still appear in state, so lock state down.
7. **How do you structure environments?** Separate root modules per environment sharing versioned modules, each with its own state key, rather than workspaces for dev and prod.
8. **What is drift and how do you find it?** Real infrastructure differing from state or code. `plan -refresh-only` shows it, and a scheduled plan with `-detailed-exitcode` alerts on it.
9. **How do you adopt existing infrastructure?** An `import` block plus `plan -generate-config-out`, then clean the code until the plan shows no changes.
10. **How do you rename or refactor without downtime?** Use `moved` blocks so the plan shows a move instead of destroy and create.
11. **What does `terraform destroy` actually do, and how do you protect against it?** It deletes everything in that state. Use `prevent_destroy`, deletion protection on resources, scoped IAM, and approval gates.
12. **What do `tflint` and `checkov` do?** `tflint` validates provider-specific correctness and conventions. `checkov` scans for insecure configuration. Both run on every PR before the plan is read.
13. **How does CI authenticate to AWS?** OIDC federation with short-lived tokens and separate roles: read-only for PR plans, scoped write for apply restricted to a protected environment.
14. **How do you version and share modules?** Semantic version tags in Git or a registry, pinned by consumers, upgraded dev first, and `moved` blocks to keep upgrades non-destructive.
15. **A prod apply failed halfway. What now?** Do not panic and do not re-run blindly. Read the error, check state with `state list` and the console, fix the cause, and run `plan` again. Terraform is idempotent, and the bucket's state versioning is the safety net if state itself is bad.

---

[⬅ Back to Terraform & IaC](./README.md) · [🏠 Home](../README.md)

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
