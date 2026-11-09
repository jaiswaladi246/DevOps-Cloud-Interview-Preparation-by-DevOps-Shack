# Terraform — Core Concepts

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

> Core concepts, internals and comparisons for Terraform and Infrastructure as Code · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Terraform & IaC](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [IaC and Terraform Fundamentals](#iac-and-terraform-fundamentals) — Q1–Q25
- [Workflow and CLI](#workflow-and-cli) — Q26–Q45
- [Providers, Resources and Data Sources](#providers-resources-and-data-sources) — Q46–Q70
- [Variables, Locals, Outputs and Types](#variables-locals-outputs-and-types) — Q71–Q100
- [Loops, Conditionals, Dynamic Blocks and Functions](#loops-conditionals-dynamic-blocks-and-functions) — Q101–Q135
- [Modules](#modules) — Q136–Q155
- [State Internals](#state-internals) — Q156–Q180
- [Lifecycle, Dependencies, Provisioners and Drift](#lifecycle-dependencies-provisioners-and-drift) — Q181–Q200

---

## IaC and Terraform Fundamentals

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What is Infrastructure as Code and why do teams use it? 🟢

<details>
<summary><b>Answer</b></summary>

IaC means describing infrastructure (networks, VMs, databases, IAM) in version-controlled files and letting a tool create it, instead of clicking in a console. You get repeatability, code review, an audit trail in git, easy rebuilds of environments and a way to detect drift. In production we treat an environment that cannot be recreated from code as a liability.

</details>

### Q2. What is the difference between declarative and imperative infrastructure tools? 🟢

<details>
<summary><b>Answer</b></summary>

Declarative tools (Terraform, CloudFormation) describe the desired end state and the tool works out the steps. Imperative tools (shell scripts, AWS CLI, many Ansible playbooks) describe the steps to run in order. Declarative is easier to re-run safely because the tool diffs desired against actual; imperative gives more control but you own idempotency yourself.

</details>

### Q3. What is Terraform and what problem does it solve? 🟢

<details>
<summary><b>Answer</b></summary>

Terraform is an open-source (BSL-licensed since 1.6) IaC tool from HashiCorp that provisions infrastructure through provider plugins for AWS, Azure, GCP, Kubernetes, Datadog, GitHub and thousands more. One workflow (`init`, `plan`, `apply`) works across all of them, so you can manage cloud resources, DNS and SaaS from the same codebase.

</details>

### Q4. What is HCL and can Terraform use anything else? 🟢

<details>
<summary><b>Answer</b></summary>

HCL (HashiCorp Configuration Language) is the human-friendly syntax of `.tf` files, built from blocks, arguments and expressions. Terraform also reads a JSON variant (`.tf.json`), which is handy for machine-generated config. Both are equivalent, but nearly everything you will see is HCL.
```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "acme-logs-prod"
}
```

</details>

### Q5. Which files make up a typical Terraform root module? 🟢

<details>
<summary><b>Answer</b></summary>

Terraform loads every `.tf` file in the working directory as one module, so file names are convention only. We usually split into `main.tf` (resources), `variables.tf`, `outputs.tf`, `providers.tf` or `versions.tf` (the `terraform` block and providers), `locals.tf`, and `terraform.tfvars` for values. Subdirectories are not loaded unless called as modules.

</details>

### Q6. How does Terraform work internally? 🔴

<details>
<summary><b>Answer</b></summary>

Terraform Core reads the configuration and state, builds a dependency graph of resources, and walks it in parallel. For each resource it calls a provider plugin, a separate binary that talks to Core over gRPC and translates CRUD operations into API calls. Core never knows how to create an EC2 instance; the AWS provider does. That split is why providers version independently of Terraform.

</details>

### Q7. What is a provider in Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

A provider is a plugin that exposes resource types and data sources for one API, such as `hashicorp/aws` or `hashicorp/kubernetes`. You declare it in `required_providers`, configure it in a `provider` block (region, credentials) and `terraform init` downloads it from the registry into `.terraform/providers`.

</details>

### Q8. What is the difference between mutable and immutable infrastructure? 🟡

<details>
<summary><b>Answer</b></summary>

Mutable means you change servers in place (patch, reconfigure over SSH). Immutable means you replace them with a new image or instance whenever something changes. Terraform leans immutable: many attribute changes show as `-/+` (destroy and recreate), and we bake AMIs with Packer rather than patching live nodes. It reduces snowflake servers and drift.

</details>

### Q9. What does idempotency mean in the context of Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

Running `terraform apply` repeatedly with unchanged config and unchanged real infrastructure produces no changes after the first run. Terraform achieves that by comparing configuration, state and refreshed reality. If a second apply immediately after the first still shows a diff, something is wrong, often a provider bug or a normalised value such as an IAM policy JSON.

</details>

### Q10. How does Terraform compare with CloudFormation, Pulumi, Ansible and CDK? 🟡

<details>
<summary><b>Answer</b></summary>

CloudFormation is AWS-only and managed by AWS (no state file to handle, but weaker cross-provider reach). Pulumi and CDK for Terraform use general-purpose languages. Ansible is procedural config management that can provision but has no real state or plan. Terraform's edge is a multi-cloud provider ecosystem, a readable plan and a large module registry; the trade-off is that you manage state yourself.

</details>

### Q11. How does Terraform decide the order in which to create and destroy resources? 🔴

<details>
<summary><b>Answer</b></summary>

It builds a directed acyclic graph from references: if `aws_instance.web` uses `aws_subnet.a.id`, the subnet is created first. Independent nodes run in parallel (default `-parallelism=10`). Destroy walks the graph in reverse. Implicit edges come from expressions, explicit ones from `depends_on`. A circular reference fails with `Error: Cycle: ...`.

</details>

### Q12. What is the Terraform state file and why does it exist? 🟢

<details>
<summary><b>Answer</b></summary>

State (`terraform.tfstate`) is a JSON file mapping each resource in your config to the real object ID in the cloud, plus attributes and metadata. Terraform needs it to know what it already manages, to compute diffs, to order destroys and to avoid slow lookups of everything on every plan. Without state, Terraform cannot tell a resource it created from one that already existed.

</details>

### Q13. Why did OpenTofu appear and how does it differ from Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

In August 2023 HashiCorp moved Terraform from MPL 2.0 to the Business Source License. The community forked the last MPL version as OpenTofu, now under the Linux Foundation. Config and state are mostly compatible. OpenTofu adds features such as native state encryption, `for_each` on providers, early variable evaluation in backends and the `-exclude` flag; Terraform has its own additions like ephemeral resources and HCP-centric features.

</details>

### Q14. Can you migrate between Terraform and OpenTofu? 🟡

<details>
<summary><b>Answer</b></summary>

For Terraform up to roughly 1.5 to 1.6 with standard config it is mostly a binary swap: back up state, install `tofu`, run `tofu init` and `tofu plan` and check for no diff. Providers resolve from `registry.opentofu.org`. Newer Terraform-only features (such as ephemeral resources or write-only attributes) may not parse, so check compatibility first and migrate non-prod before prod.

</details>

### Q15. What does the `terraform` block do? 🟢

<details>
<summary><b>Answer</b></summary>

It holds settings for Terraform itself: `required_version`, `required_providers` and the `backend` or `cloud` block.
```hcl
terraform {
  required_version = ">= 1.9.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
```

</details>

### Q16. How do version constraints like `~>`, `>=` and `!=` work? 🟡

<details>
<summary><b>Answer</b></summary>

`= 1.2.3` pins exactly; `>= 1.2` sets a floor; `~> 5.1` allows 5.x where x is 1 or higher (the rightmost digit may increase); `~> 5.1.0` allows only 5.1.x; `!= 5.3.0` excludes a bad release. Constraints can be combined with commas. For modules and providers we use `~>` to take patches and minors but not majors.

</details>

### Q17. What is `.terraform.lock.hcl` and should it be committed? 🟡

<details>
<summary><b>Answer</b></summary>

The dependency lock file records the exact provider versions and checksums selected by `terraform init`. Commit it, so every engineer and CI run uses identical providers. Update it deliberately with `terraform init -upgrade`. For multi-OS teams run `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64` so hashes for all platforms are present.

</details>

### Q18. What is in the `.terraform` directory and what belongs in `.gitignore`? 🟢

<details>
<summary><b>Answer</b></summary>

It holds downloaded providers, downloaded modules and backend configuration cache. Ignore `.terraform/`, `*.tfstate`, `*.tfstate.*`, `crash.log`, `*.tfplan` and any `*.tfvars` that contain secrets. Commit `.terraform.lock.hcl`, all `.tf` files and non-secret tfvars.

</details>

### Q19. What exactly happens during `terraform plan`? 🔴

<details>
<summary><b>Answer</b></summary>

Terraform reads state, refreshes each managed resource by calling the provider's read function, then compares the refreshed state with the configuration to propose a diff. Providers validate the config (`ValidateResourceConfig`) and compute `PlanResourceChange`, which marks attributes that depend on unknown inputs as "known after apply". The result is a plan graph; nothing is changed remotely.

</details>

### Q20. What are the main block types in HCL? 🟢

<details>
<summary><b>Answer</b></summary>

`terraform`, `provider`, `resource`, `data`, `variable`, `locals`, `output`, `module`, plus newer ones: `moved`, `import`, `removed`, `check` and `ephemeral`. Nested blocks inside resources (such as `ingress` in a security group) are provider-defined. Each top-level block has labels, for example `resource "aws_vpc" "main"` has type and name.

</details>

### Q21. How do you reference values in Terraform expressions? 🟢

<details>
<summary><b>Answer</b></summary>

`var.name` for variables, `local.name` for locals, `aws_vpc.main.id` for resource attributes, `data.aws_ami.al2023.id` for data sources, `module.network.vpc_id` for module outputs, `each.key`/`each.value`, `count.index`, `path.module`, and `terraform.workspace`. These references create the dependency graph automatically.

</details>

### Q22. How do you write comments and multi-line strings in HCL? 🟢

<details>
<summary><b>Answer</b></summary>

Use `#` or `//` for single-line comments and `/* ... */` for blocks. Multi-line strings use heredocs; `<<-EOT` strips leading indentation.
```hcl
user_data = <<-EOT
  #!/bin/bash
  dnf install -y nginx
EOT
```

</details>

### Q23. What is the Terraform Registry? 🟡

<details>
<summary><b>Answer</b></summary>

registry.terraform.io is the public index of providers, modules and policy libraries. Providers are namespaced (`hashicorp/aws`, `integrations/github`). Modules follow `namespace/name/provider`, for example `terraform-aws-modules/vpc/aws`. Verified and official badges help, but we still review module source before using it. Private registries exist in HCP Terraform and Terraform Enterprise.

</details>

### Q24. What is the difference between a root module and a child module? 🟡

<details>
<summary><b>Answer</b></summary>

The root module is the directory where you run Terraform commands. Every other module is a child, called with a `module` block from the root or from another module. A module is just a folder of `.tf` files. Only the root can configure backends and, ideally, providers; children receive providers from the caller.

</details>

### Q25. How do provider plugins implement a resource, and what are SDKv2 and the Plugin Framework? 🔴

<details>
<summary><b>Answer</b></summary>

A provider registers a schema per resource and implements create, read, update and delete functions. Terraform Core sends a prior state, proposed new state and config; the provider returns the planned and applied state. SDKv2 is the older Go SDK; the Plugin Framework is the current one with better support for null, unknown values and nested attributes. Both speak the same plugin protocol (v5 or v6), so Core is agnostic.

</details>

## Workflow and CLI

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. What does `terraform init` do? 🟢

<details>
<summary><b>Answer</b></summary>

It prepares a working directory: configures the backend, downloads required providers and modules, and writes `.terraform.lock.hcl`. It is safe to re-run. Run it after adding a provider, changing module sources or switching backends.
```bash
terraform init
terraform init -upgrade   # re-resolve provider and module versions
```

</details>

### Q27. What does `terraform plan` do? 🟢

<details>
<summary><b>Answer</b></summary>

It computes and displays what Terraform would change to make real infrastructure match the configuration, without changing anything. Output uses `+` create, `-` destroy, `~` update in place and `-/+` replace. We always read the summary line (`Plan: 3 to add, 1 to change, 0 to destroy`) before applying.

</details>

### Q28. What does `terraform apply` do and what is `-auto-approve`? 🟢

<details>
<summary><b>Answer</b></summary>

`apply` runs a plan and, after you type `yes`, executes it, then updates state. `-auto-approve` skips the prompt. Use it only in pipelines that applied a reviewed saved plan, never in a shell against production out of habit.

</details>

### Q29. What does `terraform destroy` do? 🟢

<details>
<summary><b>Answer</b></summary>

It removes every resource tracked in the current state; it is equivalent to `terraform apply -destroy`. Run `terraform plan -destroy` first to preview. Resources with `prevent_destroy = true` make it fail, and some (non-empty S3 buckets) fail unless `force_destroy` is set.

</details>

### Q30. What do `terraform fmt` and `terraform validate` check? 🟢

<details>
<summary><b>Answer</b></summary>

`fmt` rewrites files into canonical style (use `fmt -check -recursive` in CI). `validate` checks syntax, types and references offline, without contacting the cloud or reading state. It needs `init` first because it needs provider schemas.

</details>

### Q31. Why save a plan with `-out` and apply that file? 🟡

<details>
<summary><b>Answer</b></summary>

`terraform plan -out=tfplan` freezes exactly what was reviewed. `terraform apply tfplan` applies those changes without re-planning, so what was approved is what runs. If state changed in the meantime Terraform rejects it as stale. Plan files can contain secrets in plaintext, so store them as protected pipeline artifacts.

</details>

### Q32. What does `-detailed-exitcode` do? 🟡

<details>
<summary><b>Answer</b></summary>

With `terraform plan -detailed-exitcode`, exit code 0 means no changes, 1 means error and 2 means changes are present. We use it in scheduled drift-detection jobs: exit 2 raises an alert.
```bash
terraform plan -detailed-exitcode -input=false -lock=false; echo $?
```

</details>

### Q33. How do you read the symbols in a plan? 🟡

<details>
<summary><b>Answer</b></summary>

`+` create, `-` destroy, `~` in-place update, `-/+` destroy then create (replace), `+/-` create then destroy (when `create_before_destroy` is set), `<=` data source read, and `(known after apply)` for values not yet computed. Attributes marked `# forces replacement` are what turns an update into a replace.

</details>

### Q34. Why is `-target` considered dangerous? 🔴

<details>
<summary><b>Answer</b></summary>

`-target` limits the plan to chosen resources and their dependencies, ignoring everything else. State and reality can drift from the code, and Terraform warns that it is "not for routine use". It is for recovery cases such as breaking a dependency cycle or applying a broken resource first. If you use it routinely, your root module is too big.
```bash
terraform apply -target='module.network.aws_subnet.private["a"]'
```

</details>

### Q35. What is the difference between `taint` and `-replace`? 🟡

<details>
<summary><b>Answer</b></summary>

`terraform taint` marked a resource in state so the next apply recreates it and is deprecated. `terraform apply -replace='aws_instance.web'` does the same in one step and shows the replacement in the plan before you approve it. Prefer `-replace`.

</details>

### Q36. What is `terraform apply -refresh-only` and how does it relate to `terraform refresh`? 🟡

<details>
<summary><b>Answer</b></summary>

`-refresh-only` reads real infrastructure and proposes updating state to match it, without changing resources, and lets you review the diff. The old `terraform refresh` did the same unreviewed and is deprecated because it could overwrite state silently, for instance after a credentials mistake.

</details>

### Q37. How do you view outputs and the current state? 🟢

<details>
<summary><b>Answer</b></summary>

`terraform output` prints root outputs (`-json` or `-raw` for scripting). `terraform show` renders the state or a saved plan; `terraform show -json tfplan` gives machine-readable JSON for policy tools. `terraform state list` lists addresses.
```bash
terraform output -raw db_endpoint
terraform show -json tfplan | jq '.resource_changes[].change.actions'
```

</details>

### Q38. What is `terraform console` useful for? 🟢

<details>
<summary><b>Answer</b></summary>

It is a REPL that evaluates expressions against current config and state, handy for testing functions and `for` expressions before committing them.
```bash
$ terraform console
> cidrsubnet("10.0.0.0/16", 8, 3)
"10.0.3.0/24"
> [for s in ["a","b"] : upper(s)]
```

</details>

### Q39. Which `terraform init` flags matter and when do you use them? 🟡

<details>
<summary><b>Answer</b></summary>

`-upgrade` re-resolves provider and module versions; `-reconfigure` ignores previous backend config; `-migrate-state` copies state to a new backend; `-backend-config=file_or_key=value` supplies partial backend config such as bucket names; `-input=false` for CI; `-lockfile=readonly` to fail rather than modify the lock file.

</details>

### Q40. Which environment variables do you use to control Terraform? 🟡

<details>
<summary><b>Answer</b></summary>

`TF_LOG` (TRACE, DEBUG, INFO, WARN, ERROR) with `TF_LOG_PATH` for debugging, `TF_VAR_name` for variables, `TF_IN_AUTOMATION=1` to trim hints in CI, `TF_INPUT=0` to disable prompts, `TF_CLI_ARGS_plan` for default flags, `TF_DATA_DIR` to relocate `.terraform`, and `TF_PLUGIN_CACHE_DIR` to share provider downloads.

</details>

### Q41. What does `terraform graph` give you? 🟡

<details>
<summary><b>Answer</b></summary>

It prints the dependency graph in DOT format, rendered by Graphviz.
```bash
terraform graph | dot -Tsvg > graph.svg
terraform graph -type=plan-destroy
```
We use it to debug unexpected ordering or a `Cycle` error, though large graphs are unreadable without filtering.

</details>

### Q42. What happens if `terraform apply` fails halfway? 🔴

<details>
<summary><b>Answer</b></summary>

There is no rollback. Resources that succeeded stay created and are recorded in state; failed ones are not (or are marked tainted if creation partly succeeded). Re-running `apply` continues from where it stopped. If the state backend itself fails to save, Terraform writes `errored.tfstate` locally; recover it with `terraform state push errored.tfstate`.

</details>

### Q43. What is the difference between `terraform validate` and `terraform plan`? 🟢

<details>
<summary><b>Answer</b></summary>

`validate` is static and offline: syntax, argument names, type consistency. `plan` talks to the provider and cloud, reads state and finds problems validate cannot, such as a nonexistent AMI, expired credentials or an invalid CIDR. Run validate on every commit for fast feedback and plan on pull requests.

</details>

### Q44. Which flags make `apply` safer in automation? 🟡

<details>
<summary><b>Answer</b></summary>

`-input=false` (never prompt), `-lock-timeout=5m` (wait for the lock rather than failing), `-parallelism=N` (tune concurrency against API rate limits), and applying a saved plan file instead of `-auto-approve` on a fresh plan. Combine with `TF_IN_AUTOMATION=1`.

</details>

### Q45. What are "known after apply" and unknown values, and why do they limit what you can do? 🔴

<details>
<summary><b>Answer</b></summary>

Unknown values are attributes the provider cannot compute until the resource exists, such as an instance ID. Terraform tracks them through the plan, but cannot use them where it must know the value to build the graph, such as `count` or `for_each` keys, or resource addresses. That is why `for_each = toset(aws_instance.x[*].id)` fails and you key by static values instead.

</details>

## Providers, Resources and Data Sources

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q46. What is a resource block and how is it addressed? 🟢

<details>
<summary><b>Answer</b></summary>

A `resource` block declares one infrastructure object. The first label is the type (provider prefix plus kind), the second a local name; together they form the address `aws_instance.web`.
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.micro"
}
```

</details>

### Q47. What is the difference between a resource and a data source? 🟢

<details>
<summary><b>Answer</b></summary>

A resource is something Terraform creates and manages in its lifecycle. A data source (`data` block) only reads existing information, for example an AMI, a VPC created elsewhere or the current account ID. Data sources never create or destroy anything and are re-read on every plan.

</details>

### Q48. How do you configure a provider such as AWS? 🟢

<details>
<summary><b>Answer</b></summary>

Declare it in `required_providers` and add a `provider` block for settings like region. Credentials should come from the environment, shared config, SSO or an assumed role, never from literals in code.
```hcl
provider "aws" {
  region = "ap-south-1"
  default_tags { tags = { ManagedBy = "terraform" } }
}
```

</details>

### Q49. How do you use the same provider with multiple regions or accounts? 🟡

<details>
<summary><b>Answer</b></summary>

Add a second `provider` block with an `alias`, then select it with the `provider` meta-argument on resources or pass it to modules.
```hcl
provider "aws" { alias = "use1"  region = "us-east-1" }

resource "aws_acm_certificate" "cf" {
  provider    = aws.use1
  domain_name = "example.com"
  validation_method = "DNS"
}
```

</details>

### Q50. In what order does the AWS provider look for credentials? 🟡

<details>
<summary><b>Answer</b></summary>

Arguments in the provider block first, then environment variables (`AWS_ACCESS_KEY_ID`, `AWS_PROFILE`), then shared credentials and config files, then container or EC2 instance metadata roles. In CI we use OIDC to assume a role, so no long-lived keys exist anywhere. `assume_role` in the provider block handles cross-account access.

</details>

### Q51. What does `default_tags` in the AWS provider do and what are its pitfalls? 🟡

<details>
<summary><b>Answer</b></summary>

It applies a tag set to every taggable resource created by that provider, so you do not repeat tags on each resource. Resource-level tags merge over defaults. Pitfall: tags appear in `tags_all` and old provider versions showed perpetual diffs if the same key was set in both places with different values; Auto Scaling groups need explicit `tag` blocks.

</details>

### Q52. What is the difference between arguments and attributes? 🟢

<details>
<summary><b>Answer</b></summary>

Arguments are inputs you set in a block (`instance_type = "t3.micro"`). Attributes are all the values you can read from a resource, including arguments and computed values such as `id`, `arn` or `public_ip`. The provider docs list them under "Argument Reference" and "Attribute Reference".

</details>

### Q53. When is a data source read, and what happens if its arguments are unknown? 🔴

<details>
<summary><b>Answer</b></summary>

Normally during plan, so its result is available for planning. If an argument depends on a resource not yet created (unknown), Terraform defers the read to apply time and every attribute becomes "known after apply", which can ripple into larger plans. Avoid depending a data source on resources from the same apply when you can.

</details>

### Q54. Why is `aws_ami` with `most_recent = true` a risky pattern? 🟡

<details>
<summary><b>Answer</b></summary>

A new AMI published upstream changes the data source result, so the next plan wants to replace running instances. For production we pin an AMI ID through a variable or SSM parameter, or add `ignore_changes = [ami]` and roll instances through a controlled process such as an ASG instance refresh.

</details>

### Q55. Why do some attribute changes force a replacement? 🟡

<details>
<summary><b>Answer</b></summary>

The provider marks certain attributes as ForceNew because the cloud API cannot modify them in place (for example `availability_zone` on an EBS volume or `name` on many resources). The plan shows `# forces replacement` next to them. Always read plans for `-/+` on stateful resources like databases.

</details>

### Q56. What does "Optional + Computed" mean for an argument? 🔴

<details>
<summary><b>Answer</b></summary>

If you omit the argument, the provider picks a value and records it in state; if you set it, your value wins. Example: `aws_instance.availability_zone`. This is why leaving an argument out does not produce a diff, while later removing a value you previously set usually does not revert it to the default.

</details>

### Q57. How do nested blocks differ from map or list attributes in resource syntax? 🟡

<details>
<summary><b>Answer</b></summary>

Nested blocks (`ingress { ... }`) are repeatable, schema-defined sub-objects; attributes take `=` and an expression. Some providers moved from blocks to attributes, which changes syntax. Only blocks can be generated with `dynamic`; for attribute-style lists you use `for` expressions instead.

</details>

### Q58. What are `terraform_data` and `null_resource`? 🟡

<details>
<summary><b>Answer</b></summary>

Both are resources that do nothing themselves but hold values or attach provisioners and triggers. `terraform_data` (built in, Terraform 1.4+) replaces `null_resource` from the `null` provider; it has `input` and `triggers_replace` to force recreation when a value changes.
```hcl
resource "terraform_data" "bootstrap" {
  triggers_replace = [aws_instance.web.id]
  provisioner "local-exec" { command = "./register.sh" }
}
```

</details>

### Q59. What should you know before using the `random` provider? 🟡

<details>
<summary><b>Answer</b></summary>

`random_password` and `random_id` generate a value once and store it in state, so state must be protected. Values change only if a `keepers` map changes or you replace the resource. Use `random_password` with `special = false` when a target system disallows certain characters, and never print it in outputs unmarked.

</details>

### Q60. Where does `hashicorp/aws` come from in `required_providers`? 🟢

<details>
<summary><b>Answer</b></summary>

The source address format is `[hostname/]namespace/type`. If you omit the hostname Terraform uses `registry.terraform.io`, so `hashicorp/aws` means `registry.terraform.io/hashicorp/aws`. Third-party providers use their own namespace, for example `integrations/github` or `cloudflare/cloudflare`. In OpenTofu the default host is `registry.opentofu.org`.

</details>

### Q61. How do you install providers in an air-gapped or rate-limited environment? 🟡

<details>
<summary><b>Answer</b></summary>

Use a filesystem mirror (`terraform providers mirror ./mirror`) or a network mirror configured in `~/.terraformrc` under `provider_installation`, and set `plugin_cache_dir` to avoid re-downloading in CI.
```hcl
provider_installation {
  filesystem_mirror { path = "/opt/terraform/mirror"  include = ["hashicorp/*"] }
  direct { exclude = ["hashicorp/*"] }
}
```

</details>

### Q62. What is the `timeouts` block? 🟡

<details>
<summary><b>Answer</b></summary>

Many resources accept `timeouts { create = "60m" ... }` to extend how long Terraform waits for the API, essential for slow resources like RDS, EKS clusters and CloudFront. It is resource-specific, so check each resource's docs. It does not retry failures; it only extends the wait.

</details>

### Q63. What are provider-defined functions? 🟡

<details>
<summary><b>Answer</b></summary>

Since Terraform 1.8, providers can ship functions callable as `provider::<name>::<function>`, for example `provider::aws::arn_parse(...)`. They need the provider declared in `required_providers`. They are pure and run at plan time without API calls, which makes them good for parsing ARNs or encoding values safely.

</details>

### Q64. Why is it problematic to configure a provider with attributes of resources created in the same configuration? 🔴

<details>
<summary><b>Answer</b></summary>

Terraform needs provider configuration before it can plan resources of that provider. If you feed the Kubernetes or Helm provider from an EKS cluster created in the same apply, the first plan sees unknown values and can fail, or fails on destroy because the cluster disappears before its resources are removed. The fix is separate states (cluster layer, then add-ons layer) or a data source on an already-existing cluster.

</details>

### Q65. What does the `provider` meta-argument on a resource do? 🟡

<details>
<summary><b>Answer</b></summary>

It selects which configuration of a provider the resource uses, as in `provider = aws.use1`. Without it, the default (unaliased) configuration is used. Modules receive their providers via the `providers` map in the module block.

</details>

### Q66. How do you address a specific instance of a resource with `count` or `for_each`? 🟢

<details>
<summary><b>Answer</b></summary>

With an index or key in brackets: `aws_instance.web[0]` for count, `aws_subnet.private["a"]` for for_each; inside modules prefix `module.vpc.`. Quote carefully in shells.
```bash
terraform state show 'module.vpc.aws_subnet.private["ap-south-1a"]'
```

</details>

### Q67. What is `terraform_remote_state` and what are its drawbacks? 🟡

<details>
<summary><b>Answer</b></summary>

It reads root outputs from another configuration's state, so a stack can consume a VPC ID from the network stack. Drawbacks: it needs read access to the entire state (which may contain secrets), it couples consumers to the producer's backend layout, and a renamed output breaks them. We often publish values to SSM Parameter Store or use data sources by tag instead.

</details>

### Q68. How does Terraform deal with eventual consistency and API throttling? 🔴

<details>
<summary><b>Answer</b></summary>

Providers wrap many calls in retry logic with backoff, for example waiting for an IAM role to propagate. For rate limits, lower `-parallelism`, set `max_retries` on the AWS provider and split very large states. Persistent `Throttling` or `RequestLimitExceeded` errors at scale are a hint your state is too large.

</details>

### Q69. What is the `external` data source and why avoid it? 🟡

<details>
<summary><b>Answer</b></summary>

`data "external"` runs a program that reads JSON on stdin and returns a JSON map of strings on stdout. It runs on every plan on whoever's machine runs it, so it is hard to reproduce, requires tools installed on the runner and bypasses typed providers. Use it only when no provider exists, and keep the script deterministic.

</details>

### Q70. What does "Provider produced inconsistent result after apply" mean? 🔴

<details>
<summary><b>Answer</b></summary>

It is a provider bug: after create or update, the value returned for an attribute differs from the value Terraform planned as known. Core enforces that known planned values do not change. It is not something you can fix in your config; upgrade or pin the provider version, or file an issue. Often a retry or later provider release resolves it.

</details>

## Variables, Locals, Outputs and Types

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q71. How do you declare an input variable? 🟢

<details>
<summary><b>Answer</b></summary>

A `variable` block with an optional `type`, `default`, `description`, `validation`, `sensitive` and `nullable`. Without a default the variable is required.
```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance size"
  default     = "t3.micro"
}
```

</details>

### Q72. In what order does Terraform load variable values? 🟢

<details>
<summary><b>Answer</b></summary>

From lowest to highest precedence: defaults, environment variables (`TF_VAR_`), `terraform.tfvars`, `terraform.tfvars.json`, `*.auto.tfvars` and `*.auto.tfvars.json` (alphabetical), then `-var` and `-var-file` in the order given on the command line. Later wins. In HCP Terraform, workspace variables slot in between.

</details>

### Q73. How do you define an output and what is it for? 🟢

<details>
<summary><b>Answer</b></summary>

An `output` block exposes a value. In the root module it prints after apply and can be read by `terraform output`; in a child module it is how the caller gets values.
```hcl
output "alb_dns" {
  value       = aws_lb.main.dns_name
  description = "Public DNS of the ALB"
}
```

</details>

### Q74. What is the difference between locals and variables? 🟢

<details>
<summary><b>Answer</b></summary>

Variables are inputs set by the caller. Locals are named expressions internal to the module, computed from variables, resources and other locals. Use locals to avoid repeating an expression or to give a derived value a name, like `local.name_prefix = "${var.app}-${var.env}"`; do not use them as disguised inputs.

</details>

### Q75. What are the primitive types in Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

`string`, `number` and `bool`. Terraform converts between them automatically where unambiguous, so `"5"` can be used as a number and `true` as `"true"`. `null` represents absence of a value and is valid for any type.

</details>

### Q76. How do list, set and map differ? 🟡

<details>
<summary><b>Answer</b></summary>

A `list(T)` is ordered and allows duplicates and is indexed by position. A `set(T)` is unordered with unique values and has no index, so you cannot do `set[0]`. A `map(T)` has string keys with values of one type. `for_each` accepts a map or set of strings; convert lists with `toset()`.

</details>

### Q77. What are `object` and `tuple` types? 🟡

<details>
<summary><b>Answer</b></summary>

`object({ name = string, port = number })` is a fixed set of named attributes with individual types. `tuple([string, number])` is a fixed-length list with a type per position. Use objects for structured input; they validate shape at plan time, unlike `any` or loose maps.

</details>

### Q78. How does `optional()` work in object types? 🟡

<details>
<summary><b>Answer</b></summary>

Since 1.3 you can mark attributes optional and give defaults.
```hcl
variable "db" {
  type = object({
    engine  = string
    size_gb = optional(number, 50)
    backup  = optional(bool, true)
  })
}
```
Callers can omit `size_gb` and `backup`; without a default an omitted optional attribute is `null`.

</details>

### Q79. Why is `type = any` discouraged? 🟡

<details>
<summary><b>Answer</b></summary>

It disables validation and defers type errors until the value is used, often with confusing messages deep inside a resource. Terraform also infers one concrete type from the value passed, which can change behaviour between callers. Use precise object and map types in module interfaces.

</details>

### Q80. How does a `validation` block work? 🟡

<details>
<summary><b>Answer</b></summary>

It runs a condition on the input and fails with your message before any resource is planned.
```hcl
variable "env" {
  type = string
  validation {
    condition     = contains(["dev", "qa", "prod"], var.env)
    error_message = "env must be dev, qa or prod."
  }
}
```

</details>

### Q81. What does `sensitive = true` do on a variable or output? 🟢

<details>
<summary><b>Answer</b></summary>

It redacts the value in plan and apply output (`(sensitive value)`) and in `terraform output`. It does not encrypt anything: the value is still stored in plaintext in state and in plan files. Protect the state backend, and prefer ephemeral values or secret managers for real secrets.

</details>

### Q82. What does `nullable = false` do? 🟡

<details>
<summary><b>Answer</b></summary>

By default a caller may pass `null` to a variable, which then bypasses the default. With `nullable = false`, passing `null` is rejected or replaced by the default, so modules do not receive unexpected nulls. We set it on most scalar variables with defaults.

</details>

### Q83. Which tfvars files are loaded automatically? 🟢

<details>
<summary><b>Answer</b></summary>

`terraform.tfvars`, `terraform.tfvars.json`, and any `*.auto.tfvars` or `*.auto.tfvars.json` in the working directory. Any other file, such as `prod.tfvars`, must be passed with `-var-file=prod.tfvars`. Terraform does not load tfvars from child module directories.

</details>

### Q84. How do you pass a complex value through an environment variable? 🟡

<details>
<summary><b>Answer</b></summary>

`TF_VAR_name` takes HCL or JSON syntax for lists, maps and objects.
```bash
export TF_VAR_subnets='["10.0.1.0/24","10.0.2.0/24"]'
export TF_VAR_tags='{"team":"platform","env":"prod"}'
```
Handy in CI, though for large structures a `.tfvars.json` file is easier to review.

</details>

### Q85. How do outputs work between modules and what can you set on them? 🟡

<details>
<summary><b>Answer</b></summary>

A child module's outputs are read as `module.<name>.<output>`. An output can be `sensitive`, can have `depends_on` (rarely needed) and, since 1.2, `precondition` blocks to validate before exposing. Only outputs are visible outside a module; internal resources are not addressable from the caller.

</details>

### Q86. What are good uses of locals, and what are the limits? 🟡

<details>
<summary><b>Answer</b></summary>

Naming derived values, merging tag maps, building maps for `for_each` and avoiding repeated expressions. Locals can reference other locals but not form cycles. Overusing locals for a hundred small things makes code harder to read than a plain literal; if a value is configurable by the caller, make it a variable instead.

</details>

### Q87. How does Terraform convert between types automatically? 🔴

<details>
<summary><b>Answer</b></summary>

Primitive conversions are implicit (number to string and back, bool to string). Collections convert if element types convert, so a list of numbers can be passed where `list(string)` is expected. Tuples and objects convert to lists and maps when elements share a type. Use explicit functions (`tostring`, `tonumber`, `tolist`, `toset`, `tomap`) to make intent clear.

</details>

### Q88. What happens if a required variable has no value? 🟢

<details>
<summary><b>Answer</b></summary>

Interactive runs prompt for it; with `-input=false` (CI) it fails with `No value for required variable`. Provide it via tfvars, `-var`, or `TF_VAR_`, or give the variable a sensible default if one exists.

</details>

### Q89. Can a variable validation reference other variables? 🔴

<details>
<summary><b>Answer</b></summary>

Since Terraform 1.9 yes: `condition` and `error_message` can reference other variables, locals, data sources and resources, not only the variable being validated. Before 1.9 each validation could only reference its own variable, so cross-field rules needed `precondition` blocks on a resource or output. Check `required_version` before relying on it.

</details>

### Q90. What are ephemeral variables and outputs? 🔴

<details>
<summary><b>Answer</b></summary>

Introduced in Terraform 1.10, `ephemeral = true` on a variable or output means the value is available during the run but never persisted to state or plan files. It is intended for secrets such as tokens passed into providers or to write-only arguments. Ephemeral values can only be used in ephemeral contexts, otherwise Terraform errors.

</details>

### Q91. What are write-only arguments? 🔴

<details>
<summary><b>Answer</b></summary>

Added in Terraform 1.11, write-only arguments (named with a `_wo` suffix, such as `password_wo` on `aws_db_instance`) accept a value that Terraform sends to the provider but never stores in plan or state. Because there is no stored value to diff, you pair it with a version argument (`password_wo_version`) and bump that number to trigger a change. Support depends on the provider resource.

</details>

### Q92. How does sensitivity propagate, and what is `nonsensitive()`? 🔴

<details>
<summary><b>Answer</b></summary>

Any expression using a sensitive value yields a sensitive result, and if an output references one it must itself be `sensitive = true` or Terraform errors. `nonsensitive(value)` strips the mark when you are sure the value is safe, for example a derived hash. Use it sparingly: it makes the value visible in logs.

</details>

### Q93. How would you design a variable that creates several similar resources? 🟡

<details>
<summary><b>Answer</b></summary>

Use `map(object(...))` keyed by a stable name and feed it to `for_each`.
```hcl
variable "buckets" {
  type = map(object({
    versioning = optional(bool, true)
    expire_days = optional(number)
  }))
}
resource "aws_s3_bucket" "this" { for_each = var.buckets  bucket = each.key }
```
Adding or removing a key touches only that resource.

</details>

### Q94. How do you document variables and outputs for a module? 🟢

<details>
<summary><b>Answer</b></summary>

Always add `description` to each, and generate the README table with `terraform-docs markdown table .`, ideally from a pre-commit hook so docs never go stale. Good descriptions state units, allowed values and defaults' meaning.

</details>

### Q95. How do you pass several var files and what is the result of overlapping keys? 🟡

<details>
<summary><b>Answer</b></summary>

Repeat `-var-file`. They are processed in order and later values override earlier ones, as do `-var` flags mixed in at the position they appear.
```bash
terraform plan -var-file=common.tfvars -var-file=prod.tfvars -var="replicas=6"
```

</details>

### Q96. Can you use variables in the backend block? 🟡

<details>
<summary><b>Answer</b></summary>

No. Backend configuration is read before variables are evaluated, so `var.` references fail. Use partial configuration with `terraform init -backend-config=...` and a per-environment file instead. OpenTofu 1.8 added early evaluation, allowing variables and locals in backend and module source arguments.

</details>

### Q97. What does `null` mean in Terraform? 🟢

<details>
<summary><b>Answer</b></summary>

`null` means "no value". Setting an argument to `null` is the same as omitting it, so the provider default applies. It is useful in conditionals: `kms_key_id = var.encrypt ? aws_kms_key.k.arn : null`.

</details>

### Q98. Why would you use `set` rather than `list` for a variable? 🟡

<details>
<summary><b>Answer</b></summary>

When order and duplicates carry no meaning, such as security group IDs or subnet CIDRs, a set avoids spurious diffs from reordering and de-duplicates automatically. Use a list when order matters, for instance route priority or `element()` indexing.

</details>

### Q99. How do `try()` and `can()` help with validation and optional values? 🟡

<details>
<summary><b>Answer</b></summary>

`try(expr, fallback)` returns the first expression that evaluates without error; `can(expr)` returns true or false. Combine with regex for validation.
```hcl
condition = can(regex("^[a-z0-9-]{3,20}$", var.name))
value     = try(var.config.timeout, 30)
```

</details>

### Q100. Why does a conditional with different object types produce "Inconsistent conditional result types"? 🔴

<details>
<summary><b>Answer</b></summary>

Both branches of `cond ? a : b` must convert to a common type. Two objects with different attributes do not unify, so Terraform errors. Solve by returning the same shape in both branches (use `null` or empty values), wrapping the result in `tomap`/`jsonencode`, or by moving the choice to a `for_each` or `count` toggle on the resource.

</details>

## Loops, Conditionals, Dynamic Blocks and Functions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. How does `count` work? 🟢

<details>
<summary><b>Answer</b></summary>

`count = N` creates N instances of a resource or module, addressed `aws_instance.web[0]` to `[N-1]`. Inside the block, `count.index` gives the position.
```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = var.ami
  instance_type = "t3.micro"
  tags          = { Name = "web-${count.index}" }
}
```

</details>

### Q102. How does `for_each` work? 🟢

<details>
<summary><b>Answer</b></summary>

`for_each` takes a map or a set of strings and creates one instance per element, addressed by key. Inside, `each.key` and `each.value` are available.
```hcl
resource "aws_iam_user" "u" {
  for_each = toset(["asha", "ravi", "meera"])
  name     = each.key
}
```

</details>

### Q103. When should you choose `for_each` over `count`? 🟡

<details>
<summary><b>Answer</b></summary>

Whenever the instances are distinct things identified by a name. With `count`, removing an item from the middle of a list shifts every later index, so Terraform wants to destroy and recreate the shifted resources. With `for_each`, keys are stable, so only the removed key is destroyed. We keep `count` for simple on/off toggles and truly identical replicas.

</details>

### Q104. How do you use `for_each` with a list of objects? 🟡

<details>
<summary><b>Answer</b></summary>

Convert the list to a map with a `for` expression using a stable unique key.
```hcl
locals {
  rules = { for r in var.rule_list : r.name => r }
}
resource "aws_security_group_rule" "r" {
  for_each  = local.rules
  from_port = each.value.port
  # ...
}
```

</details>

### Q105. What does "The for_each value depends on resource attributes that cannot be determined until apply" mean? 🔴

<details>
<summary><b>Answer</b></summary>

Terraform must know the keys of `for_each` (or the value of `count`) at plan time to build the graph. If keys come from something computed, such as `aws_instance.x.id`, they are unknown. Fix it by keying on values known statically (names from variables), or apply in two steps with `-target` only as a last resort.

</details>

### Q106. How do you create a resource conditionally? 🟢

<details>
<summary><b>Answer</b></summary>

Use `count` with a boolean expression (or `for_each` with an empty map).
```hcl
resource "aws_eip" "nat" {
  count = var.enable_nat ? 1 : 0
}
# reference: one(aws_eip.nat[*].id)
```

</details>

### Q107. What is the conditional expression syntax? 🟢

<details>
<summary><b>Answer</b></summary>

`condition ? true_value : false_value`. Both results must be of compatible types. It is used for toggling arguments (`var.env == "prod" ? "m5.large" : "t3.small"`) and for count toggles.

</details>

### Q108. What are `for` expressions and how do you filter? 🟡

<details>
<summary><b>Answer</b></summary>

They build lists or maps from collections. Brackets produce a list, braces a map, and `if` filters.
```hcl
[for s in var.subnets : s.id if s.public]
{ for k, v in var.tags : lower(k) => v }
```

</details>

### Q109. What is the splat expression and what does `one()` add? 🔴

<details>
<summary><b>Answer</b></summary>

`aws_instance.web[*].id` collects an attribute from every instance into a list. It also turns a single object or null into a list of zero or one elements, which is why `one(aws_eip.nat[*].id)` returns the value or `null` for resources with a count toggle. `one()` errors if the list has more than one element.

</details>

### Q110. What is a `dynamic` block? 🟡

<details>
<summary><b>Answer</b></summary>

It generates repeated nested blocks from a collection.
```hcl
resource "aws_security_group" "web" {
  name = "web"
  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    }
  }
}
```
Only nested blocks can be dynamic, not top-level resources or plain attributes.

</details>

### Q111. What does `iterator` do in a dynamic block and when should you avoid dynamic blocks? 🟡

<details>
<summary><b>Answer</b></summary>

By default the iterator is named after the block label (`ingress.value`). `iterator = rule` renames it, which helps in nested dynamics. Avoid dynamic blocks for two or three static items: they hurt readability and make plans harder to follow. Use them for variable-length input.

</details>

### Q112. How do you loop over nested structures, for example subnets per VPC? 🔴

<details>
<summary><b>Answer</b></summary>

Flatten into a single list of objects with unique keys, then `for_each` over it.
```hcl
locals {
  subnets = flatten([
    for vpc_k, vpc in var.vpcs : [
      for s in vpc.subnets : { key = "${vpc_k}-${s.name}", vpc = vpc_k, cidr = s.cidr }
    ]
  ])
}
resource "aws_subnet" "s" {
  for_each   = { for s in local.subnets : s.key => s }
  vpc_id     = aws_vpc.v[each.value.vpc].id
  cidr_block = each.value.cidr
}
```

</details>

### Q113. What do `merge`, `concat` and `lookup` do? 🟡

<details>
<summary><b>Answer</b></summary>

`merge(map1, map2)` combines maps with later keys winning, great for tag maps. `concat(list1, list2)` joins lists. `lookup(map, key, default)` reads a key with a fallback; for newer code, `map[key]` or `try(map[key], default)` is often clearer.

</details>

### Q114. How do `coalesce`, `coalescelist` and `try` differ? 🟡

<details>
<summary><b>Answer</b></summary>

`coalesce(a, b, c)` returns the first non-null, non-empty-string argument. `coalescelist` returns the first non-empty list. `try` returns the first expression that does not raise an error, so it can handle missing attributes. `coalesce` does not rescue errors, and `try` does not skip nulls.

</details>

### Q115. Which string and collection functions come up most often? 🟢

<details>
<summary><b>Answer</b></summary>

`length`, `join`, `split`, `upper`, `lower`, `format`, `replace`, `substr`, `contains`, `element`, `keys`, `values`, `sort`, `distinct`, `slice`. In an interview, know how to use `format("%s-%03d", var.name, count.index)` and `join(",", var.list)`.

</details>

### Q116. How do you carve subnets with `cidrsubnet`? 🟡

<details>
<summary><b>Answer</b></summary>

`cidrsubnet(prefix, newbits, netnum)` adds newbits to the prefix length and picks the netnum-th block.
```hcl
cidrsubnet("10.0.0.0/16", 8, 2)   # "10.0.2.0/24"
cidrhost("10.0.2.0/24", 10)       # "10.0.2.10"
cidrsubnets("10.0.0.0/16", 4, 4, 8)  # several blocks of different size
```

</details>

### Q117. What is the difference between `file` and `templatefile`? 🟡

<details>
<summary><b>Answer</b></summary>

`file(path)` reads content verbatim. `templatefile(path, vars)` renders a template with `${var}` interpolation and directives like `%{ for x in list }...%{ endfor }`. Use it for user data and config files.
```hcl
user_data = templatefile("${path.module}/init.sh.tpl", { app_port = var.port })
```

</details>

### Q118. Why use `jsonencode` for IAM policies? 🟡

<details>
<summary><b>Answer</b></summary>

It builds valid JSON from HCL objects, so there are no quoting mistakes and you can use expressions inside. It also normalises key order so the policy JSON compares cleanly. Heredoc JSON is easy to break and gives noisy diffs.
```hcl
policy = jsonencode({
  Version = "2012-10-17"
  Statement = [{ Effect = "Allow", Action = ["s3:GetObject"], Resource = "${aws_s3_bucket.b.arn}/*" }]
})
```
For attached policies, `aws_iam_policy_document` is another good option.

</details>

### Q119. What do `zipmap`, `keys`, `values` and `toset` do? 🟡

<details>
<summary><b>Answer</b></summary>

`zipmap(keys_list, values_list)` builds a map from two lists. `keys(map)` and `values(map)` return sorted keys and the corresponding values. `toset(list)` removes duplicates and converts to a set, which `for_each` needs.

</details>

### Q120. Which newer string functions replace clumsy regex checks? 🟡

<details>
<summary><b>Answer</b></summary>

`startswith(str, prefix)` and `endswith` (1.3), `strcontains(str, substr)` (1.5), plus `trimprefix`, `trimsuffix` and `format`. They make validations readable and avoid regex mistakes.
```hcl
condition = startswith(var.bucket_name, "acme-") && !strcontains(var.bucket_name, "_")
```

</details>

### Q121. How do you group items by a key using `for` expressions? 🔴

<details>
<summary><b>Answer</b></summary>

Use the `...` grouping suffix, which collects values with the same key into a list.
```hcl
locals {
  by_az = { for s in var.servers : s.az => s.name... }
}
# {"a" = ["web1","web3"], "b" = ["web2"]}
```
Without `...` duplicate keys raise an error.

</details>

### Q122. Why can `timestamp()` cause trouble? 🟡

<details>
<summary><b>Answer</b></summary>

It returns the current time on every evaluation, so putting it into a resource attribute creates a diff on every plan. For tags we avoid it or use `lifecycle { ignore_changes = [tags["created"]] }`. Terraform 1.5 added `plantimestamp()`, which returns the time of the plan and is meant for checks and validations rather than resource arguments. `uuid()` has the same problem.

</details>

### Q123. How do you trigger a resource when a file changes? 🟡

<details>
<summary><b>Answer</b></summary>

Hash the file and use that in a trigger or attribute.
```hcl
resource "aws_s3_object" "cfg" {
  bucket = aws_s3_bucket.b.id
  key    = "app.conf"
  source = "app.conf"
  etag   = filemd5("app.conf")
}
```
`filebase64sha256` is typical for Lambda `source_code_hash`.

</details>

### Q124. What are `regex`, `regexall` and `replace` used for? 🟡

<details>
<summary><b>Answer</b></summary>

`regex(pattern, str)` returns the first match or errors if none, so wrap with `try` or `can`. `regexall` returns a list of all matches (empty if none). `replace(str, search, repl)` substitutes, treating a search wrapped in `/.../` as regex. Terraform uses RE2 syntax.

</details>

### Q125. What do `count.index`, `each.key` and `each.value` mean? 🟢

<details>
<summary><b>Answer</b></summary>

`count.index` is the zero-based position when `count` is used. With `for_each` over a map, `each.key` is the map key and `each.value` the element; over a set, key and value are both the string. They are only valid inside the resource or module that uses the meta-argument.

</details>

### Q126. How do you switch a resource from `count` to `for_each` without recreating it? 🔴

<details>
<summary><b>Answer</b></summary>

Change the config, then tell Terraform each old address now maps to a new one using `moved` blocks (1.1+), so state is rewritten during plan with no API calls.
```hcl
moved {
  from = aws_subnet.private[0]
  to   = aws_subnet.private["a"]
}
```
Verify with `terraform plan`: it should show "has moved" and zero destroys. Without moved blocks you would run `terraform state mv` for every instance.

</details>

### Q127. How do you safely read a map key that may not exist? 🟡

<details>
<summary><b>Answer</b></summary>

`lookup(map, "key", default)` or `try(map["key"], default)`. Direct indexing with a missing key fails: `Invalid index: the given key does not identify an element in this collection value`. For nested optional attributes `try(var.cfg.a.b, null)` is common.

</details>

### Q128. What do `sort`, `reverse`, `slice`, `range`, `min` and `max` do? 🟡

<details>
<summary><b>Answer</b></summary>

`sort` orders strings, `reverse` reverses a list, `slice(list, start, end)` takes a sub-list, `range(n)` generates 0..n-1 (handy to build counts of names), `min` and `max` take numbers as separate arguments, so pass a list with the expansion symbol: `max(var.sizes...)`. There is no built-in `sum`; use a `for` expression with a local or `length`-based arithmetic where you can.

</details>

### Q129. What does `setproduct` give you? 🟡

<details>
<summary><b>Answer</b></summary>

It returns every combination of elements from the input sets or lists, useful for cross-products such as environments by regions or subnets by route tables.
```hcl
for_each = { for p in setproduct(var.envs, var.regions) : "${p[0]}-${p[1]}" => p }
```

</details>

### Q130. How do you reference the outputs of a module that uses `for_each`? 🔴

<details>
<summary><b>Answer</b></summary>

The module becomes a map of objects keyed by the `for_each` key, so you read `module.app["blue"].lb_dns` or collect with `{ for k, m in module.app : k => m.lb_dns }`. Splat does not work on maps; use a `for` expression or `values(module.app)[*].lb_dns`.

</details>

### Q131. Why does "Sensitive values, or values derived from sensitive values, cannot be used as for_each arguments" happen? 🔴

<details>
<summary><b>Answer</b></summary>

`for_each` keys appear in resource addresses, plans and state paths, which are not redacted, so Terraform refuses to leak a sensitive value through them. Restructure so the keys are non-sensitive (key by name and keep the secret in `each.value`), or use `nonsensitive()` on a value you know is not secret.

</details>

### Q132. Is `"${var.name}"` different from `var.name`? 🟢

<details>
<summary><b>Answer</b></summary>

In Terraform 0.12 and later, a lone `"${var.name}"` is redundant; write `var.name` directly. Interpolation is only needed when combining with other text, like `"${var.app}-${var.env}"`. Linters such as tflint flag the old style.

</details>

### Q133. What are `path.module`, `path.root` and `path.cwd`? 🟢

<details>
<summary><b>Answer</b></summary>

`path.module` is the directory of the module containing the expression, `path.root` is the root module's directory, `path.cwd` is where Terraform was launched. Use `path.module` when loading files that live next to a module so it works when called from anywhere.

</details>

### Q134. How do you upload a directory of files or enumerate files? 🟡

<details>
<summary><b>Answer</b></summary>

`fileset(path, pattern)` returns a set of relative file paths, which you can feed straight to `for_each`.
```hcl
resource "aws_s3_object" "site" {
  for_each = fileset("${path.module}/site", "**")
  bucket   = aws_s3_bucket.web.id
  key      = each.value
  source   = "${path.module}/site/${each.value}"
  etag     = filemd5("${path.module}/site/${each.value}")
}
```

</details>

### Q135. How do hashing and encoding functions behave with respect to state? 🔴

<details>
<summary><b>Answer</b></summary>

`sha256`, `md5`, `base64encode`, `bcrypt` are deterministic except `bcrypt`, which salts and so changes every plan. Their results, like all attribute values, are written to state; hashing a secret does not make the stored input safe. Use hashes for change detection, not secrecy.

</details>

## Modules

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q136. What is a Terraform module and why use one? 🟢

<details>
<summary><b>Answer</b></summary>

A module is a directory of `.tf` files that groups resources into a reusable unit with inputs (variables) and outputs. Every configuration is a module; reusable ones are called from others. We write modules to enforce standards (every bucket encrypted, tagged and logged) and to stop copy-paste across teams.

</details>

### Q137. How do you call a module? 🟢

<details>
<summary><b>Answer</b></summary>

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name    = "prod"
  cidr    = "10.0.0.0/16"
}
```
Run `terraform init` (or `terraform get`) to download it, then use `module.vpc.vpc_id` elsewhere.

</details>

### Q138. How do inputs and outputs work with modules? 🟢

<details>
<summary><b>Answer</b></summary>

Variables defined in the child are its inputs, passed as arguments in the `module` block. The child exposes values through `output` blocks; the parent reads them as `module.<name>.<output>`. A module cannot see the caller's variables or resources unless they are passed in.

</details>

### Q139. Which module source types exist and where does `version` apply? 🟡

<details>
<summary><b>Answer</b></summary>

Local paths (`./modules/vpc`), the Terraform Registry, GitHub and generic Git URLs, Mercurial, HTTP URLs, S3 and GCS buckets. The `version` argument only works for registry sources. For Git, pin with `?ref=v1.4.2`.
```hcl
source = "git::https://github.com/acme/tf-modules.git//vpc?ref=v1.4.2"
```
The double slash selects a subdirectory.

</details>

### Q140. How should you version and pin modules? 🟡

<details>
<summary><b>Answer</b></summary>

Publish semver tags, and have callers pin an exact version or `~>` range for registry modules or a tag or commit SHA for Git modules. Never track `main`. Pinning means upgrades happen as a reviewed change with a plan, and rollbacks are a one-line revert.

</details>

### Q141. Can you use `count` and `for_each` on modules? 🟡

<details>
<summary><b>Answer</b></summary>

Yes, since Terraform 0.13 for modules, giving `module.app[0]` or `module.app["blue"]`. It lets you stamp out an environment per key. Watch out: removing a key destroys everything inside that module instance, and modules that define their own provider blocks cannot be used with `count`/`for_each`.

</details>

### Q142. How do providers get passed to a module? 🔴

<details>
<summary><b>Answer</b></summary>

Implicitly, the module inherits the default provider configurations. For aliases, use the `providers` map.
```hcl
module "dr" {
  source    = "./modules/app"
  providers = { aws = aws.dr }
}
```
Inside a reusable module declare `configuration_aliases` in `required_providers` if it needs more than one, and never put `provider` blocks with credentials in child modules.

</details>

### Q143. Why is `depends_on` on a module a poor default? 🔴

<details>
<summary><b>Answer</b></summary>

It makes every resource in the module depend on the target, so Terraform defers reading data sources inside and plans show wide "known after apply" changes. Prefer passing an output of the dependency as a real input so the graph carries the dependency precisely.

</details>

### Q144. When should you not write a module? 🟡

<details>
<summary><b>Answer</b></summary>

When it would be a thin wrapper around a single resource that exposes every argument of that resource; it adds a layer without adding policy or abstraction. A good module encodes opinions, hides repetition and has a small interface. Use modules for patterns (VPC with subnets and routes, service with ALB and ASG), not for one-to-one renaming.

</details>

### Q145. What is the standard module layout? 🟢

<details>
<summary><b>Answer</b></summary>

`main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`, a `README.md`, an `examples/` folder showing usage and optionally `modules/` for submodules and `tests/`. The registry publishing convention is the same, and tools like terraform-docs and tflint assume it.

</details>

### Q146. Should a module output whole resources or individual attributes? 🟡

<details>
<summary><b>Answer</b></summary>

Individual attributes are a stable contract: consumers depend only on what you chose to expose. Outputting an entire resource object leaks all its attributes (including sensitive ones) and makes internal refactors breaking changes. Use whole objects only for deliberate "bundle" outputs such as `{ id, arn, name }`.

</details>

### Q147. What does `terraform init` do for modules and when must you re-run it? 🔴

<details>
<summary><b>Answer</b></summary>

It downloads remote modules into `.terraform/modules` and records them in `modules.json`. Re-run `init` after adding a module, changing `source` or `version`, or editing local module paths in a way that changes the module tree. Local path modules are referenced in place rather than copied, so edits to them apply immediately.

</details>

### Q148. How do you move existing resources into a module without destroying them? 🔴

<details>
<summary><b>Answer</b></summary>

Wrap the resources in the module, then add `moved` blocks in the root mapping old addresses to new ones.
```hcl
moved {
  from = aws_s3_bucket.logs
  to   = module.logs.aws_s3_bucket.this
}
```
Plan must show moves and no replacements. For many resources or across state files, `terraform state mv` is the manual alternative.

</details>

### Q149. How do you publish a module to the public registry? 🟡

<details>
<summary><b>Answer</b></summary>

Host it in a public GitHub repository named `terraform-<PROVIDER>-<NAME>` (for example `terraform-aws-vpc`), with a root `main.tf`, `README.md`, and push semver tags like `v1.0.0`. Sign in to the registry with GitHub and publish the repo. Private registries in HCP Terraform use the same naming and tag conventions.

</details>

### Q150. Composition versus deeply nested modules: which is better? 🟡

<details>
<summary><b>Answer</b></summary>

Composition. A flat root that calls small modules and wires outputs to inputs is easy to read and test. Deep nesting (module calling module calling module) hides resources, makes passing providers painful, and amplifies breaking changes. We stay at one or two levels.

</details>

### Q151. How do you make a module work for several environments? 🟡

<details>
<summary><b>Answer</b></summary>

Keep environment differences in inputs, not in the module. The module exposes variables (sizes, counts, CIDRs, feature toggles) with safe defaults, and each environment's root passes its own values. Avoid `if var.env == "prod"` scattered through the module; expose explicit toggles like `multi_az` instead.

</details>

### Q152. What limitations do `for_each` modules have with providers? 🔴

<details>
<summary><b>Answer</b></summary>

A module call using `for_each` cannot also define the provider inside that module, and removing a key whose provider config lived inside the module leaves Terraform unable to destroy it ("Provider configuration not present"). Keep provider configuration in the root and pass it in, and destroy resources before removing their provider.

</details>

### Q153. What do you check before using a third-party registry module? 🟡

<details>
<summary><b>Answer</b></summary>

Source code (what IAM or network it opens by default), maintenance activity and open issues, pinned dependencies, how it handles encryption and logging, test coverage and licence. We pin the version, review the diff on upgrades and sometimes fork into our internal registry.

</details>

### Q154. How does a local module path work? 🟢

<details>
<summary><b>Answer</b></summary>

`source = "./modules/vpc"` is relative to the calling module's directory. Terraform treats it as part of the same repo, so no `version` is allowed. It is the simplest way to start and to share code inside a mono-repo.

</details>

### Q155. How do you handle a module upgrade that introduces breaking changes? 🔴

<details>
<summary><b>Answer</b></summary>

Read the release notes, bump the version on a branch and run `plan` in each environment, starting with dev. Look for replacements of stateful resources. If addresses changed use `moved` blocks (module authors should ship them) or `terraform state mv`. Apply to non-prod first, then promote the same version to prod.

</details>

## State Internals

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q156. What does the state file contain? 🟢

<details>
<summary><b>Answer</b></summary>

For every managed resource: its address, provider, the real resource ID, all attribute values (including computed ones and secrets) and dependencies. It also records outputs, the Terraform version and a serial and lineage for consistency. Treat it as sensitive data, not just metadata.

</details>

### Q157. What is the difference between local and remote state? 🟢

<details>
<summary><b>Answer</b></summary>

Local state is `terraform.tfstate` on disk: fine for experiments, bad for teams because it is not shared, locked or backed up. Remote state lives in a backend such as S3, GCS, Azure Blob or HCP Terraform, giving shared access, locking, versioning and encryption at rest.

</details>

### Q158. How do you configure the S3 backend? 🟢

<details>
<summary><b>Answer</b></summary>

```hcl
terraform {
  backend "s3" {
    bucket       = "acme-tfstate-prod"
    key          = "network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```
Then run `terraform init`. Enable bucket versioning, block public access and restrict IAM to the CI role.

</details>

### Q159. What is state locking and why does it matter? 🟡

<details>
<summary><b>Answer</b></summary>

When a command that can write state starts, Terraform takes a lock so two runs cannot modify the same state concurrently, which would corrupt it or create duplicate resources. A second run fails with `Error acquiring the state lock` and shows who holds it. Local state locks via a file, remote backends via their own mechanism.

</details>

### Q160. How does S3-native locking work and how does it replace DynamoDB? 🔴

<details>
<summary><b>Answer</b></summary>

Terraform 1.10 introduced `use_lockfile = true` (experimental; stable in 1.11), which writes a `.tflock` object next to the state using S3 conditional writes (`If-None-Match`), so no DynamoDB table is needed. The `dynamodb_table` argument is deprecated. When migrating, you can enable both temporarily so mixed-version teams stay locked, then remove DynamoDB.

</details>

### Q161. How do you inspect resources in state? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
terraform state list
terraform state show aws_instance.web
terraform state pull | jq '.resources | length'
```
`list` filters by address prefix, `show` prints attributes of one resource, and `pull` prints the raw JSON state.

</details>

### Q162. When do you use `terraform state mv`? 🟡

<details>
<summary><b>Answer</b></summary>

To rename a resource or move it between modules or state files without destroying it, for example after refactoring.
```bash
terraform state mv aws_instance.web module.app.aws_instance.web
terraform state mv -state-out=other.tfstate aws_s3_bucket.b aws_s3_bucket.b
```
It edits state only, so run `plan` afterwards to confirm no diff. Prefer `moved` blocks when you can, since they are reviewable and travel with code.

</details>

### Q163. When do you use `terraform state rm`? 🟡

<details>
<summary><b>Answer</b></summary>

To make Terraform forget a resource without destroying it, for example when handing ownership to another stack or removing a resource that was already deleted by hand. The real object remains and is no longer managed. In Terraform 1.7+ a `removed` block does the same declaratively.

</details>

### Q164. What are `state pull` and `state push` and why is push dangerous? 🔴

<details>
<summary><b>Answer</b></summary>

`pull` downloads the current state; `push` uploads a local file as the new state, bypassing checks beyond serial and lineage. A bad push can erase knowledge of real resources. Always back up first (`terraform state pull > backup.tfstate`), prefer S3 versioning for rollback and use `push` only for recovery like `errored.tfstate`.

</details>

### Q165. How did `terraform import` work before import blocks? 🟡

<details>
<summary><b>Answer</b></summary>

`terraform import aws_s3_bucket.logs acme-logs` attached an existing object to a resource address already written in config, updating state only. It did not write configuration, so you wrote the resource block first and iterated until the plan was clean. Import IDs differ per resource; check the docs' Import section.

</details>

### Q166. How do import blocks improve on the CLI command? 🟡

<details>
<summary><b>Answer</b></summary>

Since Terraform 1.5 you declare `import { to = ..., id = ... }` in code, so imports appear in `plan`, can be reviewed in PRs and run through CI. `terraform plan -generate-config-out=generated.tf` drafts the resource configuration for you.
```hcl
import {
  to = aws_s3_bucket.logs
  id = "acme-logs"
}
```
Remove the import block after apply (or leave it; it is idempotent).

</details>

### Q167. How do `moved` blocks compare to `terraform state mv`? 🔴

<details>
<summary><b>Answer</b></summary>

Both rewrite addresses in state without touching real infrastructure. `moved` is declarative, shows in the plan as "has moved to", works for everyone who pulls the code (each environment's state migrates on its next apply) and is safe to keep. `state mv` is imperative, one-off and must be repeated per state file.

</details>

### Q168. What is the `removed` block? 🟡

<details>
<summary><b>Answer</b></summary>

Introduced in Terraform 1.7, it removes a resource from state without destroying the real object.
```hcl
removed {
  from = aws_instance.legacy
  lifecycle { destroy = false }
}
```
Without `destroy = false` Terraform plans to destroy the resource. It is the reviewable replacement for `state rm`.

</details>

### Q169. What is the structure of the state file and what are `serial` and `lineage`? 🔴

<details>
<summary><b>Answer</b></summary>

It is JSON with `version` (format), `terraform_version`, `serial` (incremented on every write), `lineage` (a UUID fixed when state is created), `outputs` and `resources`. Backends reject writes whose serial is older than stored state, and refuse pushing state of another lineage, which prevents overwriting one stack's state with another's.

</details>

### Q170. Why is state a security concern, and what do you do about it? 🔴

<details>
<summary><b>Answer</b></summary>

Passwords, private keys, connection strings and tokens from resources like `aws_db_instance` or `tls_private_key` are stored in plaintext. We encrypt the bucket with KMS, block public access, limit read access to a few roles, keep access logs, avoid `terraform_remote_state` for broad sharing and use ephemeral or write-only values to keep secrets out of state where providers support it.

</details>

### Q171. What are Terraform CLI workspaces? 🟡

<details>
<summary><b>Answer</b></summary>

Multiple named state instances for the same configuration and backend. `terraform workspace new staging`, `select`, `list`, `show`, `delete`. The current name is `terraform.workspace`. In S3, non-default workspaces are stored under `env:/<name>/<key>`. They share code, backend and credentials, so they are not strong isolation.

</details>

### Q172. Workspaces versus separate directories per environment: which do you choose? 🔴

<details>
<summary><b>Answer</b></summary>

For real environments we use separate directories (or repos) with their own state, backend and credentials, since prod and dev need different accounts, access and often different shapes. CLI workspaces suit short-lived copies of identical stacks (feature or PR environments). Their downsides: it is easy to run against the wrong workspace, and code branches on `terraform.workspace`.

</details>

### Q173. When and how do you use `terraform force-unlock`? 🟡

<details>
<summary><b>Answer</b></summary>

When a crashed run or killed CI job leaves a stale lock. Confirm nobody is actually running, then `terraform force-unlock <LOCK_ID>` using the ID from the error. With S3 native locking you can also remove the `.tflock` object. Never force-unlock while a legitimate apply is running.

</details>

### Q174. What does "state snapshot was created by Terraform vX, which is newer than current vY" mean? 🔴

<details>
<summary><b>Answer</b></summary>

Someone ran a newer Terraform version against this state and it was written in a newer format; older versions refuse to read it to avoid corruption. Upgrade your CLI to at least that version. Prevention: pin `required_version` and the CLI version in CI and in local tooling (tfenv, mise, asdf).

</details>

### Q175. What does `terraform state replace-provider` do? 🟡

<details>
<summary><b>Answer</b></summary>

It rewrites the provider address recorded for resources in state, needed when a provider's source changes, such as `registry.terraform.io/-/aws` to `hashicorp/aws`, or when moving to OpenTofu or a private mirror namespace.
```bash
terraform state replace-provider registry.terraform.io/hashicorp/aws registry.example.com/acme/aws
```

</details>

### Q176. How do you migrate state from one backend to another safely? 🔴

<details>
<summary><b>Answer</b></summary>

1. Back up: `terraform state pull > backup.tfstate`.
2. Edit the `backend` block to the new one.
3. Run `terraform init -migrate-state` and confirm the copy.
4. Run `terraform plan` and expect no changes.
5. Keep the old state read-only for a while before deleting it.
Make sure nobody applies during the migration.

</details>

### Q177. Which backends does Terraform support? 🟢

<details>
<summary><b>Answer</b></summary>

`s3`, `gcs`, `azurerm`, `http`, `pg`, `kubernetes`, `consul`, `oss` and the `cloud` block for HCP Terraform; `local` is the default. Not all support locking equally. OpenTofu supports the same family plus client-side state encryption.

</details>

### Q178. How does refresh work and what is the effect of `-refresh=false`? 🔴

<details>
<summary><b>Answer</b></summary>

During plan or apply Terraform calls each provider's read to update state attributes from reality before diffing. `-refresh=false` skips this and trusts state, which speeds large plans but hides drift and can produce a plan that fails at apply because reality moved. We use it only for fast iterations in dev, never for the authoritative plan.

</details>

### Q179. Can you import resources into `for_each` addresses, or into modules? 🟡

<details>
<summary><b>Answer</b></summary>

Yes. The `to` address can include a key or module path, and from Terraform 1.7 `import` blocks accept `for_each`.
```hcl
import {
  for_each = { a = "subnet-0abc", b = "subnet-0def" }
  to       = aws_subnet.private[each.key]
  id       = each.value
}
```

</details>

### Q180. Why should you not edit the state file by hand? 🟢

<details>
<summary><b>Answer</b></summary>

A typo or wrong serial can corrupt state and orphan real infrastructure, and manual edits leave no audit trail. Use `state mv`, `state rm`, `import`, `moved`, `removed` or `-replace`, which validate changes. If you must edit, back up, change a pulled copy and push it with an incremented serial.

</details>

## Lifecycle, Dependencies, Provisioners and Drift

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. What are meta-arguments? 🟢

<details>
<summary><b>Answer</b></summary>

Arguments that every resource (or module) supports regardless of provider: `depends_on`, `count`, `for_each`, `provider` and `lifecycle`. Modules support the same except `lifecycle`. They are processed by Terraform Core rather than by the provider.

</details>

### Q182. What lifecycle options are available? 🟢

<details>
<summary><b>Answer</b></summary>

Inside `lifecycle { }`: `create_before_destroy`, `prevent_destroy`, `ignore_changes`, `replace_triggered_by` (1.2+), and `precondition` and `postcondition` blocks (1.2+). They change how Terraform plans replacements and which diffs it respects.

</details>

### Q183. How does `create_before_destroy` work and what can go wrong? 🔴

<details>
<summary><b>Answer</b></summary>

On replacement Terraform builds the new object first, then destroys the old, avoiding downtime for things like load balancer targets or certificates. Problems: names or unique values must not collide (use `name_prefix` or random suffixes), quota limits may be hit temporarily, and the flag propagates to dependencies, which can create unexpected cycles.

</details>

### Q184. What does `prevent_destroy` guarantee and what does it not? 🔴

<details>
<summary><b>Answer</b></summary>

It makes any plan that would destroy that resource fail with `Instance cannot be destroyed`. It is a guard-rail for databases and state buckets. It does not stop someone deleting the whole resource block (then Terraform has no lifecycle to read), running `state rm`, or deleting the object outside Terraform.

</details>

### Q185. When do you use `ignore_changes`? 🟡

<details>
<summary><b>Answer</b></summary>

When something else legitimately modifies an attribute, such as autoscaling changing `desired_capacity`, or an external system adding tags. List attributes explicitly.
```hcl
lifecycle {
  ignore_changes = [desired_capacity, tags["last_deployed"]]
}
```
`ignore_changes = all` freezes everything after creation and hides real drift, so we avoid it.

</details>

### Q186. What does `replace_triggered_by` do? 🔴

<details>
<summary><b>Answer</b></summary>

It forces replacement of a resource when another resource or attribute changes, even if the resource itself shows no diff.
```hcl
lifecycle {
  replace_triggered_by = [aws_launch_template.app.latest_version]
}
```
Useful when a dependency changes in a way the dependent does not notice (for instance a config object feeding a running instance).

</details>

### Q187. How do `precondition` and `postcondition` differ? 🟡

<details>
<summary><b>Answer</b></summary>

A `precondition` is checked before the resource is planned or applied and validates its inputs and assumptions ("the AMI must be x86_64"). A `postcondition` runs after and validates the result ("the subnet must have a public IP"). Both fail the run with your `error_message`. They can also sit on data sources and outputs.

</details>

### Q188. What is a `check` block and how does it differ from validation? 🔴

<details>
<summary><b>Answer</b></summary>

A `check` block (1.5+) holds assertions, optionally with a scoped data source, that run at the end of plan and apply. Failures produce warnings, not errors, so they do not block the run. We use them for health probes like "the endpoint returns 200" or certificate expiry, while `validation` and `precondition` block bad changes.
```hcl
check "site_up" {
  data "http" "home" { url = "https://${aws_lb.main.dns_name}" }
  assert {
    condition     = data.http.home.status_code == 200
    error_message = "Site is not returning 200"
  }
}
```

</details>

### Q189. What is the difference between implicit and explicit dependencies? 🟢

<details>
<summary><b>Answer</b></summary>

Implicit: Terraform infers order from references (`subnet_id = aws_subnet.a.id`). Explicit: you add `depends_on = [aws_iam_role_policy.x]` for dependencies the graph cannot see, such as an IAM policy that must exist before an instance starts. Prefer implicit references; use `depends_on` sparingly and comment why.

</details>

### Q190. What are provisioners and why does HashiCorp call them a last resort? 🟡

<details>
<summary><b>Answer</b></summary>

Provisioners (`local-exec`, `remote-exec`, `file`) run scripts when a resource is created or destroyed. They are imperative, not part of the plan, cannot be diffed, need network and credentials access to the host and leave tainted resources on failure. Better options: user data or cloud-init, golden images with Packer, configuration tools like Ansible, or provider resources.

</details>

### Q191. How do provisioner failures and destroy-time provisioners behave? 🔴

<details>
<summary><b>Answer</b></summary>

By default a failed provisioner fails the apply and marks the resource tainted, so it is recreated next time. `on_failure = continue` ignores the error. `when = destroy` runs before destruction and can reference only the resource's own attributes (via `self`); if it fails the destroy halts, so they are brittle.

</details>

### Q192. What does the `connection` block do? 🟡

<details>
<summary><b>Answer</b></summary>

It tells `remote-exec` and `file` provisioners how to reach the machine (SSH or WinRM): host, user, private key, bastion host and timeout. It requires network reachability from wherever Terraform runs, which is one reason we avoid these provisioners in CI.

</details>

### Q193. What is configuration drift and how does it happen? 🟢

<details>
<summary><b>Answer</b></summary>

Drift is when real infrastructure differs from what Terraform state and code say. Causes: manual console changes, other automation (autoscaling, operators), emergency fixes and provider-side changes. Drift is silent until the next plan, which is why detection should be scheduled.

</details>

### Q194. How do you detect drift? 🟡

<details>
<summary><b>Answer</b></summary>

Run `terraform plan -refresh-only` (or a normal `plan -detailed-exitcode`) on a schedule and alert on exit code 2. HCP Terraform offers health assessments that do this natively, and tools like driftctl (now archived) or Spacelift and env0 drift detection do the same. The goal is to find drift before you need an urgent change.

</details>

### Q195. How do you remediate drift? 🔴

<details>
<summary><b>Answer</b></summary>

Decide which side is right. If code is right, `apply` to revert reality. If the manual change should stay, update the code to match (or `apply -refresh-only` to accept it into state, then align code). If an external system legitimately owns an attribute, use `ignore_changes`. Then fix the process that allowed the manual edit, such as tighter IAM.

</details>

### Q196. Why do `create_before_destroy` and dependencies sometimes cause "Cycle" errors? 🔴

<details>
<summary><b>Answer</b></summary>

`create_before_destroy` reverses the destroy ordering for that resource and its dependencies, so a dependency edge may be satisfied in one direction for create and the opposite for destroy, forming a cycle. Run `terraform graph -type=plan` to see it, then break the loop by removing the unnecessary reference, splitting the resource (for example separate `aws_security_group_rule`) or setting the lifecycle flag consistently.

</details>

### Q197. What does `terraform test` do at a basic level? 🟢

<details>
<summary><b>Answer</b></summary>

Since 1.6 it runs `.tftest.hcl` files containing `run` blocks that apply (or plan) real configuration and assert on results, then destroys what it created.
```hcl
run "bucket_is_private" {
  command = plan
  assert {
    condition     = aws_s3_bucket_public_access_block.b.block_public_acls == true
    error_message = "Public ACLs must be blocked"
  }
}
```

</details>

### Q198. How does `depends_on` affect data sources? 🔴

<details>
<summary><b>Answer</b></summary>

A data source with `depends_on` cannot be read at plan time if the target has pending changes, so its read is deferred to apply and its attributes become unknown, causing wider downstream diffs. This is why we prefer referencing an attribute from the dependency in the data source's arguments rather than using `depends_on`.

</details>

### Q199. How would you break a dependency cycle between two resources? 🔴

<details>
<summary><b>Answer</b></summary>

Find the loop (`terraform graph | dot -Tsvg`, or the error's list of nodes). The usual cause is two security groups referencing each other inline: split the rules into separate `aws_security_group_rule` or `aws_vpc_security_group_ingress_rule` resources so both groups exist first. Another trick is introducing an intermediate resource or passing a static name instead of an attribute.

</details>

### Q200. What is HCP Terraform and how does it differ from open-source Terraform alone? 🟡

<details>
<summary><b>Answer</b></summary>

HCP Terraform (formerly Terraform Cloud) is HashiCorp's managed service for remote runs, state storage with locking and history, access control, policy enforcement with Sentinel or OPA, VCS-driven runs, a private registry and drift detection. The CLI is the same; the service adds collaboration and governance. Terraform Enterprise is the self-hosted version.

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
