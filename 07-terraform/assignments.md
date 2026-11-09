# Terraform & IaC — Assignments

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

> 25 graded exercises, from your first `terraform apply` to policy-as-code pipelines · 🟢 10 Beginner · 🟡 10 Intermediate · 🔴 5 Advanced

[⬅ Back to Terraform & IaC](./README.md) · [🏠 Home](../README.md)

## How to use these

Do them in order within a level. Work in a throwaway AWS account or sandbox, keep a Git repo with one folder per assignment, and run `terraform destroy` when you finish each one. Hints are collapsed on purpose: try for 15 minutes first. Assignments 1 to 3 can be done without an AWS account using the `random` and `local` providers.

## Table of contents

- [🟢 Beginner](#-beginner) — A1 to A10
- [🟡 Intermediate](#-intermediate) — A11 to A20
- [🔴 Advanced](#-advanced) — A21 to A25

---

## 🟢 Beginner

### A1. Your first apply with no cloud account 🟢

**Objective:** Learn the init, plan, apply and destroy cycle using local providers.

**Tasks**
1. Create a folder with a `versions.tf` that requires Terraform `>= 1.9` and the `hashicorp/random` and `hashicorp/local` providers.
2. Add a `random_pet` resource and a `local_file` that writes the pet name into `hello.txt`.
3. Run `terraform init`, `plan`, `apply`, then look at `terraform.tfstate`.
4. Change the file content in code, run `plan`, and read the symbol next to the resource.
5. Run `terraform destroy`.

**Expected outcome:** `hello.txt` appears after apply and disappears after destroy. You can explain what `+`, `~` and `-/+` mean in a plan.

<details>
<summary>Hints</summary>

- `local_file` needs `filename` and `content`. Reference the pet with `random_pet.name.id`.
- A content change on `local_file` shows `-/+` (replace) because the resource cannot update in place.
- Open the state file and find the resource address and its attributes.

</details>

**Bonus:** Add `keepers` to `random_pet` and show what makes it regenerate.

### A2. Variables, locals and outputs 🟢

**Objective:** Parameterise a configuration and expose results.

**Tasks**
1. Add input variables `project` (string), `environment` (string, validated to be `dev`, `staging` or `prod`) and `tags` (map of strings).
2. Create a `locals` block that builds a `name_prefix` of `<project>-<environment>` and merges default tags with `var.tags`.
3. Use the prefix in a `local_file` name. Add an output for the file path.
4. Supply values three ways: `terraform.tfvars`, `-var` on the command line and `TF_VAR_environment`. Note which one wins.
5. Pass an invalid environment and read the error.

**Expected outcome:** The validation message appears for bad input. You can state the variable precedence order from memory.

<details>
<summary>Hints</summary>

- `validation { condition = contains(["dev","staging","prod"], var.environment) error_message = "..." }`
- Precedence, lowest to highest: defaults, environment variables, `terraform.tfvars`, `*.auto.tfvars`, then `-var` and `-var-file`.
- `merge(local.default_tags, var.tags)` combines maps, with later maps winning on conflicts.

</details>

**Bonus:** Make one variable `sensitive = true` and show it is hidden in the plan but present in state.

### A3. Read the state like an engineer 🟢

**Objective:** Get comfortable with the state commands you will use during incidents.

**Tasks**
1. Using the A1 or A2 project, run `terraform state list` and `terraform state show <address>`.
2. Run `terraform show -json | jq '.values.root_module.resources[].address'`.
3. Rename a resource in code from `random_pet.name` to `random_pet.main` and run `plan`. Observe the destroy and create.
4. Reverse it, then rename using `terraform state mv random_pet.name random_pet.main` and plan again.
5. Add a `moved` block instead of the `state mv` and compare.

**Expected outcome:** A rename with `state mv` or `moved` produces `No changes`, while a bare rename plans a replacement.

<details>
<summary>Hints</summary>

- `moved { from = random_pet.name  to = random_pet.main }` goes in code and is reviewable, which is why we prefer it over `state mv`.
- `terraform state rm` makes Terraform forget a resource without deleting it. Do not run it unless you mean it.

</details>

**Bonus:** Back up state with `terraform state pull > backup.tfstate` and describe when you would use `state push`.

### A4. Provision an S3 bucket on AWS 🟢

**Objective:** Create your first real AWS resource safely.

**Tasks**
1. Configure the `aws` provider (`~> 6.0`) with a region and authenticate with an AWS CLI profile.
2. Create an S3 bucket with a unique name using `random_id` as a suffix.
3. Add separate resources for versioning, default encryption (SSE-S3) and the public access block.
4. Output the bucket ARN. Apply, then confirm with `aws s3api get-bucket-versioning`.
5. Destroy.

**Expected outcome:** A private, versioned, encrypted bucket exists after apply and is gone after destroy.

<details>
<summary>Hints</summary>

- Since AWS provider v4 these settings are separate resources: `aws_s3_bucket_versioning`, `aws_s3_bucket_server_side_encryption_configuration`, `aws_s3_bucket_public_access_block`.
- Use `force_destroy = true` in a lab so destroy works when versioned objects exist.
- `aws sts get-caller-identity` confirms which account you are about to change.

</details>

**Bonus:** Add a lifecycle rule that expires noncurrent versions after 30 days.

### A5. Data sources 🟢

**Objective:** Reference existing infrastructure without managing it.

**Tasks**
1. Use `aws_caller_identity` and `aws_region` data sources and output the account ID.
2. Use `aws_availability_zones` to list available AZs and output the first two.
3. Use `aws_ami` to find the latest Amazon Linux 2023 AMI and output its ID and name.
4. Use `aws_vpc` with `default = true` to read the default VPC and its subnets with `aws_subnets`.

**Expected outcome:** Outputs show real account, AZ and AMI values. You can explain the difference between a `resource` and a `data` block.

<details>
<summary>Hints</summary>

- `aws_ami` needs `owners = ["amazon"]` and a `name` filter like `al2023-ami-2023.*-x86_64`.
- `aws_subnets` takes `filter { name = "vpc-id" values = [data.aws_vpc.default.id] }`.
- Data sources are read during plan, so they must be able to resolve at that time.

</details>

**Bonus:** Explain what happens when a data source depends on a resource that does not exist yet.

### A6. EC2 instance with a security group 🟢

**Objective:** Deploy a small web server and connect resources by reference.

**Tasks**
1. In the default VPC, create a security group allowing HTTP (80) from anywhere and no SSH.
2. Launch a `t3.micro` Amazon Linux 2023 instance with `user_data` that installs and starts nginx.
3. Associate the security group and output the public IP.
4. `curl` the IP. Then change the `user_data` text and observe what the plan does to the instance.

**Expected outcome:** nginx responds on port 80. You can say why user data changes can recreate the instance.

<details>
<summary>Hints</summary>

- Use `vpc_security_group_ids`, not `security_groups`, for VPC instances.
- Newer providers replace an instance when `user_data` changes unless `user_data_replace_on_change` is false. Check the plan before applying.
- `dnf install -y nginx && systemctl enable --now nginx` for AL2023.

</details>

**Bonus:** Use SSM Session Manager (instance profile with `AmazonSSMManagedInstanceCore`) instead of opening any inbound port for admin access.

### A7. `count` and conditional resources 🟢

**Objective:** Create resources conditionally and in repetition.

**Tasks**
1. Add a boolean variable `create_bucket` and use `count = var.create_bucket ? 1 : 0` on a bucket.
2. Output the bucket name using a safe expression that works when the count is 0.
3. Create three buckets using `count = 3` with names derived from `count.index`.
4. Remove the middle bucket from a list-based `count` setup and observe which resources are replaced.

**Expected outcome:** You see that removing an item from the middle of a `count` list shifts addresses and plans unwanted replacements.

<details>
<summary>Hints</summary>

- `one(aws_s3_bucket.optional[*].id)` returns `null` when the list is empty.
- `aws_s3_bucket.b[1]` is addressed by index, so deleting index 1 renumbers the rest.

</details>

**Bonus:** Rewrite it with `for_each = toset(var.names)` and show the same removal now only destroys one bucket.

### A8. `for_each` with maps 🟢

**Objective:** Create stable, named instances of a resource.

**Tasks**
1. Define a variable `buckets` as a map of objects such as `{ logs = { versioning = true }, assets = { versioning = false } }`.
2. Use `for_each` to create buckets, using `each.key` for the name and `each.value.versioning` to toggle versioning.
3. Output a map of bucket name to ARN using a `for` expression.
4. Add a new key and confirm only one resource is added.

**Expected outcome:** Adding or removing a key never touches the others.

<details>
<summary>Hints</summary>

- Versioning resource: `for_each = { for k, v in var.buckets : k => v if v.versioning }`.
- Output: `{ for k, b in aws_s3_bucket.this : k => b.arn }`.
- `for_each` keys must be known at plan time.

</details>

**Bonus:** Add a `type = map(object({ versioning = optional(bool, false) }))` constraint so the field can be omitted.

### A9. Formatting, validation and pre-commit 🟢

**Objective:** Build the habit of automated hygiene.

**Tasks**
1. Break formatting on purpose, then run `terraform fmt -check -diff` and `terraform fmt -recursive`.
2. Introduce a reference to an undefined variable and run `terraform validate`.
3. Create a `.gitignore` for Terraform (state, `.terraform/`, plan files) and commit `.terraform.lock.hcl`.
4. Install `pre-commit` and add hooks for `terraform_fmt` and `terraform_validate` (from the `antonbabenko/pre-commit-terraform` repo).

**Expected outcome:** A commit with bad formatting is rejected locally.

<details>
<summary>Hints</summary>

- `validate` needs `terraform init -backend=false` first if you do not want to touch the backend.
- Never commit `*.tfstate`. Do commit the lock file.

</details>

**Bonus:** Explain why the lock file matters for reproducible builds and what `terraform init -upgrade` does to it.

### A10. A tiny VPC from scratch 🟢

**Objective:** Understand the networking resources that every later project builds on.

**Tasks**
1. Create a VPC with one public subnet, an internet gateway and a route table with `0.0.0.0/0` to the IGW.
2. Associate the route table with the subnet.
3. Launch an instance in the subnet with a security group allowing HTTP and verify with `curl`.
4. Draw the dependency graph with `terraform graph | dot -Tpng > graph.png`.

**Expected outcome:** The instance is reachable from the internet and the graph shows IGW, route and subnet dependencies.

<details>
<summary>Hints</summary>

- Subnets need `map_public_ip_on_launch = true` (or an EIP) for a public IP.
- Terraform orders resources by references, so you rarely need `depends_on` here.
- `terraform graph` needs Graphviz (`dot`) installed.

</details>

**Bonus:** Add a private subnet and a NAT gateway, and explain the cost implication.

---

## 🟡 Intermediate

### A11. Move state to S3 with locking 🟡

**Objective:** Migrate local state to a remote, locked, versioned backend.

**Tasks**
1. Create a state bucket (versioned, encrypted, public access blocked) in a separate bootstrap config.
2. Add a `backend "s3"` block with `use_lockfile = true` and no bucket value.
3. Run `terraform init -backend-config="bucket=<name>" -migrate-state` and confirm the prompt.
4. Verify the object in S3 and that local `terraform.tfstate` is now empty.
5. Start an `apply` and leave it at the prompt. In another terminal run `terraform plan` and capture the lock error.

**Expected outcome:** State lives in S3. The second command fails with `Error acquiring the state lock`, showing a `.tflock` object.

<details>
<summary>Hints</summary>

- Native S3 locking requires Terraform 1.11 or newer for the stable behaviour.
- On older versions use `dynamodb_table` with a `LockID` string hash key.
- `terraform force-unlock <ID>` is for dead locks only.

</details>

**Bonus:** Break the lock on purpose by killing a run with `kill -9`, then recover.

### A12. Write a reusable S3 module 🟡

**Objective:** Package a secure bucket as a module with a clean interface.

**Tasks**
1. Create `modules/secure-bucket` with `main.tf`, `variables.tf`, `outputs.tf` and `versions.tf`.
2. Inputs: `name`, `versioning` (bool), `kms_key_arn` (optional), `lifecycle_days` (number), `tags`.
3. Enforce encryption, a public access block and an HTTPS-only bucket policy inside the module.
4. Call the module twice from a root with different inputs. Validate inputs with `validation` blocks.
5. Add a `README` for the module generated with `terraform-docs`.

**Expected outcome:** Two buckets from one module, every input documented, and invalid input is rejected with a helpful message.

<details>
<summary>Hints</summary>

- Use `dynamic "rule"` or conditional resources for optional KMS.
- Do not put a `provider` block inside a module. Declare `required_providers` only.
- Outputs are the module's public API, so keep them minimal and described.

</details>

**Bonus:** Add a `terraform test` file for it (see A19).

### A13. Dev and prod: tfvars, workspaces or directories 🟡

**Objective:** Compare environment strategies and choose one with reasons.

**Tasks**
1. Implement the same small stack three ways: with `-var-file` per environment, with workspaces, and with `envs/dev` and `envs/prod` root modules.
2. For each, answer: where does state live, how do you know which environment you are targeting, and how do you vary the backend?
3. Deliberately run a destructive command against the wrong environment in the workspace variant to feel the risk.
4. Write a one-page decision note recommending an approach for a team of ten.

**Expected outcome:** A decision note that names isolation, blast radius, review clarity and per-environment permissions as criteria.

<details>
<summary>Hints</summary>

- Workspaces share one backend config and one code path. Directories allow different backends, providers and versions.
- Show the workspace in your prompt with `terraform workspace show`, but do not rely on humans noticing.
- A common answer: directories per environment and shared modules.

</details>

**Bonus:** Show how Terragrunt or a generator would remove the duplicated root files.

### A14. Import an existing resource 🟡

**Objective:** Bring hand-made infrastructure under Terraform management.

**Tasks**
1. Create an S3 bucket and an EC2 security group manually in the console or CLI.
2. Write `import` blocks for them and run `terraform plan -generate-config-out=generated.tf`.
3. Clean up the generated code: remove computed or default attributes and replace literals with variables.
4. Run `plan` until it says `No changes`. Then apply to record state.
5. Repeat for one resource with the legacy `terraform import` CLI and compare.

**Expected outcome:** Both resources are managed, and a plan shows no diff.

<details>
<summary>Hints</summary>

- `import { to = aws_s3_bucket.legacy  id = "my-bucket-name" }`
- Import IDs differ per resource. Check the "Import" section of each resource's docs.
- Generated config is a starting point, not production code.

</details>

**Bonus:** Import a whole VPC with its subnets using `for_each` in `import` blocks.

### A15. Refactor safely with `moved` blocks 🟡

**Objective:** Restructure code without destroying real infrastructure.

**Tasks**
1. Take a flat config with an `aws_instance.web` and a security group.
2. Move both into a new module `modules/web` and add `moved` blocks so the plan shows no destroys.
3. Convert a `count`-based resource to `for_each` using `moved` blocks per instance.
4. Rename a module call and prove the plan is empty.

**Expected outcome:** Every refactor plan reads `has moved` or `No changes`, never `destroy`.

<details>
<summary>Hints</summary>

- `moved { from = aws_instance.web  to = module.web.aws_instance.this }`
- `from = aws_instance.web[0]` and `to = aws_instance.web["a"]` for count to for_each.
- Keep `moved` blocks in code for a release or two so other states can catch up.

</details>

**Bonus:** Explain how `moved` differs from `terraform state mv` and why teams prefer it in review.

### A16. Dynamic blocks and `for` expressions 🟡

**Objective:** Generate repeated nested blocks from data.

**Tasks**
1. Define `ingress_rules` as a list of objects (`port`, `cidrs`, `description`).
2. Build a security group using a `dynamic "ingress"` block.
3. Create an output that groups rule descriptions by port using a `for` expression with `...` grouping.
4. Rewrite it with `aws_vpc_security_group_ingress_rule` and `for_each`, and explain which approach you prefer.

**Expected outcome:** Adding a rule to the variable adds exactly one rule to AWS.

<details>
<summary>Hints</summary>

- `dynamic "ingress" { for_each = var.ingress_rules  content { from_port = ingress.value.port ... } }`
- Separate rule resources avoid the conflict between inline rules and `aws_security_group_rule`, and give stable addresses.
- Group-by syntax: `{ for r in var.rules : r.port => r.description... }`.

</details>

**Bonus:** Use `flatten` to expand one rule per CIDR.

### A17. Lint and scan: fix what `tflint` and `checkov` find 🟡

**Objective:** Make quality gates part of your daily loop.

**Tasks**
1. Take your A6 or A10 code and run `tflint --init && tflint`.
2. Run `checkov -d .` and list the failed checks.
3. Fix at least five findings in code (encrypted volumes, IMDSv2, restricted egress, descriptions, no public IPs).
4. Add two documented skips using inline `#checkov:skip=ID:reason` with a real justification.
5. Add a `.tflint.hcl` with the AWS ruleset and the `recommended` terraform preset.

**Expected outcome:** Both tools exit clean, and each skip has a written reason.

<details>
<summary>Hints</summary>

- IMDSv2: `metadata_options { http_tokens = "required" }`.
- Root volume: `root_block_device { encrypted = true }`.
- `checkov --compact --quiet` shortens output. `checkov -c CKV_AWS_79` runs one check.

</details>

**Bonus:** Output SARIF and upload it to GitHub code scanning.

### A18. Plan on PR, comment the result 🟡

**Objective:** Build the review half of an IaC pipeline.

**Tasks**
1. Create a GitHub Actions workflow triggered on `pull_request` for changes under `envs/**` and `modules/**`.
2. Run `fmt -check`, `validate`, `tflint` and `terraform plan -no-color`.
3. Post the plan as a PR comment (update the existing comment instead of creating a new one each push).
4. Make the checks required with branch protection.

**Expected outcome:** Every push to a PR updates one comment with the latest plan, and a failing `fmt` blocks merge.

<details>
<summary>Hints</summary>

- Use `-detailed-exitcode` so you can distinguish no changes (0), changes (2) and errors (1).
- Set `terraform_wrapper: false` in `setup-terraform` if you pipe output yourself.
- To update a comment, find it by a hidden marker like `<!-- tf-plan -->` with `github.rest.issues.listComments`.

</details>

**Bonus:** Truncate large plans at about 60,000 characters and attach the full plan as an artifact.

### A19. Test your module with `terraform test` 🟡

**Objective:** Add automated tests to a module.

**Tasks**
1. In your secure-bucket module, create `tests/basic.tftest.hcl`.
2. Write a `run` block with `command = plan` that asserts encryption and the public access block are configured.
3. Write a test that expects failure for an invalid name using `expect_failures`.
4. Write one `command = apply` test that creates a real bucket and asserts on its outputs, then runs cleanup automatically.
5. Run `terraform test` locally and in CI.

**Expected outcome:** `terraform test` reports all runs passed.

<details>
<summary>Hints</summary>

- `run "name" { command = plan  assert { condition = ...  error_message = "..." } }`
- `variables { }` inside a run block overrides inputs for that run.
- `mock_provider "aws" {}` lets plan-level tests run without credentials.

</details>

**Bonus:** Use a `mock_provider` so unit tests run without AWS access and keep a single integration test.

### A20. Lifecycle rules in practice 🟡

**Objective:** Control replacement, protection and drift with lifecycle settings.

**Tasks**
1. Add `prevent_destroy = true` to a bucket and try `terraform destroy`. Record the error.
2. Add `ignore_changes = [desired_capacity]` to an ASG, change it in the console, and show plan stays clean.
3. Add `create_before_destroy = true` to a launch template and a security group that an instance uses, and explain the naming problem it introduces.
4. Use `replace_triggered_by` to recreate an instance when a config file resource changes.
5. Use `precondition` and `postcondition` to validate an AMI architecture.

**Expected outcome:** You can say when each setting helps and when it hides real problems.

<details>
<summary>Hints</summary>

- With `create_before_destroy`, fixed names collide. Use `name_prefix`.
- `prevent_destroy` blocks the whole plan if any resource would be destroyed, not only that resource.
- `ignore_changes = all` is almost always a smell.

</details>

**Bonus:** Show `terraform apply -replace=aws_instance.web` and explain how it replaced the deprecated `taint` command.

---

## 🔴 Advanced

### A21. OIDC and least-privilege pipeline roles 🔴

**Objective:** Remove long-lived cloud keys from CI and scope what pipelines can do.

**Tasks**
1. Create the GitHub OIDC provider and two IAM roles in Terraform: a read-only plan role and an apply role.
2. Restrict the trust policy `sub` claim: pull requests for plan, protected environments for apply.
3. Replace `AdministratorAccess` on the apply role with a policy limited to the services your stack uses. Find missing permissions from `AccessDenied` messages and CloudTrail.
4. Add a permissions boundary to the apply role so it cannot create IAM admins.
5. Prove with a test PR from a branch that the apply role cannot be assumed outside the environment.

**Expected outcome:** No `AWS_ACCESS_KEY_ID` stored in GitHub. Assuming the apply role from a PR job fails with `Not authorized to perform sts:AssumeRoleWithWebIdentity`.

<details>
<summary>Hints</summary>

- Subjects look like `repo:ORG/REPO:pull_request`, `repo:ORG/REPO:ref:refs/heads/main` and `repo:ORG/REPO:environment:prod`.
- Use `StringEquals` for exact subs and `StringLike` only when you need wildcards.
- Use IAM Access Analyzer policy generation from CloudTrail to draft the least-privilege policy.

</details>

**Bonus:** Add `aud` and `job_workflow_ref` conditions so only your reusable workflow can assume the role.

### A22. Split state and share data across stacks 🔴

**Objective:** Reduce blast radius by splitting one big state into layers.

**Tasks**
1. Take the lab project and split it into `network` (VPC) and `app` (ALB + ASG) stacks with separate state keys.
2. Publish network outputs through SSM Parameter Store from the network stack and read them in the app stack with data sources. Then do the same with `terraform_remote_state` and compare.
3. Migrate existing resources between states without recreating anything, using `terraform state mv -state-out` or `removed` and `import` blocks.
4. Document the apply order and enforce it in the pipeline with job dependencies.

**Expected outcome:** Both stacks plan clean with no resource recreation. The app stack can be applied without write access to the network state.

<details>
<summary>Hints</summary>

- Back up both states before touching anything: `terraform state pull > backup.tfstate`.
- `removed { from = aws_vpc.this  lifecycle { destroy = false } }` drops a resource from state without destroying it.
- `terraform_remote_state` exposes all outputs and requires read access to the whole state file, which is why SSM or a data lookup is often safer.

</details>

**Bonus:** Describe how you would handle a circular dependency between two stacks.

### A23. Scheduled drift detection and remediation policy 🔴

**Objective:** Find out-of-band changes automatically and decide how to respond.

**Tasks**
1. Create a scheduled GitHub Actions workflow (`cron`) that runs `terraform plan -detailed-exitcode -refresh-only` for each environment.
2. On exit code 2, open or update a GitHub issue with the plan summary and notify Slack through a webhook secret.
3. Create real drift: change a security group rule in the console and an ASG size with the CLI.
4. Write a policy for each case: revert via apply, codify the change, or import and then codify.
5. Add ignore rules for fields that legitimately change (for example `desired_capacity` under autoscaling).

**Expected outcome:** A scheduled run opens an issue that shows the drifted attributes within one cycle.

<details>
<summary>Hints</summary>

- Exit codes: `0` no changes, `1` error, `2` changes present.
- Use `terraform show -no-color` on a saved plan to build a short summary.
- Use read-only credentials for the drift job so it can never change anything.

</details>

**Bonus:** Add an AWS Config or CloudTrail rule that alerts on manual changes to tagged resources.

### A24. Versioned module registry and a safe upgrade path 🔴

**Objective:** Publish modules like a product and upgrade consumers without surprises.

**Tasks**
1. Move the VPC module to its own repository with semantic version tags (`v1.0.0`).
2. Consume it with `source = "git::https://github.com/ORG/terraform-aws-vpc.git?ref=v1.0.0"` from dev and prod.
3. Make a backwards-compatible change (`v1.1.0`) and a breaking one (`v2.0.0`, for example renaming an input with a `moved` block for resources).
4. Upgrade dev first, review the plan, then prod. Record what the plan showed.
5. Add module tests and a CI release workflow that runs them before tagging.
6. Add a `CHANGELOG.md` and describe the support policy.

**Expected outcome:** Dev and prod pin different versions during a rollout, and the v2 upgrade plan shows no destroys thanks to `moved` blocks.

<details>
<summary>Hints</summary>

- Use `terraform init -upgrade` after changing a `ref`.
- Constrain registry modules with `version = "~> 1.1"`. Git sources only support `?ref=`.
- Keep `moved` blocks for at least one major version.

</details>

**Bonus:** Publish to a private registry (Terraform Cloud, or a Git-based internal one) and compare the experience.

### A25. Policy as code and cost guardrails in the pipeline 🔴

**Objective:** Add automated approval rules on top of the plan.

**Tasks**
1. Export the plan as JSON: `terraform plan -out=tfplan && terraform show -json tfplan > plan.json`.
2. Write three OPA (Rego) policies and run them with `conftest test plan.json`: deny security groups open to `0.0.0.0/0` on port 22, require tags `Owner` and `Environment`, and deny instance types outside an allow-list.
3. Add Infracost to the PR workflow to comment the monthly cost change and fail if a PR adds more than a threshold.
4. Make the pipeline fail with a clear message for each policy and show both a failing and a passing PR.
5. Document how an exception is requested and approved.

**Expected outcome:** A PR that opens SSH to the world is blocked automatically, and each PR shows its cost delta.

<details>
<summary>Hints</summary>

- Look at `resource_changes[]` in the plan JSON, and use `change.after` for the planned values.
- `conftest test --policy policy/ plan.json` returns a non-zero exit on `deny`.
- Run policies against the plan, not just the source, so module output and computed values are covered.

</details>

**Bonus:** Write the same policy in Sentinel or Checkov custom checks and compare maintenance cost.

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
