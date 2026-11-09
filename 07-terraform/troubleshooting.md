# Terraform — Troubleshooting Drills

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

> Real Terraform failures with the exact error text: state locks, lost state, imports, cycles, provider conflicts, bad plans, AWS auth, modules, workspaces and CI · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Terraform](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [State Locking and Backend Errors](#state-locking-and-backend-errors) — Q1–Q25
- [Lost, Corrupted and Drifted State](#lost-corrupted-and-drifted-state) — Q26–Q50
- [Import, Already Exists and State Surgery](#import-already-exists-and-state-surgery) — Q51–Q75
- [Dependencies, Cycles and Providers](#dependencies-cycles-and-providers) — Q76–Q100
- [Plan Errors and Unexpected Replacements](#plan-errors-and-unexpected-replacements) — Q101–Q125
- [Authentication and Cloud API Errors](#authentication-and-cloud-api-errors) — Q126–Q150
- [Modules, Variables, Sensitive Values and Workspaces](#modules-variables-sensitive-values-and-workspaces) — Q151–Q175
- [Partial Applies, Version Mismatches and CI Failures](#partial-applies-version-mismatches-and-ci-failures) — Q176–Q200

```mermaid
flowchart TD
    A["Terraform command fails"] --> B{"Which phase?"}
    B -->|"init"| C["Backend, providers, modules, lock file"]
    B -->|"validate / plan"| D["Syntax, references, for_each, cycles"]
    B -->|"refresh / plan"| E["Auth, state lock, drift"]
    B -->|"apply"| F["Cloud API errors, timeouts, partial apply"]
    C --> G["Read the first Error block, not the last"]
    D --> G
    E --> G
    F --> G
    G --> H["Reproduce with TF_LOG=DEBUG if the message is unclear"]
```

In production we always read the **first** `Error:` block, note which command and phase produced it, and check state, versions and credentials before touching code.

---

## State Locking and Backend Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `terraform plan` fails with `Error acquiring the state lock` and `ConditionalCheckFailedException: The conditional request failed`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another plan/apply holds the lock right now; a previous run crashed (CI runner killed, laptop closed, Ctrl-C twice) and left the lock item in DynamoDB.

**Diagnose →** The error prints `Lock Info` with `ID`, `Who`, `Operation` and `Created`. Check whether that person or pipeline is still running.
```bash
aws dynamodb scan --table-name terraform-locks \
  --query 'Items[].{id:LockID.S,info:Info.S}'
```

**Fix →** If a run is active, wait. If the holder is confirmed dead, release it with the lock ID from the message:
```bash
terraform force-unlock 6f1c2a9e-3b1d-4c7a-9a0e-2d5f8f1b7c11
```

**Prevent →** Serialize pipelines per stack (CI concurrency groups), give CI jobs a graceful timeout so Terraform can release the lock, and never use `-lock=false` on shared state.

</details>

### Q2. With the S3 backend and `use_lockfile = true`, you get `Error acquiring the state lock` with `StatusCode: 412` and `PreconditionFailed`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Terraform 1.10+ S3-native locking writes a `<key>.tflock` object with a conditional `If-None-Match` PUT. A 412 means that lock object already exists: another run holds it or a dead run left it behind.

**Diagnose →**
```bash
aws s3api head-object --bucket acme-tfstate --key prod/network/terraform.tfstate.tflock
aws s3 cp s3://acme-tfstate/prod/network/terraform.tfstate.tflock -   # shows Who / Created
```

**Fix →** Confirm no run is active, then `terraform force-unlock <ID>`. Deleting the `.tflock` object by hand works too but is the last resort.

**Prevent →** Same as DynamoDB locking: one pipeline per state key at a time, graceful job cancellation. Make sure the bucket policy allows `s3:PutObject` and `s3:DeleteObject` on `*.tflock`, otherwise unlocks silently fail.

</details>

### Q3. `terraform force-unlock` returns `Failed to unlock state: lock ID "..." does not match existing lock`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You copied an old lock ID; the original lock was already released and a **new** run has taken a fresh lock; or you are pointed at a different workspace/key.

**Diagnose →** Re-run `terraform plan` to see the current `Lock Info`, and check `terraform workspace show` and the backend key.

**Fix →** Use the ID from the latest error message. If the new lock belongs to an active run, do not unlock it.

**Prevent →** Treat `force-unlock` as a manual, logged action. Paste the exact `Lock Info` into the incident/PR before unlocking.

</details>

### Q4. A teammate ran `terraform apply -lock=false` "because the lock was stuck". Now the plan shows resources you never touched being recreated. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two applies wrote state concurrently. The last writer won, so the state no longer records objects the other run created. Terraform now plans to create (duplicate) or destroy things based on an incomplete state.

**Diagnose →**
```bash
aws s3api list-object-versions --bucket acme-tfstate --prefix prod/app/terraform.tfstate \
  --query 'Versions[].{v:VersionId,t:LastModified,size:Size}'
terraform state list | sort > now.txt
```
Download the previous versions and diff `terraform state list` output between them.

**Fix →** Freeze applies. Either restore the most complete state version and `terraform import` what the other run created, or keep the current one and import the missing objects. Re-plan until it shows only intended changes.

**Prevent →** Remove the habit: block `-lock=false` in CI wrappers, and teach `force-unlock` with verification instead.

</details>

### Q5. `terraform init` fails with `Error: Backend configuration changed`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You edited the `backend` block (bucket, key, region, role) or passed different `-backend-config` values than the ones cached in `.terraform/terraform.tfstate`.

**Diagnose →**
```bash
cat .terraform/terraform.tfstate | jq '.backend.config | {bucket,key,region}'
```
Compare with the `backend` block and your `-backend-config` file.

**Fix →**
- You want to **move** state to the new location: `terraform init -migrate-state`.
- The state is already at the new location, or you only switched env config locally: `terraform init -reconfigure`.

**Prevent →** In CI, always run `terraform init -reconfigure -backend-config=env/prod.hcl` in a clean workspace so no stale `.terraform` is reused.

</details>

### Q6. `terraform init -migrate-state` asks `Do you want to copy existing state to the new backend?` and the new backend already has a state. What is the danger? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target key already contains state, often because someone already migrated, or two stacks were pointed at the same key by mistake.

**Diagnose →**
```bash
terraform state pull > local-copy.json
aws s3 cp s3://acme-tfstate/prod/app/terraform.tfstate remote.json
jq '.lineage, .serial, (.resources|length)' local-copy.json remote.json
```
Different `lineage` means they are unrelated states.

**Fix →** Answer `no` until you know which state is authoritative. If both are real, use different keys. If one is a stale copy, keep the newer/complete one.

**Prevent →** Use one unique key per stack (`<env>/<component>/terraform.tfstate`) and enable bucket versioning so an overwrite is recoverable.

</details>

### Q7. `Error: Backend initialization required, please run "terraform init"` with `Reason: Initial configuration of the requested backend "s3"`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fresh clone or new CI runner with no `.terraform/` directory; you added a `backend` block to a stack that used local state.

**Diagnose →** `ls -la .terraform/` — it is missing or has no `terraform.tfstate`.

**Fix →** Run `terraform init` (with `-backend-config` if you use partial config). If you are converting from local state, `terraform init -migrate-state` and confirm the copy.

**Prevent →** Make `init` the first step of every pipeline job, including apply jobs that consume a saved plan.

</details>

### Q8. `terraform init` fails with `Error: Variables not allowed` pointing at the `backend "s3"` block. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You wrote `bucket = var.state_bucket` or `key = "${var.env}/terraform.tfstate"`. The backend is initialized before variables are evaluated, so no variables, locals or functions are allowed there.

**Diagnose →** Read the line number in the error; it points into the `terraform { backend ... }` block.

**Fix →** Use partial configuration:
```hcl
terraform {
  backend "s3" {}
}
```
```bash
terraform init -backend-config=backends/prod.hcl
```

**Prevent →** Keep a `backends/<env>.hcl` per environment, or let Terragrunt generate the backend block.

</details>

### Q9. In CI, `terraform init` hangs forever after printing `key  The path to the state file inside the bucket  Enter a value:`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Partial backend config with a required setting missing (the `-backend-config` file path is wrong, or the file lacks `key`). Terraform falls back to an interactive prompt.

**Diagnose →** Check the job's working directory and that the backend file exists: `ls backends/ && cat backends/prod.hcl`.

**Fix →** Pass the correct file and disable prompts:
```bash
terraform init -input=false -backend-config=backends/prod.hcl
```

**Prevent →** Always use `-input=false` in automation and set `TF_IN_AUTOMATION=1`. A missing value then fails fast instead of hanging until the job timeout.

</details>

### Q10. `Error: Failed to get existing workspaces: S3 bucket "acme-tfstate" does not exist.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo in the bucket name; the bootstrap stack that creates the state bucket was never applied; you are using credentials for a different AWS account.

**Diagnose →**
```bash
aws sts get-caller-identity
aws s3api head-bucket --bucket acme-tfstate
```

**Fix →** Fix the name or switch to the correct account/profile. If it truly doesn't exist, apply the bootstrap stack (local state, then migrate it into the bucket).

**Prevent →** Keep backend config in versioned `backends/*.hcl` files and print `aws sts get-caller-identity` at the start of every pipeline.

</details>

### Q11. `Unable to list objects in S3 bucket "acme-tfstate" with prefix "env:/": operation error S3: ListObjectsV2, https response error StatusCode: 403`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role can read/write the state object but lacks `s3:ListBucket` on the bucket. Terraform lists `env:/` to discover workspaces during `init`.

**Diagnose →**
```bash
aws s3api list-objects-v2 --bucket acme-tfstate --prefix env:/ --max-items 1
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::111122223333:role/tf-ci \
  --action-names s3:ListBucket --resource-arns arn:aws:s3:::acme-tfstate
```

**Fix →** Grant `s3:ListBucket` on `arn:aws:s3:::acme-tfstate` (optionally conditioned on `s3:prefix`), plus `s3:GetObject`/`s3:PutObject` on the key, and `s3:DeleteObject` for `.tflock` if using native locking.

**Prevent →** Keep a tested least-privilege backend policy as a module and attach it to every Terraform role.

</details>

### Q12. Apply finishes all resource changes, then fails with `Failed to save state` and `AccessDenied` mentioning `kms:GenerateDataKey`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bucket uses SSE-KMS with a customer-managed key (`kms_key_id` in the backend or bucket default encryption). The role can read state (or the old object used a different key) but cannot encrypt new writes.

**Diagnose →** Look for `errored.tfstate` in the working directory. Check the key policy and role policy for `kms:GenerateDataKey`, `kms:Encrypt` and `kms:Decrypt`.
```bash
aws kms get-key-policy --key-id alias/tfstate --policy-name default
```

**Fix →** Grant the KMS permissions, then push the saved state immediately:
```bash
terraform state push errored.tfstate
```
Do not run `apply` again first; that would work from the old state and duplicate resources.

**Prevent →** Test backend write permissions in a sandbox stack whenever IAM or KMS policies change.

</details>

### Q13. `Error: Failed to persist state to backend` and Terraform says it wrote `errored.tfstate`. The CI job ended. What now? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Network drop, expired credentials or a permission issue at the moment of writing state. The cloud resources were changed, but the remote state was not updated.

**Diagnose →** Find the `errored.tfstate` file. In CI it is lost unless you upload the working directory on failure. Check what the run created from the apply log.

**Fix →**
- If you have the file: fix credentials, then `terraform state push errored.tfstate`.
- If the file is gone: the state is behind reality. Use the apply log to list created resource IDs and `terraform import` them (or use `import` blocks), then re-plan.

**Prevent →** In pipelines, upload `errored.tfstate` as an artifact on failure (`if: failure()`), and use credentials whose session lasts longer than the longest apply.

</details>

### Q14. `terraform state push` fails with `cannot import state with lineage "a1..." over unrelated state with lineage "b2..."`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file you push came from a different stack or a re-created state (new lineage). Terraform protects you from overwriting an unrelated state.

**Diagnose →**
```bash
terraform state pull | jq '.lineage, .serial'
jq '.lineage, .serial' errored.tfstate
```

**Fix →** Make sure you are in the right directory/workspace/backend key. Only if you are certain the file is the correct one, use `terraform state push -force file.tfstate`, after taking a `state pull` backup.

**Prevent →** Never copy state files between stacks. Use `terraform state mv -state-out` or `removed` + `import` for moving resources.

</details>

### Q15. `Error refreshing state: state data in S3 does not have the expected content. The checksum calculated for the state stored in S3 does not match the checksum stored in DynamoDB.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With the legacy DynamoDB lock table, Terraform also stores an MD5 digest (`<bucket>/<key>-md5` item). Someone uploaded or restored state directly in S3, or a previous write failed halfway.

**Diagnose →** Check S3 object versions and the `-md5` item in the table:
```bash
aws dynamodb get-item --table-name terraform-locks \
  --key '{"LockID":{"S":"acme-tfstate/prod/app/terraform.tfstate-md5"}}'
```

**Fix →** Wait a minute and retry (it can be a delay). If it persists and the S3 object is the version you want, update the `Digest` to the "Calculated checksum" shown in the error, or delete the `-md5` item.

**Prevent →** Restore state through `terraform state push`, not `aws s3 cp`. Moving to `use_lockfile = true` (Terraform 1.10+) removes the DynamoDB digest entirely.

</details>

### Q16. After upgrading to Terraform 1.11 you see `Warning: Deprecated Parameter` for `dynamodb_table`. On an older laptop `use_lockfile` gives `An argument named "use_lockfile" is not expected here`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DynamoDB locking is deprecated in favour of S3-native locking. `use_lockfile` was added in 1.10, so older binaries reject it.

**Diagnose →** `terraform version` on each machine and CI image; check `required_version`.

**Fix →** Standardize on 1.10+ everywhere, then migrate in two steps: set both `use_lockfile = true` and `dynamodb_table` (both locks are taken), roll out, then remove `dynamodb_table`.

**Prevent →** Pin `required_version = ">= 1.10, < 2.0"` and the CI image version, and use `tfenv`/`mise` with a `.terraform-version` file.

</details>

### Q17. `Error: Failed to get existing workspaces: ... StatusCode: 301 ... requested bucket from "us-east-1", actual location "ap-south-1"`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The backend `region` doesn't match the bucket's region. The backend region is independent of the provider region.

**Diagnose →** `aws s3api get-bucket-location --bucket acme-tfstate`

**Fix →** Set `region = "ap-south-1"` in the backend config and run `terraform init -reconfigure`.

**Prevent →** Keep the state bucket region in the shared `backends/*.hcl` files, not inferred from `AWS_REGION`.

</details>

### Q18. `terraform init` fails on the backend with `No valid credential sources found`, but `aws s3 ls` works in the same shell. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The backend uses its own credential settings: `profile` or `assume_role` in the backend block refers to a profile that does not exist on this machine; `AWS_PROFILE` differs; the SSO session for that profile expired.

**Diagnose →**
```bash
grep -A8 'backend "s3"' *.tf; cat backends/prod.hcl
aws sts get-caller-identity --profile <profile-from-backend>
TF_LOG=DEBUG terraform init 2>&1 | grep -i -E 'credential|profile|assume'
```

**Fix →** Align the backend `profile`/`assume_role` with what exists, or remove `profile` and rely on environment credentials. Run `aws sso login --profile ...` if the session expired.

**Prevent →** Keep the backend free of personal profile names. Prefer `assume_role { role_arn = ... }` with environment base credentials so laptops and CI behave the same.

</details>

### Q19. Two pipelines for different stacks keep blocking each other with `Error acquiring the state lock`, though they manage unrelated resources. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both stacks use the **same backend key** (copy-pasted `backend.tf`), so they share one state and one lock. That's worse than locking: each apply wants to destroy the other's resources.

**Diagnose →**
```bash
grep -rn 'key' --include='*.tf' --include='*.hcl' . | grep tfstate
terraform state list | head
```
If one stack's `state list` shows the other's resources, they share state.

**Fix →** Give each stack its own key. Split the shared state with `terraform state mv -state-out=` or `removed`/`import` blocks into the new key.

**Prevent →** Derive keys from directory path (Terragrunt `path_relative_to_include()`), or add a CI check that no two stacks use the same key.

</details>

### Q20. `Error: Error acquiring the state lock` with `AccessDeniedException: User: arn:aws:sts::...:assumed-role/tf-ci/... is not authorized to perform: dynamodb:PutItem`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role has S3 access but no DynamoDB permissions on the lock table, or the table is in another region/account.

**Diagnose →** `aws dynamodb describe-table --table-name terraform-locks --region <backend-region>`

**Fix →** Grant `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:DeleteItem` (and `DescribeTable`) on the table ARN. Or switch to `use_lockfile = true` and drop DynamoDB.

**Prevent →** Ship backend permissions as one reusable IAM policy so S3, KMS and locking are granted together.

</details>

### Q21. `terraform plan` takes 10 minutes and then fails with `Error acquiring the state lock` and `context deadline exceeded`, only from the corporate network. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A proxy or firewall blocks or slows the DynamoDB/S3 endpoint; `HTTPS_PROXY` is set but `NO_PROXY` routes VPC endpoints incorrectly; DNS resolves S3 to a private endpoint you can't reach.

**Diagnose →**
```bash
env | grep -i proxy
TF_LOG=DEBUG terraform plan 2>&1 | grep -E 'dynamodb|s3\.|timeout'
curl -sv https://dynamodb.ap-south-1.amazonaws.com 2>&1 | head
```

**Fix →** Configure the proxy correctly (and trust its CA via the OS store or `AWS_CA_BUNDLE`), or run Terraform from a runner inside the VPC.

**Prevent →** Run shared-state Terraform from controlled runners, not from laptops on varied networks. `-lock-timeout=5m` helps only with genuine contention, not network issues.

</details>

### Q22. Apply was cancelled in GitHub Actions. The next run fails with the state lock held by `Who: runner@fv-az123` and the plan shows half the resources changed. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cancelling a job sends SIGINT then kills the process after a short grace period. Terraform could not finish writing state or releasing the lock.

**Diagnose →** Check the cancelled run's log for the last `Creation complete` / `Modifications complete` lines and whether `errored.tfstate` was uploaded. Confirm no other runner is active.

**Fix →** `terraform force-unlock <ID>`, then `terraform plan`. Resources finished before the kill are usually in state (Terraform persists state periodically during apply). Import anything the log shows as created but the plan wants to create again.

**Prevent →** Don't cancel applies; use `concurrency: { group: tf-prod, cancel-in-progress: false }`. Set `timeout-minutes` generously and upload `errored.tfstate` on failure.

</details>

### Q23. HCP Terraform: `terraform plan` from a laptop says `Error: Required token could not be found` for `app.terraform.io`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `cloud {}` block (or `remote` backend) needs an API token; you haven't logged in, or the token in `~/.terraform.d/credentials.tfrc.json` expired or was revoked.

**Diagnose →** `cat ~/.terraform.d/credentials.tfrc.json | jq 'keys'`; check `TF_TOKEN_app_terraform_io` in CI.

**Fix →** `terraform login` on a laptop. In CI set `TF_TOKEN_app_terraform_io` to a team or user token from a secret.

**Prevent →** Use team tokens for automation with an expiry, rotated through your secret manager.

</details>

### Q24. HCP Terraform: the run sits in `Pending` or `Plan queued` for a long time and nothing happens. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another run in that workspace is waiting for confirmation (`Needs confirmation`), so later runs queue behind it; the workspace is locked manually; agent-pool workspaces have no healthy agent; the organization hit its concurrency limit.

**Diagnose →** Open the workspace's Runs list and the agent pool status. Check `Settings > Locking`.

**Fix →** Apply or discard the run that awaits confirmation, unlock the workspace, or restart/scale the agents.

**Prevent →** Auto-discard stale speculative runs, keep agent pools sized with health checks, and alert on runs pending longer than N minutes.

</details>

### Q25. `terraform init` warns `Warning: Missing backend configuration` and creates a local `terraform.tfstate` in CI. Later, the next pipeline wants to create everything again. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `-backend-config` was passed but there is no `backend` block in the code (so Terraform ignores it and uses local state), or the backend block lives in a file excluded from the job. Each CI run starts with an empty local state.

**Diagnose →**
```bash
grep -rn 'backend\s*"' *.tf
ls terraform.tfstate .terraform/terraform.tfstate
```

**Fix →** Add the `backend` block. If a CI run already created resources with local state that was thrown away, import them into the remote state or delete the orphans manually.

**Prevent →** Fail the pipeline if `terraform.tfstate` exists in the working directory after apply, and enforce a backend via policy or a wrapper script.

</details>

---

## Lost, Corrupted and Drifted State

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. Someone deleted the state object from the S3 bucket. `terraform plan` now wants to create 140 resources that already exist. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The object was deleted (by hand, by a lifecycle rule or by a cleanup script). With versioning on, S3 only added a delete marker.

**Diagnose →**
```bash
aws s3api list-object-versions --bucket acme-tfstate --prefix prod/app/terraform.tfstate \
  --query '{versions:Versions[].[VersionId,LastModified,Size],markers:DeleteMarkers[].[VersionId,LastModified]}'
```

**Fix →** Do not apply. Remove the delete marker (restores the latest version) or copy a known-good version back:
```bash
aws s3api delete-object --bucket acme-tfstate --key prod/app/terraform.tfstate --version-id <delete-marker-id>
terraform plan   # should show no or small changes
```

**Prevent →** Versioning + MFA delete or a bucket policy denying `s3:DeleteObject` on `*.tfstate` for humans, and a noncurrent-version retention lifecycle rule (for example 90 days).

</details>

### Q27. State is gone and the bucket had no versioning. How do you recover a production stack? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unversioned bucket, local state on a lost laptop, or a CI workspace that used local state.

**Diagnose →** Search for any copy first: `terraform.tfstate.backup` files, CI artifacts, old laptops, HCP state history. Inventory real resources by tag:
```bash
aws resourcegroupstaggingapi get-resources --tag-filters Key=stack,Values=prod-app \
  --query 'ResourceTagMappingList[].ResourceARN'
```

**Fix →** Rebuild state with `import` blocks (Terraform 1.5+), one per resource, in a fresh empty state:
```hcl
import {
  to = aws_vpc.main
  id = "vpc-0a1b2c3d4e5f67890"
}
```
Iterate `terraform plan` until it shows `0 to add, 0 to change, 0 to destroy`, fixing code where it differs.

**Prevent →** Versioned, KMS-encrypted remote state with restricted delete, plus consistent tagging (`stack`, `env`, `managed-by=terraform`) so an inventory is possible.

</details>

### Q28. `Error: Failed to load state` followed by a JSON error like `invalid character '}' looking for beginning of object key string`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone edited state by hand (trailing comma, bad merge) and uploaded it, or a truncated write from an interrupted upload.

**Diagnose →**
```bash
terraform state pull > broken.json 2>/dev/null || aws s3 cp s3://acme-tfstate/prod/app/terraform.tfstate broken.json
jq . broken.json > /dev/null   # shows the line of the syntax error
```

**Fix →** Restore the previous S3 version. If no good version exists, fix the JSON carefully, bump `serial` by one, and `terraform state push`.

**Prevent →** Never hand-edit state. Use `terraform state mv/rm`, `moved`, `removed` and `import` blocks; they validate and back up.

</details>

### Q29. The plan prints `Note: Objects have changed outside of Terraform` and lists a security group with an extra ingress rule. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Drift: someone added a rule in the console or CLI during an incident.

**Diagnose →**
```bash
terraform plan -refresh-only
aws ec2 describe-security-groups --group-ids sg-0abc123 --query 'SecurityGroups[0].IpPermissions'
```
CloudTrail (`AuthorizeSecurityGroupIngress`) tells you who and when.

**Fix →** Decide: if the rule is wanted, add it to code. If not, a normal `terraform apply` removes it. Never just apply blindly during an incident; ask first.

**Prevent →** Read-only console access in prod, break-glass roles with alerts, and nightly `terraform plan -detailed-exitcode` drift checks.

</details>

### Q30. Every plan wants to change `desired_capacity` on an Auto Scaling group back to 2, though autoscaling has raised it to 6. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Terraform manages an attribute that something else (scaling policies, Karpenter, a human) also manages. That's expected drift, not a Terraform bug.

**Diagnose →** `terraform plan` shows `~ desired_capacity = 6 -> 2`.

**Fix →**
```hcl
resource "aws_autoscaling_group" "web" {
  # ...
  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```
Or don't set `desired_capacity` at all.

**Prevent →** For every attribute, pick one owner. Same pattern for ECS `desired_count` with service autoscaling.

</details>

### Q31. A resource was deleted manually in the console. The plan wants to create it, and an apply fails because a dependent resource still references the old ID. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Refresh noticed the object is gone and planned a create, but dependents (route table association, target group attachment) were not updated in the same plan because their attributes didn't appear to change.

**Diagnose →** `terraform plan` shows `+ create` for the deleted resource, and dependents show no change or `known after apply` IDs.

**Fix →** Usually one apply resolves it; if dependents still point at the old ID, force them to be recreated:
```bash
terraform apply -replace='aws_route_table_association.private["a"]'
```

**Prevent →** Restrict delete permissions in prod; use `prevent_destroy` on critical resources so even Terraform won't remove them without a code change.

</details>

### Q32. `terraform plan -refresh-only` shows drift, and you apply it. The next normal plan still wants to revert the change. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `apply -refresh-only` updates **state** to match reality but doesn't change **code**. The code still says the old value, so the next plan reverts.

**Diagnose →** Compare the value in code with `terraform state show <addr>`.

**Fix →** Update the code to the new value (or add `ignore_changes`), then plan should be clean.

**Prevent →** Use refresh-only only to record accepted drift, and always follow up with a code change in the same PR.

</details>

### Q33. You inherited a stack whose state lives in a committed `terraform.tfstate` in Git. Three engineers have different versions locally. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No remote backend; each engineer applied from their own copy, so the copies forked.

**Diagnose →**
```bash
git log --oneline -- terraform.tfstate
for f in a.tfstate b.tfstate c.tfstate; do jq -r '"\(.serial) \(.lineage) \(.resources|length)"' $f; done
```
Also scan the state for secrets: it's in Git history.

**Fix →** Pick the most complete copy (highest serial, same lineage), import anything others created, migrate to S3 with `terraform init -migrate-state`, delete the file from the repo and add `*.tfstate*` to `.gitignore`. Rotate any secrets found in history.

**Prevent →** Remote backend with locking from day one; pre-commit hook blocking `*.tfstate`.

</details>

### Q34. A plan shows an RDS instance's `engine_version` going from `15.8` back to `15.4`. Nobody changed it. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `auto_minor_version_upgrade = true` let AWS upgrade the engine during a maintenance window; your code still pins `15.4`. Downgrading isn't possible, so the apply would fail.

**Diagnose →** `aws rds describe-db-instances --db-instance-identifier prod-db --query 'DBInstances[0].EngineVersion'`

**Fix →** Pin only the major version (`engine_version = "15"`) so minor upgrades don't show as drift, or update code to `15.8`. Alternatively `ignore_changes = [engine_version]`.

**Prevent →** Decide who owns upgrades: either AWS (major-only pin) or Terraform (auto minor off, explicit bumps via PRs).

</details>

### Q35. Plans always show `~ tags_all` changes on hundreds of resources after adding `default_tags` to the provider. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Normal on the first apply after adding `default_tags`. Persisting changes come from a resource also setting the same tag key in `tags` with a different value, or a tag value computed at apply time (for example `timestamp()`).

**Diagnose →** `terraform plan | grep -B3 -A6 tags_all` to see which keys flip.

**Fix →** Apply once. Remove duplicated keys from resource-level `tags`, and never use `timestamp()`/`uuid()` in tags.

**Prevent →** Put static org tags only in `default_tags`; keep resource `tags` for resource-specific keys.

</details>

### Q36. An IAM policy shows changes every plan even though the JSON is "the same". 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A heredoc JSON string differs from how AWS normalizes it (statement order, single-string vs list `Action`, whitespace), or principals were given as account IDs that AWS rewrites to ARNs.

**Diagnose →** In the plan diff, look for `Action = "s3:GetObject"` vs `Action = ["s3:GetObject"]` or reordered keys.

**Fix →** Generate policies with `data "aws_iam_policy_document"` or `jsonencode()` and match AWS's canonical form (full ARNs for principals, lists where AWS uses lists).

**Prevent →** Ban raw heredoc JSON policies in code review; use policy documents everywhere.

</details>

### Q37. After an incident, someone changed an EC2 instance type in the console. `terraform plan` now wants to stop and resize it back during business hours. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Drift on `instance_type`; the AWS provider changes it in place by stopping and starting the instance.

**Diagnose →** `terraform plan` shows `~ instance_type = "m6i.xlarge" -> "m6i.large"`.

**Fix →** If the bigger size is now correct, update code and apply (no-op). If not, schedule the resize in a maintenance window.

**Prevent →** Incident runbooks must include "open a PR to codify any manual change within 24 hours".

</details>

### Q38. Terraform state shows an S3 bucket, but `aws s3api head-bucket` returns 404, and the plan still shows no changes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You ran `terraform plan -refresh=false` or a CI wrapper disables refresh; the provider is pointed at a different region/account from where the bucket was; a provider alias mismatch makes the refresh query the wrong endpoint.

**Diagnose →**
```bash
terraform state show aws_s3_bucket.logs | grep -E 'region|arn'
aws sts get-caller-identity
grep -rn 'refresh=false' .github/ scripts/
```

**Fix →** Plan with refresh enabled using the correct provider configuration. Terraform will then notice the bucket is gone and plan a create.

**Prevent →** Only use `-refresh=false` for speed on very large stacks when you also run scheduled full-refresh drift checks.

</details>

### Q39. `terraform state list` shows `aws_instance.web` but the plan says `aws_instance.web will be destroyed` with `(because aws_instance.web is not in configuration)`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The block was deleted or renamed in code, or it lives in a file that isn't in this directory (moved to a module, or a file renamed to `.tf.bak`).

**Diagnose →** `grep -rn 'resource "aws_instance" "web"' .`

**Fix →** If renamed or moved into a module, add a `moved` block. If intentionally removed from management without destroying it, use a `removed` block with `lifecycle { destroy = false }`. If it should be destroyed, apply.

**Prevent →** Treat any unexpected `destroy` in plan as a blocker in review; add a CI check that fails on destroys unless a label is set.

</details>

### Q40. Plans for the `network` stack are clean, but the `app` stack reading it via `terraform_remote_state` fails with `This object does not have an attribute named "private_subnet_ids"`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The output was added to the `network` code but not applied yet, so it's not in its state; or the app reads the wrong key/workspace.

**Diagnose →**
```bash
cd network && terraform output
terraform state pull | jq '.outputs | keys'
```

**Fix →** Apply the `network` stack so the output is written to state, or fix the `config.key`/`workspace` in the `terraform_remote_state` block.

**Prevent →** Apply producer stacks before consumers in the pipeline. Prefer SSM parameters or data sources with tags for cross-stack contracts.

</details>

### Q41. `Error: Unable to find remote state` from a `terraform_remote_state` data source: `No stored state was found for the given workspace in the given backend.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong bucket/key/region in `config`; the producer uses workspaces (`env:/staging/...`) but the consumer reads the default workspace; the producer has never been applied.

**Diagnose →** `aws s3 ls s3://acme-tfstate/ --recursive | grep network`

**Fix →** Match the exact key, and set `workspace = "staging"` in the data source if the producer uses workspaces.

**Prevent →** Document every stack's state key in one place, or publish shared values to SSM Parameter Store instead.

</details>

### Q42. An EKS managed node group shows drift on `scaling_config[0].desired_size` and launch template version after every cluster upgrade. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cluster Autoscaler/Karpenter changes `desired_size`; console or `eksctl` upgrades bump the launch template version and AMI release outside Terraform.

**Diagnose →** `aws eks describe-nodegroup --cluster-name prod --nodegroup-name general --query 'nodegroup.[scalingConfig,launchTemplate,releaseVersion]'`

**Fix →** `ignore_changes = [scaling_config[0].desired_size]`, and do upgrades through Terraform (`version`, `release_version` or the launch template) instead of the console.

**Prevent →** One upgrade path only. Block `eks:UpdateNodegroupVersion` for humans in prod.

</details>

### Q43. `terraform show` displays a resource with `status: tainted` and the plan says `is tainted, so must be replaced`. Nobody ran `terraform taint`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous apply created the resource but a later step on it failed (provisioner error, health wait timeout). Terraform marks it tainted because it may be half-configured.

**Diagnose →** Check the earlier apply log for the failure on that resource; check the real object's health.

**Fix →** If the object is actually fine, `terraform untaint aws_instance.web`. Otherwise let the plan replace it.

**Prevent →** Avoid provisioners; use user_data/cloud-init or configuration management. Set realistic `timeouts {}` for slow resources.

</details>

### Q44. Drift detection in CI fails every night on a Lambda function's `source_code_hash`, though no one deployed. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ZIP is built in CI and contains timestamps or file permission differences, so its hash changes every build; or the app team deploys Lambda code with another tool.

**Diagnose →** Build the archive twice and compare `sha256sum`; check CloudTrail for `UpdateFunctionCode` from non-Terraform principals.

**Fix →** Make builds reproducible (`archive_file` from a fixed source dir, stable file modes), or split ownership: Terraform manages the function config and `ignore_changes = [source_code_hash, s3_key]`, CI deploys the code.

**Prevent →** Agree which tool owns application code vs infrastructure.

</details>

### Q45. A plan wants to destroy and recreate a whole module after a teammate restored state from a backup taken before a big refactor. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The backup predates `moved`/`state mv` operations, so the restored state uses the old addresses while code uses new ones. If the `moved` blocks were deleted already, Terraform can't map them.

**Diagnose →** `terraform state list | head` and compare to code addresses (`module.network.aws_vpc.this` vs `aws_vpc.main`).

**Fix →** Re-add the `moved` blocks (or run the same `terraform state mv` commands) and re-plan until the destroys disappear. Then import anything created after the backup.

**Prevent →** Keep `moved` blocks in code for a long time (several releases), and take a `state pull` backup right before every refactor, not days earlier.

</details>

### Q46. Plan output shows `# (12 unchanged attributes hidden)` and you can't see why a resource changes. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Terraform hides unchanged attributes by default. The change may be in a nested block or a sensitive value.

**Diagnose →**
```bash
terraform plan -out=tfplan
terraform show -json tfplan | jq '.resource_changes[] | select(.address=="aws_lb.main") | .change | {before, after}'
```

**Fix →** Use the JSON output to see full before/after values and which attributes are `after_unknown`.

**Prevent →** In CI, publish the JSON plan as an artifact and render it with a diff tool for reviewers.

</details>

### Q47. Every plan shows an update for `aws_security_group` and `aws_security_group_rule` alternately adding and removing the same rule. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Inline `ingress {}` blocks in `aws_security_group` and separate rule resources manage the same group. Each one thinks the other's rules are drift.

**Diagnose →** `grep -rn 'security_group_id\s*=\s*aws_security_group.web' .` alongside `ingress {` in the group.

**Fix →** Choose one style. Current best practice: no inline rules, use `aws_vpc_security_group_ingress_rule` / `aws_vpc_security_group_egress_rule`. Move rules with `removed`/`import` to avoid gaps.

**Prevent →** Same rule for route tables (`route {}` vs `aws_route`) and IAM (inline `policy` vs attachment resources).

</details>

### Q48. A new engineer's `terraform plan` shows 50 changes, while CI shows none for the same commit. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Different workspace, different var files or `TF_VAR_*` environment variables, a different provider version (old `.terraform` dir with an unlocked provider), or different credentials pointing at another account.

**Diagnose →**
```bash
terraform workspace show; env | grep ^TF_; aws sts get-caller-identity
terraform version; cat .terraform.lock.hcl | grep -A1 'provider "registry'
```

**Fix →** Align workspace, var files and credentials; `rm -rf .terraform && terraform init`.

**Prevent →** Wrap commands in a `Makefile`/script that selects workspace and var file, and print account ID and workspace at the top.

</details>

### Q49. `terraform plan` shows `aws_route53_record` will be created, but apply fails with `Tried to create resource record set ... but it already exists`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The record was created manually or by another stack (or ExternalDNS in Kubernetes). Terraform doesn't know it exists.

**Diagnose →** `aws route53 list-resource-record-sets --hosted-zone-id Z123 --query "ResourceRecordSets[?Name=='api.example.com.']"`

**Fix →** Import it (`id = "Z123_api.example.com_A"`), or set `allow_overwrite = true` if Terraform should take ownership.

**Prevent →** Split DNS ownership clearly (for example, ExternalDNS owns `*.apps`, Terraform owns everything else).

</details>

### Q50. A nightly drift job reports drift in the morning that disappears by itself in the afternoon. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Scheduled automation changes resources (instance scheduler stopping dev EC2 at night, scaling schedules, Lambda rotating something), or eventual consistency in APIs read during refresh.

**Diagnose →** Correlate drift timestamps with CloudTrail events and EventBridge schedules:
```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=i-0abc123 \
  --start-time 2026-11-08T00:00:00Z --max-results 20
```

**Fix →** `ignore_changes` on attributes owned by the automation, or move the drift job to a time when the environment is stable.

**Prevent →** Inventory every non-Terraform actor that writes to Terraform-managed resources and document ownership per attribute.

</details>

---

## Import, Already Exists and State Surgery

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. Apply fails with `EntityAlreadyExists: Role with name app-role already exists.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role was created manually, by another stack, or by a previous run whose state was lost. IAM names are unique per account.

**Diagnose →** `aws iam get-role --role-name app-role --query 'Role.[Arn,CreateDate,Tags]'` and check tags to find the owner.

**Fix →** If this stack should own it, import:
```hcl
import {
  to = aws_iam_role.app
  id = "app-role"
}
```
If another stack owns it, rename yours or reference it with a `data "aws_iam_role"`.

**Prevent →** Prefix names with stack/env (`${var.env}-app-role`) or use `name_prefix` so stacks can't collide.

</details>

### Q52. `terraform apply` on a new bucket fails with `BucketAlreadyExists`. You have never created that bucket. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** S3 bucket names are global across all AWS accounts; someone else owns the name. (`BucketAlreadyOwnedByYou` means it's in your account.)

**Diagnose →** `aws s3api head-bucket --bucket logs` → 403 means it exists elsewhere, 404 means free.

**Fix →** Choose a unique name, for example `acme-prod-logs-${data.aws_caller_identity.current.account_id}`, or use `bucket_prefix`.

**Prevent →** A naming module that always appends account ID and region to bucket names.

</details>

### Q53. `Error: Resource already managed by Terraform` when running `terraform import aws_s3_bucket.logs acme-logs`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** That address already has an object in state. Terraform refuses to import over it.

**Diagnose →** `terraform state show aws_s3_bucket.logs` — is it the same bucket or a different one?

**Fix →** If it's the same bucket, nothing to do. If the state points to a wrong/deleted object, `terraform state rm aws_s3_bucket.logs` first, then import.

**Prevent →** Use `import` blocks in PRs so the plan shows what will be imported before anything changes.

</details>

### Q54. `Error: Cannot import non-existent remote object` for a resource you can see in the console. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The provider is configured for a different region or account than the console; the ID format is wrong (name vs ARN vs ID); the import targets a resource using a provider alias you didn't specify.

**Diagnose →**
```bash
aws sts get-caller-identity; echo $AWS_REGION
terraform providers
```
Check the resource's "Import" section in the provider docs for the expected ID.

**Fix →** Use the correct ID and the matching provider. With `import` blocks, set `provider = aws.west` when needed.

**Prevent →** Keep a team snippet of ID formats for frequently imported types.

</details>

### Q55. `Error: Configuration for import target does not exist` when running `terraform plan` with an `import` block. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `to` address has no matching `resource` block (typo, different name, missing module path or instance key).

**Diagnose →** Compare `to = aws_instance.app` with `grep -rn 'resource "aws_instance"' .`

**Fix →** Write the resource block, or let Terraform generate it:
```bash
terraform plan -generate-config-out=generated.tf
```
Then clean up `generated.tf` and move it into proper files.

**Prevent →** Review generated config before merging; it often contains defaults and conflicting arguments.

</details>

### Q56. `terraform plan -generate-config-out=generated.tf` fails with `Conflicting configuration arguments` in the generated file. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The generator writes every attribute it reads, including mutually exclusive ones (for example `ipv6_cidr_block` and `ipv6_ipam_pool_id`, or both `name` and `name_prefix`).

**Diagnose →** Read each error's attribute pair.

**Fix →** Delete the conflicting/computed arguments from `generated.tf` (keep the one you actually set), then run `terraform plan` again. The file remains; re-running with the same `-generate-config-out` path errors because the file exists.

**Prevent →** Treat generated config as a draft. Refactor it into variables and modules before merge.

</details>

### Q57. After importing an S3 bucket, the plan still shows `+ force_destroy = false` as an update. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `force_destroy` is a Terraform-only argument; it doesn't exist in AWS, so it can't be imported. The first apply just writes it to state.

**Diagnose →** The diff touches only Terraform-only arguments; no API call changes the bucket.

**Fix →** Apply; it's a state-only update. Same for things like `skip_final_snapshot` on RDS.

**Prevent →** Know which arguments are provider-side only and explain them in the import PR.

</details>

### Q58. You imported an IAM role and the plan wants to remove two managed policy attachments that prod depends on. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The code uses `managed_policy_arns` (exclusive management) or `aws_iam_role_policy_attachments_exclusive`, so Terraform removes attachments not listed in code.

**Diagnose →** `aws iam list-attached-role-policies --role-name app-role`

**Fix →** Add all real attachments to code (or import the existing `aws_iam_role_policy_attachment` resources) before applying. Note `managed_policy_arns` is deprecated in AWS provider v5; prefer separate attachment resources.

**Prevent →** After every import, require a plan with zero changes before merging.

</details>

### Q59. `terraform import 'aws_instance.web["api"]' i-0abc` fails in bash with `Error: Index value required` or the address looks mangled. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Shell quoting. Without single quotes, bash strips the inner double quotes, so Terraform receives `aws_instance.web[api]`.

**Diagnose →** `echo aws_instance.web["api"]` prints the mangled address.

**Fix →**
```bash
terraform import 'aws_instance.web["api"]' i-0abc123          # bash/zsh
terraform import 'aws_instance.web[\"api\"]' i-0abc123        # PowerShell
```
Or avoid quoting entirely with an `import` block in code.

**Prevent →** Use `import` blocks and `moved` blocks rather than CLI addresses.

</details>

### Q60. You ran `terraform state rm aws_db_instance.main` to "fix" a plan error. The next apply fails with `DBInstanceAlreadyExists`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `state rm` only forgets the object; the database still exists. The code still declares it, so Terraform tries to create a new one with the same identifier.

**Diagnose →** `aws rds describe-db-instances --db-instance-identifier prod-db`

**Fix →** Re-import it:
```bash
terraform import aws_db_instance.main prod-db
terraform plan
```

**Prevent →** Use `state rm` only when you also remove the block (or use a `removed` block, which does both in one reviewed change).

</details>

### Q61. After `terraform state mv aws_instance.web module.app.aws_instance.web`, the plan wants to destroy the instance and create a new one. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The destination address doesn't match the code (module is called `module.application`, or the resource uses `count` so it needs `[0]`); the state now has the object at an address the code doesn't declare.

**Diagnose →** `terraform state list | grep aws_instance` vs the code address shown in the plan's `+ create`.

**Fix →** Move again to the exact address in the plan:
```bash
terraform state mv 'module.app.aws_instance.web' 'module.application.aws_instance.web[0]'
```

**Prevent →** Prefer `moved` blocks; `terraform plan` shows `has moved to` and you see mistakes before state changes.

</details>

### Q62. `Error: Moved object still exists` after adding a `moved` block. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `from` address is still declared in code, so it isn't actually moved (you copied the block instead of renaming it).

**Diagnose →** `grep -rn 'resource "aws_s3_bucket" "logs"' .` finds both old and new blocks.

**Fix →** Delete or rename the old resource block so only the `to` address exists.

**Prevent →** Rename and add the `moved` block in the same commit; review that the old address disappears.

</details>

### Q63. A `moved` block from `null_resource.bootstrap` to `terraform_data.bootstrap` fails with a resource type mismatch on an old Terraform version. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cross-type moves need Terraform 1.8+ and provider support. `null_resource` → `terraform_data` is supported from 1.9; older versions reject any move between different types.

**Diagnose →** `terraform version` locally and in CI.

**Fix →** Upgrade to 1.9+ everywhere and keep the `moved` block. On older versions, use `removed { lifecycle { destroy = false } }` plus a new `terraform_data` (which will re-run its provisioner) or accept the replacement.

**Prevent →** Check the minimum Terraform version for refactor features and bump `required_version` in the same PR.

</details>

### Q64. A `moved` block inside your module referencing a resource in a registry module fails with `Cross-package move statement`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `moved` can only refer to objects within your module package; you can't move into or out of a separately installed module's internals from inside it.

**Diagnose →** The error names the external package source.

**Fix →** Put the `moved` block in the **calling** module, using full module paths:
```hcl
moved {
  from = aws_vpc.main
  to   = module.vpc.aws_vpc.this[0]
}
```

**Prevent →** Do refactors that cross module boundaries in the root module, and look up the registry module's real resource names and instance keys first.

</details>

### Q65. You need to move a resource from stack A's state to stack B's state. `terraform state mv` with `-state-out` isn't working with the S3 backend. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `-state` / `-state-out` work on local files only; with remote backends you must use pull/push or a declarative approach.

**Diagnose →** The command complains about local state options or updates only a local file.

**Fix →** The safest approach in 1.7+: in stack A add `removed { from = aws_sqs_queue.jobs  lifecycle { destroy = false } }` and apply; in stack B add the resource plus an `import` block and apply. No hand-edited state, and both steps are reviewed plans.

**Prevent →** Plan stack boundaries early; when moving, always use the removed + import pattern.

</details>

### Q66. `removed` block plan fails with `Error: Removed resource block still exists`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The resource block is still in code; a `removed` block requires the block to be deleted.

**Diagnose →** `grep -rn 'aws_sqs_queue" "jobs"' .`

**Fix →** Delete the resource block, keep only:
```hcl
removed {
  from = aws_sqs_queue.jobs
  lifecycle {
    destroy = false
  }
}
```

**Prevent →** Remember `removed` needs Terraform 1.7+; check CI's version too.

</details>

### Q67. Importing an `aws_iam_role_policy_attachment` fails with `unexpected format for ID`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Many "association" resources use composite IDs, not a single AWS ID. For this one it's `role-name/policy-arn`.

**Diagnose →** Check the "Import" section at the bottom of the resource's provider documentation page.

**Fix →**
```hcl
import {
  to = aws_iam_role_policy_attachment.ssm
  id = "app-role/arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}
```
Other examples: `aws_route_table_association` uses `subnet-id/rtb-id`; `aws_security_group_rule` has a long composite ID.

**Prevent →** Keep a cheatsheet of composite IDs for commonly imported resources.

</details>

### Q68. You need to import 60 SQS queues created by hand. Writing 60 import blocks is error-prone. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Not an error, but a common place to make address mistakes.

**Diagnose →** List the real queues: `aws sqs list-queues --queue-name-prefix orders- --query QueueUrls`

**Fix →** Use `for_each` on `import` blocks (Terraform 1.7+), driven by the same map as the resource:
```hcl
locals {
  queues = toset(["orders-created", "orders-paid", "orders-shipped"])
}
resource "aws_sqs_queue" "q" {
  for_each = local.queues
  name     = each.key
}
import {
  for_each = local.queues
  to       = aws_sqs_queue.q[each.key]
  id       = "https://sqs.ap-south-1.amazonaws.com/111122223333/${each.key}"
}
```

**Prevent →** Delete import blocks after a clean apply, or leave them; they're no-ops once imported.

</details>

### Q69. Import finished, but now `terraform plan` shows a forced replacement on the imported resource. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An argument in code that forces replacement differs from reality: `availability_zone`, `subnet_id`, `name`, `engine`, `kms_key_id`, or `user_data` with `user_data_replace_on_change = true`.

**Diagnose →** `terraform plan | grep -B2 '# forces replacement'`, then `terraform state show <addr>` to see real values.

**Fix →** Change code to match the imported values exactly. Never apply an import PR that shows a replace.

**Prevent →** Add a CI rule: PRs containing `import` blocks must show zero `delete` and zero `update` actions.

</details>

### Q70. `terraform state rm module.legacy` was run in the wrong workspace (prod instead of staging). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong workspace selected, or `TF_WORKSPACE` set in the shell.

**Diagnose →** `terraform workspace show`; `terraform plan` in prod now shows `module.legacy` resources to be **created**.

**Fix →** Do not apply. Restore the previous state version from S3 versioning (`env:/prod/...` key), or re-import every removed resource. With a backup:
```bash
terraform state push -force prod-before-rm.json
```

**Prevent →** Take `terraform state pull > backup-$(date +%s).json` before any state command, and show the workspace in your shell prompt.

</details>

### Q71. `terraform state mv` returns `Error: Invalid target address` when moving between `count` and `for_each` instances. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing quotes around string keys, or using `[0]` where the code now uses `for_each` keys.

**Diagnose →** `terraform state list | grep aws_subnet`

**Fix →**
```bash
terraform state mv 'aws_subnet.private[0]' 'aws_subnet.private["ap-south-1a"]'
```
Better: a `moved` block per instance, which can be reviewed in a plan.

**Prevent →** When converting `count` → `for_each`, generate `moved` blocks from a list in one PR.

</details>

### Q72. You imported a resource into the default workspace instead of `staging`. Now default has a stray object. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Forgot `terraform workspace select staging` before importing.

**Diagnose →** `terraform workspace list`, then `terraform state list` in each.

**Fix →** In default: `terraform state rm <addr>` (only forgets, doesn't delete). In staging: import it again.

**Prevent →** Use separate directories/backends per environment instead of workspaces for long-lived envs, or a wrapper that requires an explicit `ENV=` argument.

</details>

### Q73. `Error: resource aws_xyz doesn't support import`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Some resources have no read API or the provider never implemented import (common for "action" resources and some older ones).

**Diagnose →** Check the provider docs and changelog; a newer provider version may add import support.

**Fix →** Upgrade the provider if support was added. Otherwise recreate under Terraform (planned window), or manage it outside Terraform and reference it via a data source.

**Prevent →** Prefer resources that support import when choosing between equivalent resource types.

</details>

### Q74. After importing an EC2 instance, every plan shows `user_data` changes or a replacement. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The provider stores a hash/base64 of user data; your code's template renders differently (trailing newline, different variables). With `user_data_replace_on_change = true` this forces replacement.

**Diagnose →**
```bash
aws ec2 describe-instance-attribute --instance-id i-0abc123 --attribute userData \
  --query 'UserData.Value' --output text | base64 -d > real.sh
diff real.sh <(terraform console <<< 'local.user_data')
```

**Fix →** Make the template output byte-identical, or `ignore_changes = [user_data]` for instances where user data only matters at first boot.

**Prevent →** For long-lived instances, keep user data minimal and push config via SSM/Ansible.

</details>

### Q75. A teammate edited the state JSON to change a resource ID, pushed it, and now refresh fails for that resource with a provider error. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Only `id` was changed; other attributes (ARN, nested IDs) still reference the old object, or `schema_version` / private provider data no longer match, so the provider can't read it.

**Diagnose →** `terraform state show <addr>`; compare `id` with `arn` and other identifiers.

**Fix →** Remove and re-import cleanly:
```bash
terraform state rm aws_lb.main
terraform import aws_lb.main arn:aws:elasticloadbalancing:ap-south-1:111122223333:loadbalancer/app/prod/abc123
```

**Prevent →** No manual state edits; reviewers should reject PRs or runbooks that suggest them.

</details>

---

## Dependencies, Cycles and Providers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. `Error: Cycle: aws_security_group.app, aws_security_group.db`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Each group's inline `ingress` references the other group's ID, so neither can be created first.

**Diagnose →**
```bash
terraform graph -type=plan | dot -Tsvg > graph.svg
```
Look for the two nodes pointing at each other.

**Fix →** Create both groups without inline rules, then add rules as separate resources:
```hcl
resource "aws_vpc_security_group_ingress_rule" "db_from_app" {
  security_group_id            = aws_security_group.db.id
  referenced_security_group_id = aws_security_group.app.id
  from_port                    = 5432
  to_port                      = 5432
  ip_protocol                  = "tcp"
}
```

**Prevent →** Standardize on standalone rule resources for all security groups.

</details>

### Q77. `Error: Cycle: local.name_prefix, local.tags`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two locals reference each other (directly or through a third).

**Diagnose →** Read both definitions; the cycle list tells you every node involved.

**Fix →** Break the loop by extracting the shared part into a third local that depends on neither.

**Prevent →** Keep locals layered: inputs → derived names → tags/maps, never back-references.

</details>

### Q78. A cycle appears only after adding `depends_on = [module.app]` to a module. The error lists dozens of nodes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Module-level `depends_on` makes **every** resource and data source in the module depend on **everything** in the other module. If any value flows back (an output of this module is used by `module.app`), you get a cycle.

**Diagnose →** The cycle list contains nodes from both modules; check which outputs cross between them.

**Fix →** Remove the module `depends_on` and pass the actual values (IDs, ARNs) as inputs; Terraform then infers precise dependencies.

**Prevent →** Treat module `depends_on` as a last resort; it also delays data sources and causes "known after apply" noise.

</details>

### Q79. Adding `create_before_destroy = true` to a launch template causes `Error: Cycle: ... (destroy deposed ...)`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `create_before_destroy` propagates to dependencies: a resource it depends on doesn't have it, so Terraform must destroy the dependency before creating the new object and destroy the dependent first. The graph conflicts.

**Diagnose →** `terraform graph -type=apply` and look for `(destroy deposed)` nodes in the cycle.

**Fix →** Add `create_before_destroy = true` to the dependencies in the chain too (and use `name_prefix` so old and new can coexist).

**Prevent →** Design replaceable chains (LT → ASG → target group attachments) with consistent lifecycle settings from the start.

</details>

### Q80. `terraform init` fails: `Could not retrieve the list of available versions for provider hashicorp/aws: no available releases match the given constraints ~> 4.0, >= 5.30.0`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Root requires AWS provider 5.x, but a child module pins `~> 4.0`. Terraform needs one version satisfying every module.

**Diagnose →**
```bash
terraform providers
```
The tree shows which module declares which constraint.

**Fix →** Upgrade the module to a release that supports AWS provider 5.x (`version = "~> 5.0"` on the registry module), or fork and relax its constraint.

**Prevent →** Modules should use loose lower bounds (`>= 5.0`); only root modules pin tightly. Bump modules and providers together with Renovate.

</details>

### Q81. `Error: Inconsistent dependency lock file` with `locked version selection 5.31.0 doesn't match the updated version constraints ">= 5.40.0"`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You raised the provider constraint in `required_providers`, but `.terraform.lock.hcl` still records the old version.

**Diagnose →** `grep -A3 'hashicorp/aws' .terraform.lock.hcl`

**Fix →**
```bash
terraform init -upgrade
git add .terraform.lock.hcl
```

**Prevent →** Always commit the lock file in the same PR as constraint changes.

</details>

### Q82. `Error: Inconsistent dependency lock file` with `provider registry.terraform.io/hashicorp/random: required by this configuration but no version is selected`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A new provider was introduced (directly or by a new module) and `terraform init` wasn't re-run, or the lock file in CI is older than the code.

**Diagnose →** `terraform providers` shows `random` but the lock file lacks it.

**Fix →** `terraform init` locally (adds the entry), commit `.terraform.lock.hcl`.

**Prevent →** CI should run `terraform init -lockfile=readonly` so missing entries fail early in PR checks, not at apply time.

</details>

### Q83. CI on Linux fails: `the current package for registry.terraform.io/hashicorp/aws 5.72.1 doesn't match any of the checksums previously recorded in the dependency lock file`. Works on Mac laptops. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The lock file was generated on `darwin_arm64` and only contains `h1:` hashes for that platform (when installed from a mirror/cache); CI downloads `linux_amd64`.

**Diagnose →** Count hashes: `grep -A20 'hashicorp/aws' .terraform.lock.hcl`.

**Fix →**
```bash
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64 -platform=darwin_amd64 -platform=windows_amd64
```
Commit the updated lock file.

**Prevent →** Make the multi-platform `providers lock` command part of your upgrade script or Renovate post-upgrade task.

</details>

### Q84. On an Apple Silicon Mac: `Provider registry.terraform.io/hashicorp/template v2.2.0 does not have a package available for your current platform, darwin_arm64`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `template` provider is archived and was never built for `darwin_arm64`. An old module still uses `data "template_file"`.

**Diagnose →** `terraform providers` shows which module requires `hashicorp/template`.

**Fix →** Replace `template_file` with the built-in `templatefile()` function, and upgrade/fork the module that uses it.

**Prevent →** Remove archived providers (`template`, old community ones) during regular dependency hygiene.

</details>

### Q85. `provider registry registry.terraform.io does not have a provider named registry.terraform.io/hashicorp/kubectl`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A non-HashiCorp provider is used without a `source` in `required_providers`, so Terraform assumes `hashicorp/<name>`. Often happens in a child module that forgot to declare it.

**Diagnose →** The error says `Did you intend to use ...?` and names the module requiring it.

**Fix →** In **every** module that uses it:
```hcl
terraform {
  required_providers {
    kubectl = {
      source  = "alekc/kubectl"
      version = "~> 2.0"
    }
  }
}
```

**Prevent →** Every module declares all its providers with `source`; `tflint` catches missing ones.

</details>

### Q86. `Error: Provider configuration not present` — `To work with module.dr.aws_s3_bucket.replica (orphan) its original provider configuration ... is required, but it has been removed.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You removed a module call (or a provider alias like `aws.dr`) while its resources are still in state. Terraform needs that provider configuration to destroy them.

**Diagnose →** `terraform state list | grep module.dr`

**Fix →** Two-step removal: put the provider block (and module call, if removed) back, apply with the module removed so resources are destroyed, then delete the provider alias in a follow-up PR. If resources should survive, use `removed` blocks first.

**Prevent →** Never delete a provider configuration in the same change that removes the resources using it.

</details>

### Q87. `Error: Module is incompatible with count, for_each, and depends_on` — `The module at module.legacy is a legacy module which contains its own local provider configurations`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The child module has `provider "aws" { ... }` blocks inside it. Modules with their own provider blocks can't be used with `count`/`for_each`/`depends_on`.

**Diagnose →** `grep -rn '^provider ' modules/legacy/`

**Fix →** Move provider configuration to the root, declare `configuration_aliases` in the module if it needs multiple, and pass providers in:
```hcl
module "legacy" {
  source    = "./modules/legacy"
  for_each  = var.regions
  providers = { aws = aws.mumbai }
}
```
Note: you still can't choose a provider per `for_each` instance dynamically; one module call per provider is needed.

**Prevent →** Module guideline: no provider blocks in reusable modules, only `required_providers`.

</details>

### Q88. Creating an EKS cluster and Kubernetes resources in one stack: `Error: Get "http://localhost/api/v1/namespaces/app": dial tcp [::1]:80: connect: connection refused`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `kubernetes` provider was configured from values unknown at plan time (cluster not created yet, or being replaced), so it fell back to defaults (`localhost`). Also happens when the cluster is replaced in the same plan.

**Diagnose →** Is the cluster being created/replaced in this plan? Does the provider use `module.eks.cluster_endpoint`?

**Fix →** Apply in two stages (cluster stack first, workloads stack second), and configure the provider with exec auth:
```hcl
provider "kubernetes" {
  host                   = data.aws_eks_cluster.this.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.this.certificate_authority[0].data)
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", "prod"]
  }
}
```

**Prevent →** Keep cluster infrastructure and in-cluster resources in separate states.

</details>

### Q89. Long applies against EKS fail midway with `Error: Unauthorized` from the Kubernetes/Helm provider. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The provider uses `data "aws_eks_cluster_auth"` token, which expires after about 15 minutes. Long applies outlive it.

**Diagnose →** The failure appears roughly 15 minutes into the apply; earlier Kubernetes resources succeeded.

**Fix →** Switch the `kubernetes` and `helm` providers to the `exec` block (`aws eks get-token`) so a fresh token is fetched as needed.

**Prevent →** Use `exec` auth as the default pattern in all modules and examples.

</details>

### Q90. `Error: Kubernetes cluster unreachable: invalid configuration: no configuration has been provided` from the Helm provider. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `helm` provider block has no `kubernetes {}` config (in Helm provider v3 it's a `kubernetes = { ... }` attribute instead of a block), no `KUBECONFIG`, or the config path points to a missing file in CI.

**Diagnose →** `grep -A10 'provider "helm"' *.tf`; check provider major version in the lock file.

**Fix →** Configure it explicitly with host/CA/exec using the syntax matching your Helm provider major version.

**Prevent →** Pin provider major versions and read upgrade guides for major bumps (Helm provider 3.x changed block syntax).

</details>

### Q91. `Provider "registry.terraform.io/hashicorp/aws" requires explicit configuration` / `The argument "region" is required, but was not set`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No `region` in the provider block and no `AWS_REGION`/`AWS_DEFAULT_REGION` or profile region in this environment (common in fresh CI runners).

**Diagnose →** `env | grep -i region; aws configure get region`

**Fix →** Set `region = var.region` in the provider block, or export `AWS_REGION` in the pipeline.

**Prevent →** Always set region explicitly in code; don't depend on per-machine defaults.

</details>

### Q92. A data source reads the wrong AMI/VPC: `data "aws_vpc"` returns a VPC from `us-east-1` though resources go to `ap-south-1`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Multiple provider configurations; the data source uses the default provider while resources use `provider = aws.mumbai`.

**Diagnose →** `terraform providers` and `grep -n 'provider\s*=' *.tf`

**Fix →** Set `provider = aws.mumbai` on the data source too.

**Prevent →** In multi-region stacks, avoid an unaliased default provider, so every resource and data source must pick one explicitly.

</details>

### Q93. A module uses `provider = aws.us_east_1` for an ACM certificate (CloudFront) and fails: the module doesn't receive that provider. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The module didn't declare the alias in `configuration_aliases`, or the caller didn't pass it in `providers`.

**Diagnose →** Check the module's `required_providers` and the call's `providers = {}` map.

**Fix →**
```hcl
# in the module
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      configuration_aliases = [aws.us_east_1]
    }
  }
}
# in the root
module "cdn" {
  source    = "./modules/cdn"
  providers = { aws = aws, aws.us_east_1 = aws.us_east_1 }
}
```

**Prevent →** Document required provider aliases in each module's README.

</details>

### Q94. `terraform init` fails behind a firewall: `could not connect to registry.terraform.io: failed to request discovery document`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No outbound internet, a proxy without its CA trusted, or a blocked `releases.hashicorp.com` (providers download from there after registry lookup).

**Diagnose →**
```bash
curl -sv https://registry.terraform.io/.well-known/terraform.json
curl -sI https://releases.hashicorp.com/
```

**Fix →** Configure `HTTPS_PROXY` and trust the proxy CA, or use a provider mirror:
```hcl
# ~/.terraformrc
provider_installation {
  network_mirror {
    url = "https://tf-mirror.internal.example.com/"
  }
}
```

**Prevent →** For air-gapped runners, prepare mirrors with `terraform providers mirror ./mirror` and bake them into the runner image.

</details>

### Q95. Parallel CI jobs sharing `TF_PLUGIN_CACHE_DIR` fail intermittently with `text file busy` or corrupt provider zips. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The plugin cache is not safe for concurrent writes; two `terraform init` runs write the same provider at once.

**Diagnose →** Failures only when jobs run in parallel on the same host.

**Fix →** Give each job its own cache dir, or pre-populate the cache in the image (read-only use is safe). Use CI cache keyed on `.terraform.lock.hcl` hash.

**Prevent →** Treat the plugin cache as per-job or pre-baked, never shared-writable across concurrent jobs.

</details>

### Q96. `Error: Required plugins are not installed` — `there is no package for registry.terraform.io/hashicorp/aws 5.72.1 cached in .terraform/providers`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The lock file was updated (pulled new commits) but `.terraform/providers` still has the old version; or the CI job restored a stale `.terraform` cache.

**Diagnose →** `ls .terraform/providers/registry.terraform.io/hashicorp/aws/`

**Fix →** `terraform init` (not `-upgrade`; the lock file already chooses the version).

**Prevent →** Cache providers by lock-file hash, and don't cache the whole `.terraform` directory.

</details>

### Q97. `depends_on` on a data source makes every plan show `(known after apply)` for things that never change. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A data source with `depends_on` (or inside a module with module-level `depends_on`) is deferred until apply whenever the dependency has **any** pending change. All values derived from it become unknown, causing noisy diffs and sometimes replacements.

**Diagnose →** In the plan, look for `<= data "..." ... (depends on a resource or a module with changes pending)` lines.

**Fix →** Remove `depends_on` and make the data source reference a specific attribute (for example `vpc_id = aws_vpc.main.id`) so the dependency is precise.

**Prevent →** Ban module-level `depends_on` in review unless documented.

</details>

### Q98. `Error: Invalid provider configuration reference` or `Error: Duplicate provider configuration` after copying files between stacks. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two `provider "aws"` blocks without aliases, or a resource refers to `aws.west` that doesn't exist in this stack.

**Diagnose →** `grep -rn 'provider "aws"' *.tf` and `grep -rn 'provider = aws\.' *.tf`

**Fix →** Keep one unaliased default; give extras `alias = "west"`, and fix references.

**Prevent →** Keep provider blocks in a single `providers.tf` per root module.

</details>

### Q99. A provider minor upgrade (5.60 → 5.70) causes plans to show unexpected attribute changes on many resources. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New defaults or new computed attributes; a bug fix changed how a value is normalized; deprecated arguments changed behaviour.

**Diagnose →** Read the provider `CHANGELOG.md` entries for the jump; run the plan with old and new versions to isolate.

**Fix →** If harmless (state-only), apply. If behaviour changed, set the argument explicitly or pin the previous minor and open an upstream issue.

**Prevent →** Upgrade providers in small steps through Renovate PRs, with plan output for every stack attached.

</details>

### Q100. After upgrading the AWS provider to 6.x, plans fail with `Unsupported argument` on arguments that worked before. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Major provider versions remove deprecated arguments and resources. Code used things marked deprecated in 5.x.

**Diagnose →** Read the provider's version 6 upgrade guide; before upgrading, `terraform validate` on 5.x shows `Warning: Argument is deprecated` for everything that will break.

**Fix →** Pin back to `~> 5.0` to unblock, fix the deprecations, then upgrade one stack at a time.

**Prevent →** Treat deprecation warnings as tech debt with an owner; pin majors in root modules (`~> 5.0`) so upgrades are deliberate.

</details>

---

## Plan Errors and Unexpected Replacements

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `Error: Invalid for_each argument` — `The "for_each" map includes keys derived from resource attributes that cannot be determined until apply`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Keys come from something created in the same run, such as `for_each = toset(aws_subnet.private[*].id)` or `{ for i in aws_instance.web : i.id => i }`.

**Diagnose →** Find where the key is computed; any resource attribute marked `(known after apply)` can't be a key.

**Fix →** Use static keys you control and put apply-time values in the map **values**:
```hcl
resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private          # keys are var-driven AZ names
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private.id
}
```
`-target` to create dependencies first works but is a workaround, not a design.

**Prevent →** Rule of thumb: `for_each` keys come from variables/locals, never from resource IDs.

</details>

### Q102. `The given "for_each" argument value is unsuitable: the "for_each" argument must be a map, or set of strings, and you have provided a value of type list of string.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You passed a list (`var.users`) directly.

**Diagnose →** `terraform console` → `type(var.users)`.

**Fix →** `for_each = toset(var.users)`, or build a map keyed by a stable field: `{ for u in var.users : u.name => u }`.

**Prevent →** Declare variables as `set(string)` or `map(object(...))` when they drive `for_each`.

</details>

### Q103. `Error: Invalid for_each argument` — `Sensitive values, or values derived from sensitive values, cannot be used as for_each arguments.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The map came from a `sensitive = true` variable or from a secret data source; even the keys become sensitive.

**Diagnose →** Trace the value back to a sensitive variable/output.

**Fix →** Iterate over non-secret keys only:
```hcl
for_each = nonsensitive(toset(keys(var.db_users)))
# use var.db_users[each.key] inside for the secret values
```
Only use `nonsensitive()` when the keys are truly not secret.

**Prevent →** Split structures: a non-sensitive map of names/config and a separate sensitive map of secrets.

</details>

### Q104. `Error: Invalid count argument` — `The "count" value depends on resource attributes that cannot be determined until apply`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `count = length(aws_instance.web[*].id)` or `count = var.enabled && aws_vpc.main.id != "" ? 1 : 0` uses an unknown value.

**Diagnose →** Find the unknown input in the `count` expression.

**Fix →** Drive `count` from a known value: `count = var.create_alarm ? 1 : 0` or `length(var.instance_names)`.

**Prevent →** Never compare resource attributes to decide whether to create something; use explicit boolean flags.

</details>

### Q105. `Error: Unsupported attribute` — `This object has no argument, nested block, or exported attribute named "public_ip_address"`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong attribute name (Azure-ism in AWS code; the AWS name is `public_ip`), or an attribute removed in a provider upgrade.

**Diagnose →** Check the resource's "Attribute Reference" docs, or `terraform state show aws_instance.web`.

**Fix →** Use the correct attribute: `aws_instance.web.public_ip`.

**Prevent →** Rely on editor tooling (terraform-ls) and `terraform validate` in pre-commit.

</details>

### Q106. `Error: Unsupported attribute` on `module.vpc.private_subnets` — `This object does not have an attribute named "private_subnets"`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The module doesn't declare that output, or it's named differently (`private_subnet_ids`), or you upgraded the module and the output was renamed.

**Diagnose →** `grep -rn '^output' .terraform/modules/vpc/outputs.tf` (for registry modules) or the module's README.

**Fix →** Use the correct output name, or add the output to your module.

**Prevent →** Treat module outputs as an API; changing them is a breaking change with a major version bump.

</details>

### Q107. `Error: Missing resource instance key` — `Because aws_instance.web has "count" set, its attributes must be accessed on specific instances.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You added `count` (or `for_each`) to a resource but references still use `aws_instance.web.id`.

**Diagnose →** `grep -rn 'aws_instance.web\.' .`

**Fix →** Use an index or splat: `aws_instance.web[0].id`, `aws_instance.web[*].id`, or `values(aws_instance.web)[*].id` for `for_each`. Add a `moved` block from `aws_instance.web` to `aws_instance.web[0]` so it isn't replaced.

**Prevent →** When introducing `count`/`for_each`, update references and add `moved` blocks in the same PR.

</details>

### Q108. `Error: Reference to undeclared resource` — `A managed resource "aws_vpc" "main" has not been declared in the root module.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The VPC is defined inside a module (so you need `module.network.vpc_id`), the resource name differs (`aws_vpc.this`), or the file defining it was deleted/renamed with a non-`.tf` extension.

**Diagnose →** `grep -rn 'resource "aws_vpc"' .`

**Fix →** Reference the right address or module output; if the VPC exists outside this stack, use `data "aws_vpc"`.

**Prevent →** Run `terraform validate` in pre-commit and PR checks.

</details>

### Q109. Plan shows `# aws_instance.web must be replaced` with `~ ami = "ami-0aaa" -> "ami-0bbb" # forces replacement`, but nobody changed the AMI. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `data "aws_ami"` with `most_recent = true` picked up a newly published image.

**Diagnose →** `terraform state show data.aws_ami.al2023 | grep -E 'id|creation_date'`

**Fix →** Pin the AMI ID in a variable updated by a deliberate PR, or `lifecycle { ignore_changes = [ami] }` and roll instances via ASG instance refresh instead.

**Prevent →** For fleets, use launch templates + ASGs so a new AMI rolls gradually instead of replacing instances during an apply.

</details>

### Q110. Removing one item from the middle of a list makes the plan destroy and recreate every subsequent resource. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `count` with a list: instances are addressed by index, so removing `[1]` shifts `[2]` → `[1]` and so on. Terraform sees changed arguments at each index.

**Diagnose →** The plan shows `aws_iam_user.u[1]`, `[2]`, `[3]` being changed/replaced.

**Fix →** Switch to `for_each = toset(var.users)` with `moved` blocks from each index to its key, then remove the item.

**Prevent →** Use `count` only for "0 or 1" or truly identical copies; use `for_each` for collections.

</details>

### Q111. Renaming a security group's `name` forces replacement, and the apply fails with `InvalidGroup.Duplicate` or hangs on `DependencyViolation`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `name` forces a new group. Default order is destroy-then-create, but the old group is attached to ENIs and can't be deleted. With `create_before_destroy`, the same name conflicts.

**Diagnose →**
```bash
aws ec2 describe-network-interfaces --filters Name=group-id,Values=sg-0abc123 \
  --query 'NetworkInterfaces[].[NetworkInterfaceId,Description]'
```

**Fix →** Use `name_prefix` plus `create_before_destroy = true` so a new group is created, attached, then the old one deleted. Or revert the rename (description/tags may be enough).

**Prevent →** Security groups should use `name_prefix` and lifecycle CBD from day one.

</details>

### Q112. Plan says `aws_db_instance.main must be replaced` after changing `storage_encrypted = true`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Encryption at rest can't be enabled in place on RDS; the provider marks it `forces replacement`. Applying would create a new empty DB and delete the old one.

**Diagnose →** `terraform plan | grep -B2 'forces replacement'`

**Fix →** Revert the change. Migrate deliberately: snapshot → copy snapshot with KMS encryption → restore to a new instance (in code, `snapshot_identifier`) → switch the app → remove the old one.

**Prevent →** `lifecycle { prevent_destroy = true }` and `deletion_protection = true` on stateful resources, and a CI gate that blocks any `delete` on RDS/S3/EFS/DynamoDB.

</details>

### Q113. `Error: Instance cannot be destroyed` — `Resource aws_s3_bucket.state has lifecycle.prevent_destroy set, but the plan calls for this resource to be destroyed.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The guard is working: your change causes a destroy or replace (rename, `for_each` key change, a forcing argument).

**Diagnose →** Find what triggers the destroy in the plan.

**Fix →** Fix the code (add `moved`, revert the forcing change). Only remove `prevent_destroy` if destroying is truly intended and approved.

**Prevent →** Keep `prevent_destroy` on state buckets, databases, KMS keys and DNS zones.

</details>

### Q114. Every plan shows `~ tags = { "LastUpdated" = "2026-11-08T10:00:00Z" -> (known after apply) }`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `timestamp()` (or `uuid()`) in tags or names; it's evaluated each run, so there's always a diff.

**Diagnose →** `grep -rn -E 'timestamp\(\)|uuid\(\)' .`

**Fix →** Remove it. If you need a creation time, use `time_static` from the `time` provider, which is stored once in state.

**Prevent →** Lint against non-deterministic functions in resource arguments.

</details>

### Q115. `Error: Inconsistent conditional result types` — `The true and false result expressions must have consistent types.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `var.enabled ? { a = 1, b = "x" } : {}` or two objects with different attributes; Terraform can't unify the types.

**Diagnose →** Read which attribute is absent in one branch.

**Fix →** Make both branches the same shape, or use `merge()`/`tomap()`:
```hcl
locals {
  extra = var.enabled ? { backup = "daily" } : tomap({})
}
```

**Prevent →** Use typed variables (`map(string)`) so values are converted consistently.

</details>

### Q116. `Error: Invalid index` — `The given key does not identify an element in this collection value: the collection has no elements.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Accessing `[0]` on an empty list: a data source with no matches (`data.aws_subnets.private.ids[0]`), or a resource with `count = 0`.

**Diagnose →** `terraform console` → `data.aws_subnets.private.ids` and check filters/tags.

**Fix →** Fix the filter, or guard: `one(aws_eip.nat[*].public_ip)` (null when empty) or `try(x[0], null)`. For required inputs, add a `postcondition` with a clear message.

**Prevent →** Add `lifecycle { postcondition { condition = length(self.ids) > 0 ... } }` on critical data sources.

</details>

### Q117. `Error: Invalid function argument` — `no file exists at "./scripts/bootstrap.sh"; this function works only with files that are distributed as part of the configuration source code`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Paths are relative to the **working directory**, not the module. Inside a module, `./scripts/...` points to the root's folder.

**Diagnose →** Check where the file lives and where `terraform` runs.

**Fix →** `file("${path.module}/scripts/bootstrap.sh")` or `templatefile("${path.module}/templates/user_data.sh.tftpl", {...})`.

**Prevent →** Always prefix file paths with `path.module` in modules.

</details>

### Q118. A plan wants to replace an EKS cluster after changing `subnet_ids`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Depending on provider and API support, some `vpc_config` changes require a new cluster; a module upgrade may also change subnet ordering or the cluster name.

**Diagnose →** `terraform plan | grep -B3 'forces replacement'` inside `module.eks`.

**Fix →** Stop. Revert the change, or check whether AWS now allows updating it via `aws eks update-cluster-config` and wait for provider support. For real migrations, build a second cluster and migrate workloads (blue/green).

**Prevent →** Gate any plan containing `delete` on `aws_eks_cluster`, RDS, or other stateful/critical resources.

</details>

### Q119. A `for_each` resource is replaced after changing a key from `"web-1"` to `"web_1"` for naming consistency. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `for_each` keys are part of the resource address; a new key means a new object.

**Diagnose →** The plan shows `aws_instance.app["web-1"]` destroyed and `["web_1"]` created.

**Fix →**
```hcl
moved {
  from = aws_instance.app["web-1"]
  to   = aws_instance.app["web_1"]
}
```

**Prevent →** Choose stable keys early; never derive keys from mutable display names.

</details>

### Q120. `Error: Provider produced inconsistent final plan` — `.tags_all: new element "Owner" has appeared`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A tag value was unknown at plan time (comes from another resource) and got combined with `default_tags`, so the provider's final plan didn't match the first plan. Known AWS provider edge case.

**Diagnose →** Find tags whose values are `(known after apply)` in the plan.

**Fix →** Re-run the apply; usually the second run succeeds since values are now known. Better: avoid tags computed from resources created in the same run.

**Prevent →** Keep tag values static (variables/locals); report persistent cases to the provider issue tracker.

</details>

### Q121. `Error: Provider produced inconsistent result after apply` — `Root object was present, but now absent.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The provider created the object, but the read right after returned "not found" due to eventual consistency, or something deleted it immediately (an SCP or a cleanup Lambda). It's labelled a provider bug.

**Diagnose →** Check whether the object exists now; check CloudTrail for create and delete events.

**Fix →** If it exists, `terraform import` it (it isn't in state). If not, re-apply. Upgrade the provider if a fix exists.

**Prevent →** Keep providers current, and avoid automation that deletes resources right after creation.

</details>

### Q122. Plans show `~ policy` changes for an S3 bucket policy built with `jsonencode()` because principal values flip between `"123456789012"` and `"arn:aws:iam::123456789012:root"`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AWS stores principals in normalized ARN form; your code uses a bare account ID.

**Diagnose →** Compare `aws s3api get-bucket-policy` output with your code.

**Fix →** Write principals in full ARN form, or build the policy with `data "aws_iam_policy_document"`, which the provider compares semantically.

**Prevent →** Use policy document data sources for every IAM/S3/KMS policy.

</details>

### Q123. `Error: Unsupported block type` — `Blocks of type "ingres" are not expected here. Did you mean "ingress"?` and similar validate errors after a provider upgrade. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typos are easy, but after upgrades, blocks may have become attributes (or were removed), so the old block syntax is no longer valid.

**Diagnose →** `terraform validate` and the provider upgrade guide.

**Fix →** Correct the typo or convert syntax (for example, block `x { }` → attribute `x = { }` where the provider changed the schema).

**Prevent →** Run `terraform validate` and `tflint` in pre-commit.

</details>

### Q124. A plan replaces an ALB target group after changing `port`, and the apply fails with `ResourceInUse: Target group is currently in use by a listener or a rule`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `port`/`protocol`/`name` force replacement. Destroy-before-create can't delete a target group still referenced by a listener.

**Diagnose →** `aws elbv2 describe-target-groups --names app-tg --query 'TargetGroups[0].LoadBalancerArns'`

**Fix →** Use `name_prefix` and `create_before_destroy = true` on the target group so the listener moves to the new one first.

**Prevent →** Make replaceable resources CBD-friendly (`name_prefix`, no hardcoded names).

</details>

### Q125. The plan shows `known after apply` on a module output, and every consumer resource now shows "must be replaced". 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A change inside the module (or module `depends_on`) makes an output unknown; consumers using it in forcing arguments (like `subnet_id`) are planned for replacement because the value *might* change.

**Diagnose →** Trace which output is unknown and why: `terraform show -json tfplan | jq '.resource_changes[] | select(.change.actions|index("delete")) | .address'`.

**Fix →** Remove the cause of the unknown (often a needless `depends_on`, or a replacing resource upstream); apply the upstream change first with a targeted apply if needed.

**Prevent →** Keep module outputs derived from stable attributes, and avoid module-level `depends_on`.

</details>

---

## Authentication and Cloud API Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. `Error: No valid credential sources found` followed by `no EC2 IMDS role found ... context deadline exceeded`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No credentials in env vars, no profile, no SSO session; the provider fell back to the EC2 metadata service, which doesn't exist on your laptop or runner.

**Diagnose →**
```bash
env | grep -E '^AWS_(PROFILE|ACCESS_KEY_ID|REGION|ROLE_ARN|WEB_IDENTITY)'
aws sts get-caller-identity
```

**Fix →** `export AWS_PROFILE=prod-admin && aws sso login`, or in CI configure OIDC credentials before Terraform runs.

**Prevent →** Print `aws sts get-caller-identity` as the first step of every pipeline.

</details>

### Q127. `Error: Cannot assume IAM Role` — `api error AccessDenied: User: arn:aws:iam::111111111111:user/ci is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::222222222222:role/terraform`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target role's **trust policy** doesn't trust the caller, the caller's identity policy lacks `sts:AssumeRole` on that ARN, a required `ExternalId` is missing, or an SCP denies it.

**Diagnose →**
```bash
aws iam get-role --role-name terraform --query 'Role.AssumeRolePolicyDocument'   # run in account 222
aws sts assume-role --role-arn arn:aws:iam::222222222222:role/terraform --role-session-name test
```

**Fix →** Add the caller as `Principal` in the trust policy and `sts:AssumeRole` in the caller's policy; pass `external_id` in `assume_role {}` if the trust policy requires it.

**Prevent →** Manage trust relationships in Terraform in a central IAM stack, tested by a pipeline smoke test.

</details>

### Q128. GitHub Actions: `Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The trust policy's `token.actions.githubusercontent.com:sub` condition doesn't match (different repo, branch, environment or PR context); wrong `aud`; the OIDC provider isn't created in that account.

**Diagnose →** Common `sub` values: `repo:acme/infra:ref:refs/heads/main`, `repo:acme/infra:environment:prod`, `repo:acme/infra:pull_request`. Compare with the trust policy.

**Fix →**
```json
"Condition": {
  "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
  "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:acme/infra:environment:prod" }
}
```

**Prevent →** Separate roles for plan (PRs, read-only) and apply (protected environment only).

</details>

### Q129. GitHub Actions log says `It looks like you might be trying to authenticate with OIDC. Did you mean to set the id-token permission?` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow lacks `permissions: id-token: write`, so no OIDC token can be requested.

**Diagnose →** Check the job/workflow `permissions:` block.

**Fix →**
```yaml
permissions:
  id-token: write
  contents: read
```

**Prevent →** Keep a reusable Terraform workflow with the right permissions instead of copy-pasting YAML.

</details>

### Q130. Applies that run longer than an hour fail with `ExpiredToken: The security token included in the request is expired`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Assumed-role sessions default to 1 hour; long applies (EKS, RDS, CloudFront) outlive them. Role chaining is capped at 1 hour.

**Diagnose →** The error appears about 60 minutes into the apply.

**Fix →** Increase the role's `MaxSessionDuration` and request more (for example `role-duration-seconds: 7200` in the GitHub action, or `duration = "2h"` in the provider's `assume_role`). Avoid chaining roles when long sessions are needed.

**Prevent →** Split slow resources into their own stacks, and set session durations deliberately.

</details>

### Q131. `InvalidClientTokenId: The security token included in the request is invalid.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Deleted/rotated access keys still in env vars or `~/.aws/credentials`; a session token missing (`AWS_SESSION_TOKEN` not set for temporary creds); using credentials in an opt-in region not enabled for the account.

**Diagnose →** `aws configure list` shows which source is in use.

**Fix →** Remove stale `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`, set all three values for temporary creds, or re-login via SSO.

**Prevent →** Prefer SSO and OIDC over long-lived keys.

</details>

### Q132. `failed to refresh cached credentials, refresh cached SSO token failed, unable to refresh SSO token ... InvalidGrantException`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The IAM Identity Center session expired.

**Diagnose →** `aws sts get-caller-identity --profile prod-admin` fails the same way.

**Fix →** `aws sso login --profile prod-admin` (or `--sso-session` name), then rerun.

**Prevent →** Use `sso_session` profiles so refresh tokens work, and set a session length that fits your working day.

</details>

### Q133. `UnauthorizedOperation: You are not authorized to perform this operation. Encoded authorization failure message: ...` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing EC2 permission, an explicit deny (SCP, permission boundary), or a condition such as required tags or allowed regions.

**Diagnose →**
```bash
aws sts decode-authorization-message --encoded-message '<message>' \
  --query DecodedMessage --output text | jq .
```
It shows the action, resource and which policy type denied it.

**Fix →** Add the permission, or adjust code to satisfy the condition (add required tags, use allowed regions/instance types).

**Prevent →** Keep the Terraform role's permissions in code and test new resource types in a sandbox account first.

</details>

### Q134. `AccessDenied ... with an explicit deny in a service control policy`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An Organizations SCP blocks that action/region/resource (for example, deny outside approved regions, deny creating IAM users, deny without tags).

**Diagnose →** The error names the SCP deny; ask the platform team for the SCP, or check `aws organizations list-policies-for-target` with access to the management account.

**Fix →** Change your code to comply (region, tags) or request an approved exception. You can't bypass an SCP from the account.

**Prevent →** Publish SCP guardrails to teams and encode them as policy-as-code checks (OPA/Sentinel/Checkov) that fail in PR.

</details>

### Q135. `SignatureDoesNotMatch` / `Signature expired: ... is now earlier than ...` on a VM-based runner. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Clock skew: the runner's clock is off by more than 5 minutes; or a secret key with copy-paste whitespace.

**Diagnose →** `date -u` vs real time; `timedatectl status`.

**Fix →** Enable NTP (`timedatectl set-ntp true`, chrony), or fix the secret value.

**Prevent →** Use OIDC instead of static keys, and keep runner images with time sync enabled.

</details>

### Q136. Terraform deploys to the wrong account: resources appear in the sandbox account although you targeted prod. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `AWS_PROFILE`/env credentials override what you expected; `assume_role` missing in the provider; static keys from a previous job lingering on a self-hosted runner.

**Diagnose →** `aws sts get-caller-identity`; `terraform state show` an affected resource to see the ARN's account ID.

**Fix →** Destroy the misplaced resources (or `state rm` and clean up manually), then fix credentials and apply to prod.

**Prevent →** Add `allowed_account_ids` to the provider; Terraform refuses to run against any other account:
```hcl
provider "aws" {
  region              = "ap-south-1"
  allowed_account_ids = ["222222222222"]
}
```

</details>

### Q137. Lambda create fails with `InvalidParameterValueException: The role defined for the function cannot be assumed by Lambda.` right after Terraform created the role. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IAM eventual consistency: the role exists but hasn't propagated to Lambda yet. Or the trust policy doesn't list `lambda.amazonaws.com`.

**Diagnose →** Retry after a minute; if it still fails, check the trust policy.

**Fix →** Re-run apply (usually succeeds). The AWS provider retries for a while; if not enough, add a short `time_sleep` depending on the role and policy attachments.

**Prevent →** Make the function depend on the policy attachments too (`depends_on = [aws_iam_role_policy_attachment.logs]`), and keep providers updated for better retry logic.

</details>

### Q138. EC2 launch fails with `Invalid IAM Instance Profile name` though the instance profile was created seconds before. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Same eventual consistency for instance profiles; propagation can take several seconds.

**Diagnose →** `aws iam get-instance-profile --instance-profile-name app` works when you check later.

**Fix →** Re-apply. If it recurs, add a `time_sleep` of 10–15 seconds after the instance profile.

**Prevent →** Create IAM in a separate, earlier stack for things that are reused.

</details>

### Q139. `Error: waiting for EKS Node Group (prod:general) create: unexpected state 'CREATE_FAILED'` after 20+ minutes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Nodes couldn't join: no route to the EKS endpoint/ECR (private subnets without NAT or VPC endpoints), node role missing `AmazonEKSWorkerNodePolicy`/`AmazonEC2ContainerRegistryReadOnly`/CNI policy, insufficient capacity for the instance type, or access entries missing.

**Diagnose →**
```bash
aws eks describe-nodegroup --cluster-name prod --nodegroup-name general \
  --query 'nodegroup.health.issues'
```
Issues like `NodeCreationFailure` or `AsgInstanceLaunchFailures` point to the cause.

**Fix →** Fix networking/IAM/capacity, then re-apply. The failed node group is in state as tainted and will be replaced.

**Prevent →** Use a known-good VPC module with endpoints/NAT, and test node group config in a dev cluster.

</details>

### Q140. Destroying a VPC hangs, then fails with `DependencyViolation: The subnet 'subnet-0abc' has dependencies and cannot be deleted.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Resources not managed by this state still use the subnet: ENIs from Lambda in VPC, load balancers created by the Kubernetes controller, EKS nodes, VPC endpoints from another stack.

**Diagnose →**
```bash
aws ec2 describe-network-interfaces --filters Name=subnet-id,Values=subnet-0abc \
  --query 'NetworkInterfaces[].[NetworkInterfaceId,InterfaceType,Description]' --output table
```

**Fix →** Delete the dependents first (Kubernetes `Service type=LoadBalancer`/Ingress before cluster teardown; wait for Lambda hyperplane ENIs to be released, which can take up to around 40 minutes), then re-run destroy.

**Prevent →** Destroy in reverse stack order (workloads → cluster → network) and tag controller-created resources.

</details>

### Q141. Detaching an internet gateway fails: `Network vpc-0abc has some mapped public address(es). Please unmap those public address(es) before detaching the gateway.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Elastic IPs or public IPs (NAT gateways, instances, load balancers) are still in the VPC, possibly not managed by this stack.

**Diagnose →** `aws ec2 describe-addresses --filters Name=domain,Values=vpc` and check NAT gateways in the VPC.

**Fix →** Delete NAT gateways/EIPs/instances first, then destroy again. Make the IGW explicitly depend on nothing that outlives it.

**Prevent →** Keep the network stack's dependents in other stacks destroyed first.

</details>

### Q142. `BucketNotEmpty: The bucket you tried to delete is not empty` during `terraform destroy`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Objects (or object versions/delete markers in a versioned bucket) remain.

**Diagnose →** `aws s3api list-object-versions --bucket logs --max-items 5`

**Fix →** For disposable buckets set `force_destroy = true`, apply, then destroy. For important buckets, empty them deliberately or keep them.

**Prevent →** `force_destroy = true` only in ephemeral environments.

</details>

### Q143. Applies fail randomly with `ThrottlingException: Rate exceeded` or `RequestLimitExceeded` in large stacks. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Many resources refreshed/created in parallel (default 10), plus other tools hitting the same API.

**Diagnose →** `TF_LOG=DEBUG` shows retries; errors cluster on one service (Route 53, IAM, API Gateway have low limits).

**Fix →** Lower parallelism and raise retries:
```bash
terraform apply -parallelism=4
```
```hcl
provider "aws" {
  max_retries = 25
}
```

**Prevent →** Split huge states; Route 53 and IAM-heavy stacks especially.

</details>

### Q144. `Error: creating EC2 Instance: VcpuLimitExceeded: You have requested more vCPU capacity than your current vCPU limit`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Service quota for the instance family (On-Demand Standard instances counted in vCPUs) is reached; new accounts start with low limits.

**Diagnose →**
```bash
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A
```

**Fix →** Request an increase (Service Quotas console or `request-service-quota-increase`), or use smaller/fewer instances.

**Prevent →** Manage quota requests per account in your account-vending process.

</details>

### Q145. `Unsupported: Your requested instance type (t3.micro) is not supported in your requested Availability Zone (us-east-1e)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Hardcoded AZs or `data "aws_availability_zones"` returning AZs without that instance type.

**Diagnose →**
```bash
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters Name=instance-type,Values=t3.micro --query 'InstanceTypeOfferings[].Location'
```

**Fix →** Use `data "aws_ec2_instance_type_offerings"` to pick supported AZs, or exclude the bad AZ.

**Prevent →** Don't hardcode AZ names; AZ letters differ per account.

</details>

### Q146. `InvalidAMIID.NotFound` when deploying the same code to a new region. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AMI IDs are region-specific; a hardcoded `ami-` from another region.

**Diagnose →** `aws ec2 describe-images --image-ids ami-0abc --region ap-south-1`

**Fix →** Look up the AMI per region with `data "aws_ami"` or an SSM public parameter (`/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`).

**Prevent →** No hardcoded AMI IDs in reusable modules.

</details>

### Q147. An ACM certificate with DNS validation causes the apply to hang on `aws_acm_certificate_validation` until timeout. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Validation CNAMEs weren't created, were created in the wrong hosted zone (private zone, or a zone not delegated from the registrar), or CAA records block Amazon.

**Diagnose →**
```bash
aws acm describe-certificate --certificate-arn <arn> --query 'Certificate.DomainValidationOptions'
dig +short _abc123.api.example.com CNAME
dig +short NS example.com
```

**Fix →** Create the records in the public authoritative zone (`for_each` over `domain_validation_options`), fix delegation/CAA, then re-apply.

**Prevent →** Keep certificate + validation records in one module, and remember CloudFront certificates must be in `us-east-1`.

</details>

### Q148. `ResourceConflictException: The operation cannot be performed at this time. An update is in progress for resource` on Lambda updates. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two updates to the same function (code and configuration) in quick succession; Lambda only allows one at a time while `LastUpdateStatus` is `InProgress`.

**Diagnose →** `aws lambda get-function-configuration --function-name api --query '[State,LastUpdateStatus]'`

**Fix →** Re-run apply. Upgrade the AWS provider (newer versions wait for update completion). Avoid other tools updating the same function concurrently.

**Prevent →** One owner for function updates (Terraform or the app deploy pipeline, not both).

</details>

### Q149. `MalformedPolicyDocument: Has prohibited field Resource` when creating an IAM role. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You put an identity policy (with `Resource`) in `assume_role_policy`. Trust policies must have `Principal` and `Action: sts:AssumeRole`, no `Resource`.

**Diagnose →** Inspect what's passed to `assume_role_policy`.

**Fix →** Separate them: trust policy via `data "aws_iam_policy_document"` with `principals {}`; permissions via `aws_iam_role_policy` or attachments.

**Prevent →** Name documents clearly (`*_trust`, `*_permissions`) in code.

</details>

### Q150. A KMS-encrypted resource fails with `AccessDeniedException ... is not authorized to perform: kms:CreateGrant`, though the role has `kms:*` in its IAM policy. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** KMS needs **both** the key policy and IAM policy to allow access (unless the key policy delegates to IAM with the account root principal). Cross-account keys need the key policy to name your account.

**Diagnose →**
```bash
aws kms get-key-policy --key-id <key-id> --policy-name default --output text | jq .
```
Look for the root principal statement or your role.

**Fix →** Add the Terraform role (or root delegation) to the key policy, including `kms:CreateGrant` with `kms:GrantIsForAWSResource` for services like EBS/RDS.

**Prevent →** Manage key policies in Terraform with the consumers declared explicitly, and test with a sandbox key.

</details>

---

## Modules, Variables, Sensitive Values and Workspaces

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. `Error: Module not installed` — `This module is not yet installed. Run "terraform init" to install all modules required by this configuration.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A new `module` block was added or its `source`/`version` changed since the last init.

**Diagnose →** `cat .terraform/modules/modules.json | jq '.Modules[] | {Key, Source, Version}'`

**Fix →** `terraform init` (or `terraform get` for modules only).

**Prevent →** Every CI job runs `init` first; don't cache `.terraform/modules` across commits.

</details>

### Q152. `Error: Failed to download module` — `git::https://github.com/acme/tf-modules.git?ref=v2.3.0 ... exit status 128` in CI, works locally. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Private repo and the CI runner has no Git credentials (your laptop uses your SSH key or credential helper); the tag doesn't exist; SSH host key not trusted.

**Diagnose →**
```bash
git ls-remote https://github.com/acme/tf-modules.git v2.3.0
GIT_TERMINAL_PROMPT=0 git clone https://github.com/acme/tf-modules.git /tmp/m
```

**Fix →** Give the job a token via Git config:
```bash
git config --global url."https://x-access-token:${GH_TOKEN}@github.com/".insteadOf "https://github.com/"
```
or an SSH deploy key with `known_hosts`. Push the missing tag if needed.

**Prevent →** Publish internal modules to a private registry (HCP Terraform, Artifactory, GitLab) with versioned releases.

</details>

### Q153. `Error: Invalid module source address` for `source = "github.com/acme/modules//vpc?ref=v1.0"` with a `version = "1.0.0"` argument. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `version` only works with registry sources; Git sources pin with `?ref=`. Also typos in the `//subdir` syntax.

**Diagnose →** Read the `source` string carefully: registry format is `namespace/name/provider`.

**Fix →** Remove `version` for Git sources; keep `?ref=v1.0.0`. For registry: `source = "terraform-aws-modules/vpc/aws"` and `version = "~> 5.0"`.

**Prevent →** Use registry sources where possible and pin versions.

</details>

### Q154. After bumping a registry module from 4.x to 5.x, plan shows dozens of resources destroyed and recreated. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Major module versions rename internal resources, switch `count` to `for_each`, or change defaults. Without `moved` blocks in the module (some include them), addresses no longer match.

**Diagnose →** Read the module's upgrade guide/CHANGELOG; list destroyed vs created addresses from `terraform show -json tfplan`.

**Fix →** Revert the bump. Add `moved` blocks in your root mapping old → new addresses, set any changed defaults explicitly, and re-plan until only expected in-place updates remain.

**Prevent →** Pin modules with `~> 4.2` (no surprise majors), and upgrade majors in a dedicated PR per stack.

</details>

### Q155. `Error: No value for required variable` — `The root module input variable "db_password" is not set, and has no default value.` in CI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The local `terraform.tfvars` (gitignored) or your shell's `TF_VAR_db_password` isn't present in CI.

**Diagnose →** `env | grep TF_VAR_` in the job (masked); check which `-var-file` is passed.

**Fix →** Provide it from the CI secret store as `TF_VAR_db_password`, or better, read the secret inside Terraform from Secrets Manager (or use an ephemeral resource/write-only argument where supported).

**Prevent →** Commit non-secret `env/*.tfvars`; keep secrets out of variables entirely where possible.

</details>

### Q156. `Error: Invalid value for variable` — `This was checked by the validation rule at variables.tf:8,3-13.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The value fails a `validation` block (wrong environment name, CIDR format, instance type not in the allowed list).

**Diagnose →** The error prints the rule's `error_message`; check the value from tfvars/env.

**Fix →** Correct the input. If the rule is too strict, update the validation in a reviewed PR.

**Prevent →** Write clear `error_message` text that says what's allowed.

</details>

### Q157. `Warning: Value for undeclared variable` — `The root module does not declare a variable named "regoin" but a value was found in file "terraform.tfvars".` and the region silently defaults. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo in tfvars; Terraform only warns for undeclared vars in files (it errors for `-var` on the CLI). The real variable keeps its default.

**Diagnose →** Read warnings, not just errors; `terraform console` → `var.region`.

**Fix →** Fix the typo. Consider removing defaults for important variables so a missing value fails.

**Prevent →** Fail CI on warnings for undeclared variables (grep the plan output), and use `tflint` rules.

</details>

### Q158. `Error: Output refers to sensitive values` — `Terraform requires that any root module output containing sensitive data be explicitly marked as sensitive`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The output uses a sensitive variable or a sensitive resource attribute (like `aws_db_instance.main.password`, or a module output marked sensitive).

**Diagnose →** Trace the value; check for `sensitive = true` upstream.

**Fix →** Add `sensitive = true` to the output. Read it with `terraform output -raw db_password` only when needed. Or don't output secrets at all.

**Prevent →** Prefer outputting secret ARNs/names (Secrets Manager) instead of secret values.

</details>

### Q159. A security review finds plaintext database passwords in the S3 state file, although variables are marked `sensitive = true`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `sensitive` only hides values in CLI output; state stores all arguments and attributes in plaintext JSON.

**Diagnose →** `terraform state pull | jq '.. | .password? // empty'`

**Fix →** Rotate the exposed credentials. Move to patterns that keep secrets out of state: `manage_master_user_password = true` on RDS (Secrets Manager), ephemeral resources (1.10+) and write-only arguments like `password_wo` (1.11+, provider support required).

**Prevent →** Encrypt state with KMS, restrict `s3:GetObject` on state to the pipeline role, and treat state as secret.

</details>

### Q160. Plan shows `(sensitive value)` for a whole `user_data` or tags block, and reviewers can't see what changes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A single sensitive input (for example a token inside a template) marks the whole derived value as sensitive.

**Diagnose →** Find which input in the template is sensitive.

**Fix →** Don't embed secrets in user data; fetch them at boot from SSM/Secrets Manager using the instance role. Only use `nonsensitive()` if the value truly isn't secret.

**Prevent →** Keep secrets out of templates so plans stay reviewable.

</details>

### Q161. `terraform apply` in a terminal ran against prod because `terraform workspace show` said `prod`, though you selected `dev` earlier. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `TF_WORKSPACE=prod` is exported in the shell (overrides `workspace select`), or the `.terraform/environment` file was changed by another command in the same directory.

**Diagnose →** `echo $TF_WORKSPACE; cat .terraform/environment`

**Fix →** `unset TF_WORKSPACE`; re-select dev. Review what was applied to prod and revert via a proper plan if needed.

**Prevent →** Show workspace and AWS account in your prompt; prefer separate directories/backends per environment for prod; protect prod with separate credentials that a dev shell doesn't have.

</details>

### Q162. `Workspace "staging" doesn't exist. You can create this workspace with the "new" subcommand.` in CI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fresh backend key, wrong backend config, or a typo in the workspace name.

**Diagnose →** `terraform workspace list`

**Fix →** `terraform workspace select -or-create staging` (Terraform 1.4+), or check the backend first if it should already exist.

**Prevent →** Use `select -or-create` only for ephemeral environments; for long-lived ones, a missing workspace should fail loudly (it often means the wrong backend).

</details>

### Q163. A `count = terraform.workspace == "prod" ? 3 : 1` setup created 3 NAT gateways in staging. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CI job for staging ran in a workspace named `prod` (or the comparison uses a name that doesn't match, like `production`). Logic tied to `terraform.workspace` breaks silently.

**Diagnose →** `terraform workspace show` in the job; compare names exactly.

**Fix →** Drive sizing from per-env tfvars (`nat_gateway_count = 1`), not from the workspace name.

**Prevent →** Use `terraform.workspace` only for naming; put behaviour differences in variables.

</details>

### Q164. A module call with `for_each = var.services` fails: `The "for_each" set includes values derived from resource attributes that cannot be determined until apply` only on first apply. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `var.services` (a root variable) is fine, but you merged it with something unknown (for example `merge(var.services, { for k, v in aws_ecr_repository.r : k => ... })`).

**Diagnose →** `terraform console` with the plan state; trace which part is unknown.

**Fix →** Keep the `for_each` keys from static data (`keys(var.services)`) and pass resource values inside each element.

**Prevent →** Separate "which instances" (static keys) from "their config" (can be dynamic).

</details>

### Q165. A child module's `variable "tags" { type = map(string) }` rejects your input: `Invalid value for input variable ... element "cost_center": string required`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A value in the map isn't a string (number, bool, null, nested object).

**Diagnose →** Look at the element named in the error.

**Fix →** Convert: `cost_center = tostring(var.cost_center)`, or remove nulls with `{ for k, v in var.tags : k => v if v != null }`.

**Prevent →** Normalize tags in one locals block before passing to modules.

</details>

### Q166. A module `variable` with `type = object({ name = string, size = number })` fails when callers omit `size`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** All object attributes are required unless marked optional.

**Diagnose →** Error says `attribute "size" is required`.

**Fix →** Use optional attributes with defaults (Terraform 1.3+):
```hcl
variable "disk" {
  type = object({
    name = string
    size = optional(number, 50)
  })
}
```

**Prevent →** Design module inputs with `optional()` so adding fields isn't a breaking change.

</details>

### Q167. Two module instances in the same account fail on second create with `already exists` errors for IAM roles and log groups. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The module hardcodes names (`name = "app-role"`) instead of building them from inputs; each instance tries the same name.

**Diagnose →** `grep -rn 'name\s*=\s*"' modules/app/`

**Fix →** Build names from a `name`/`env` input or use `name_prefix`.

**Prevent →** Module guideline: every globally or account-unique name must derive from inputs.

</details>

### Q168. A module output is `null` in the root although the resource exists. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The output uses `try(aws_x.this[0].id, null)` and the resource has a different index/key; or the resource is conditional (`count = var.create ? 1 : 0`) and `create` is false for this call.

**Diagnose →** `terraform state list | grep module.app`, `terraform console` → `module.app`.

**Fix →** Correct the output expression (`one(aws_x.this[*].id)`), or set the create flag.

**Prevent →** Avoid `try()` that hides real errors in outputs; return explicit nulls only for genuinely optional resources.

</details>

### Q169. `terraform apply` with an ephemeral resource fails: `Ephemeral values are not valid` when you assign the secret to a normal resource argument. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ephemeral values (Terraform 1.10+) can only flow into other ephemeral contexts: provider config, ephemeral variables/outputs, locals, provisioner/connection blocks, or **write-only** arguments (1.11+). Normal arguments would persist them in state, so Terraform blocks it.

**Diagnose →** Check if the target argument is write-only in the provider docs (for example `password_wo` on `aws_db_instance`).

**Fix →**
```hcl
ephemeral "random_password" "db" {
  length = 24
}
resource "aws_db_instance" "main" {
  # ...
  password_wo         = ephemeral.random_password.db.result
  password_wo_version = 1
}
```
Bump `password_wo_version` to rotate.

**Prevent →** Check Terraform and provider versions in CI support ephemeral/write-only features before adopting them.

</details>

### Q170. A module with `count = var.enabled ? 1 : 0` breaks every reference with `Missing resource instance key` after you added the count. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Module instances now have keys (`module.cache[0]`); references and state addresses changed.

**Diagnose →** The plan wants to destroy `module.cache.*` and create `module.cache[0].*`.

**Fix →**
```hcl
moved {
  from = module.cache
  to   = module.cache[0]
}
```
Update references to `module.cache[0].endpoint` or `one(module.cache[*].endpoint)`.

**Prevent →** Add `moved` blocks whenever you add `count`/`for_each` to an existing module or resource.

</details>

### Q171. Terragrunt: `Error: Backend configuration changed` appears in every module after upgrading Terragrunt. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The generated `backend.tf` content changed (new defaults, different key format or added settings), so cached `.terraform` doesn't match. Or `.terragrunt-cache` is stale.

**Diagnose →** Compare generated backend files between versions; check `remote_state` block and `path_relative_to_include()` output.

**Fix →** Clear caches (`find . -type d -name .terragrunt-cache -prune -exec rm -rf {} +`) and run `terragrunt init -reconfigure`. If the **key** changed, migrate state (`-migrate-state`) or fix the key back.

**Prevent →** Pin the Terragrunt version (`terragrunt_version_constraint`) and test upgrades on one stack first.

</details>

### Q172. HCP Terraform workspace plan fails with `No value for required variable` although the variable is set in the workspace. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The variable was created as an **Environment variable** (without the `TF_VAR_` prefix) instead of a **Terraform variable**; or it's set on a variable set not attached to this workspace/project; or the CLI-driven run uses a local `-var-file` that HCP doesn't read.

**Diagnose →** Check the workspace Variables page: category "terraform" vs "env", and attached variable sets.

**Fix →** Create it as a Terraform variable (or env var named `TF_VAR_name`), and attach the variable set to the workspace or project.

**Prevent →** Manage HCP variables and variable sets with the `tfe` provider.

</details>

### Q173. A data source inside a reusable module fails in some accounts with `no matching EC2 VPC found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The module looks up a VPC by tag (`Name = "main"`) that isn't consistent across accounts, or the provider region differs.

**Diagnose →** `aws ec2 describe-vpcs --filters Name=tag:Name,Values=main --region ap-south-1`

**Fix →** Pass `vpc_id` as an input instead of looking it up inside the module.

**Prevent →** Reusable modules should take IDs as inputs; root modules do the lookups.

</details>

### Q174. A `check` block prints `Warning: Check block assertion failed` but the apply succeeds. Is this a problem? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** By design, `check` blocks (1.5+) only warn; they don't block applies. A failed assertion means a health condition isn't met (for example an endpoint returns non-200).

**Diagnose →** Read the assertion's `error_message` and test the target manually (`curl -s -o /dev/null -w '%{http_code}' https://api.example.com/health`).

**Fix →** Investigate the real problem. If it must block, use a `precondition`/`postcondition` on the resource instead.

**Prevent →** Send check failures from CI/HCP drift runs to alerting, so warnings aren't ignored.

</details>

### Q175. `Error: Resource postcondition failed` — `The selected AMI must be for the x86_64 architecture.` after a data source returns a new image. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The postcondition caught a bad lookup result: filters matched an `arm64` image published recently.

**Diagnose →** `terraform console` → `data.aws_ami.app.architecture` and `.name`.

**Fix →** Tighten filters (`architecture = x86_64`, exact name pattern, `owners`) so the lookup is deterministic.

**Prevent →** Keep postconditions on lookups whose results matter; they turn surprises into clear plan-time errors.

</details>

---

## Partial Applies, Version Mismatches and CI Failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. An apply failed halfway: 14 resources created, then `Error: creating RDS DB Instance ... InvalidParameterCombination`. What state are you in? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Terraform doesn't roll back. Resources created before the failure are in state; the failed one is not (or is tainted if it was created but a later wait failed).

**Diagnose →**
```bash
terraform state list | wc -l
terraform plan    # shows only what is still missing or tainted
```

**Fix →** Fix the cause (here, an invalid engine/instance class combination) and re-apply. Terraform continues from where it stopped.

**Prevent →** `terraform validate` and dev-environment applies catch most parameter errors; keep stacks small so partial failures are easy to reason about.

</details>

### Q177. `Error: Saved plan is stale` — `The given plan file can no longer be applied because the state was changed by another operation after the plan was created.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another apply (or a `state` command) ran on the same state between your `plan -out` and `apply`. Common when two PRs plan in parallel and one merges first.

**Diagnose →** Check state history / pipeline runs between the plan and apply timestamps.

**Fix →** Re-run `terraform plan -out=tfplan`, review, then apply the new plan.

**Prevent →** Plan → apply in one pipeline with concurrency control per stack; require branches to be up to date before merge.

</details>

### Q178. Apply job fails: `Failed to read plan from plan file` — `The given plan file was created by Terraform v1.9.5, but this is v1.9.8.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Plan and apply jobs used different Terraform versions (different runner images or `setup-terraform` with `latest`). Plan files are not portable across versions.

**Diagnose →** `terraform version` in both job logs.

**Fix →** Pin one version in both jobs and re-plan.

**Prevent →** `terraform_version: 1.9.8` in `hashicorp/setup-terraform`, plus `required_version` in code, plus a `.terraform-version` file shared with laptops.

</details>

### Q179. `Error: Unsupported Terraform Core version` — `This configuration does not support Terraform version 1.5.7.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The code or a module requires a newer version (`required_version = ">= 1.7"`) than your binary.

**Diagnose →** `grep -rn required_version . .terraform/modules`

**Fix →** Upgrade the binary (`tfenv install 1.9.8 && tfenv use 1.9.8`, or `mise use terraform@1.9.8`). Don't lower the constraint without knowing why it was set.

**Prevent →** Version manager file in every repo; CI uses the same version.

</details>

### Q180. A teammate with Terraform 1.10 applied; now your 1.6 gets `state snapshot was created by Terraform v1.10.0, which is newer than current v1.6.6; upgrade to Terraform v1.10.0 or greater to work with this state`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Terraform records its version in state; older binaries refuse to read state written by newer ones (to avoid losing data they don't understand).

**Diagnose →** `terraform state pull | jq -r .terraform_version` (from a newer binary).

**Fix →** Upgrade your binary. Downgrading state isn't supported; restoring an older state version would lose changes.

**Prevent →** Upgrade Terraform as a team decision: bump `required_version` lower bound and CI image in one PR; nobody applies with a different version.

</details>

### Q181. `Error: Resource instance managed by newer provider version` — `The current state of aws_instance.web was created by a newer provider version than is currently selected.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone applied with a newer AWS provider (unlocked or an older lock file in your branch). State now has a newer schema version.

**Diagnose →** Compare `.terraform.lock.hcl` on your branch with main.

**Fix →** Rebase on main and `terraform init`, or `terraform init -upgrade` to at least that provider version.

**Prevent →** Commit the lock file; CI uses `-lockfile=readonly`; Renovate manages provider bumps.

</details>

### Q182. In GitHub Actions, `terraform plan -detailed-exitcode` makes the job fail whenever there are changes. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `-detailed-exitcode` returns `0` (no changes), `1` (error), `2` (changes present). Most CI systems treat any non-zero exit as failure.

**Diagnose →** The job log shows exit code 2.

**Fix →**
```bash
set +e
terraform plan -detailed-exitcode -out=tfplan -input=false
code=$?
set -e
if [ "$code" -eq 1 ]; then exit 1; fi
echo "has_changes=$([ "$code" -eq 2 ] && echo true || echo false)" >> "$GITHUB_OUTPUT"
```

**Prevent →** Wrap the logic in a shared script used by all pipelines.

</details>

### Q183. `terraform output -raw vpc_id` in a GitHub Actions step returns extra text like `::debug::` lines or the command echo. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `hashicorp/setup-terraform` installs a wrapper script by default that captures stdout/stderr into step outputs and adds noise when you capture output in shell.

**Diagnose →** `which terraform` points to the wrapper script.

**Fix →**
```yaml
- uses: hashicorp/setup-terraform@v3
  with:
    terraform_version: 1.9.8
    terraform_wrapper: false
```

**Prevent →** Disable the wrapper in shared workflows that parse Terraform output.

</details>

### Q184. A CI apply job waits until timeout after printing `Do you want to perform these actions? ... Enter a value:`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `terraform apply` without a saved plan or `-auto-approve` asks for confirmation; there is no TTY in CI.

**Diagnose →** The last log line is the prompt.

**Fix →** Apply the reviewed plan file: `terraform apply -input=false tfplan` (a saved plan doesn't prompt).

**Prevent →** Always `plan -out` then `apply tfplan` in pipelines; set `TF_INPUT=0` globally.

</details>

### Q185. The apply job uses the plan artifact from the plan job but fails with `Error: Inconsistent dependency lock file` or missing provider plugins. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Only `tfplan` was passed between jobs; the apply runner has a fresh checkout with no `.terraform`, or a different commit checked out.

**Diagnose →** Check the apply job runs `terraform init` and checks out the **same commit SHA** as the plan job.

**Fix →** In the apply job: checkout the same SHA, `terraform init -input=false -lockfile=readonly` with the same backend config, then `terraform apply tfplan`.

**Prevent →** Treat plan artifacts as tied to commit + Terraform version + lock file; store them with the SHA in the name.

</details>

### Q186. `terraform fmt -check` fails in CI with exit code 3, but `terraform validate` passes. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Files aren't canonically formatted; `fmt -check` returns non-zero when changes are needed.

**Diagnose →** `terraform fmt -check -recursive -diff`

**Fix →** `terraform fmt -recursive` and commit.

**Prevent →** pre-commit hook (`pre-commit-terraform` `terraform_fmt`) and editor format-on-save.

</details>

### Q187. Atlantis comments `This project is currently locked by an unapplied plan from pull #412` on your PR. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Atlantis holds a project/workspace lock from another PR that ran `atlantis plan` but wasn't applied or closed.

**Diagnose →** Open the Atlantis UI locks page or PR #412.

**Fix →** Apply or close #412, or run `atlantis unlock` on that PR (with the owner's agreement). Then `atlantis plan` again.

**Prevent →** Auto-unlock on PR close, keep PRs small, and use separate projects per stack so locks are narrow.

</details>

### Q188. PR plans pass, but the post-merge apply fails because the plan role is read-only and the apply role lacks a permission that the new resource needs. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Plans only need read/describe permissions; create/update permissions are first exercised at apply. A new resource type (for example `aws_wafv2_web_acl`) isn't in the apply role.

**Diagnose →** The apply error names the denied action (`AccessDenied ... wafv2:CreateWebACL`); decode with `aws sts decode-authorization-message` if encoded.

**Fix →** Add the permissions to the apply role (through its own IAM pipeline) and re-run apply. The partial apply resumes.

**Prevent →** Apply new resource types in a dev account with the same role definition first; review role changes alongside infrastructure PRs.

</details>

### Q189. A CI pipeline applies `dev` successfully, then the `prod` stage applies `dev`'s plan file to prod state. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Artifact name collision (`tfplan` used for both stages) or a missing `-backend-config` switch between stages. Terraform may error (`Saved plan does not match the given state`/stale) or, worse, apply with dev vars if the job re-plans with the wrong var file.

**Diagnose →** Check artifact names, the var file and backend config used in each stage's logs.

**Fix →** Stop the pipeline, inspect prod changes from the apply log, and correct with a proper prod plan.

**Prevent →** Name artifacts `tfplan-<env>-<sha>`, run each environment in its own directory/job with explicit `-backend-config=backends/<env>.hcl` and `-var-file=env/<env>.tfvars`, and print account ID + workspace before apply.

</details>

### Q190. Ctrl-C during an apply printed `Two interrupts received. Exiting immediately. Note that data loss may have occurred.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The first Ctrl-C asks Terraform to stop gracefully after in-flight operations; the second kills it. In-flight resources may exist in the cloud but not in state, and the lock may remain.

**Diagnose →**
```bash
terraform force-unlock <ID>    # only if the lock remains and nothing else runs
terraform plan                 # look for creates of things that already exist
```

**Fix →** Import any resources that exist but are missing from state; untaint or replace half-created ones.

**Prevent →** Press Ctrl-C once and wait. In CI, give jobs a graceful shutdown period.

</details>

### Q191. CI and laptops produce different plans for `archive_file` / `local_file` resources (hashes differ). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Line endings (`core.autocrlf` on Windows), file permissions, or build artifacts present locally but not in CI change the archive content.

**Diagnose →** `git ls-files --eol | grep crlf`; compare the source dirs.

**Fix →** Add `.gitattributes` (`* text=auto eol=lf`), build artifacts in CI only, and exclude local junk with `excludes` in `archive_file`.

**Prevent →** Run applies only from CI so one environment builds artifacts.

</details>

### Q192. `terraform test` fails with `Error: Test assertion failed` in CI, while the same `.tftest.hcl` passes locally. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tests with `command = apply` create real resources and depend on credentials/region/account quotas; local runs use different variables; leftover resources from a previous failed run conflict.

**Diagnose →** `terraform test -verbose` to see the plan/apply inside each run block; check for orphaned test resources by tag.

**Fix →** Pass explicit variables in the test file, use unique names per run (for example a `random_id` in a setup module), and clean orphans.

**Prevent →** Use `command = plan` and mocked providers (`mock_provider`, Terraform 1.7+) for most unit tests; reserve apply tests for a dedicated test account.

</details>

### Q193. OpenTofu migration: after switching CI to `tofu`, `init` shows provider downloads from `registry.opentofu.org` and the lock file diff changes all hashes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** OpenTofu resolves the default `hashicorp/aws` source against its own registry, so lock file entries and hashes differ from Terraform's.

**Diagnose →** `git diff .terraform.lock.hcl` shows `registry.opentofu.org/hashicorp/aws`.

**Fix →** Commit the regenerated lock file once everyone moves to OpenTofu. Don't alternate between `terraform` and `tofu` on the same state; newer features diverge.

**Prevent →** Pick one tool per repo and pin its version; run `tofu providers lock` for all platforms.

</details>

### Q194. A pipeline runs `terraform apply -target=module.app` "to save time". Weeks later, plans show many unrelated pending changes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `-target` applies only part of the graph; everything else drifts from code, and dependent changes pile up. Terraform warns: `Resource targeting is in effect`.

**Diagnose →** Run a full `terraform plan` and review the accumulated diff.

**Fix →** Review and apply the full plan in a maintenance window, in steps if needed.

**Prevent →** Ban `-target` in CI except for documented recovery runbooks; split large stacks instead of targeting.

</details>

### Q195. Pull request plans fail with `Error: Error acquiring the state lock` whenever two PRs touch the same stack. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Plans take the state lock too (to read a consistent snapshot); two concurrent PR plans collide.

**Diagnose →** Pipeline logs show overlapping plan runs on the same key.

**Fix →** Retry with a lock timeout: `terraform plan -lock-timeout=5m`. For PR plans, `-lock=false` is acceptable *only* for read-only speculative plans that are never applied.

**Prevent →** Concurrency groups per stack in CI; HCP Terraform/Atlantis queue runs automatically.

</details>

### Q196. Renovate opened a PR bumping the AWS provider; CI plan shows changes in 30 stacks. How do you triage? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New provider defaults, schema normalization or deprecations; most diffs are harmless, a few might be real behaviour changes.

**Diagnose →**
```bash
for d in stacks/*/; do
  (cd "$d" && terraform plan -input=false -out=tfplan >/dev/null && \
   terraform show -json tfplan | jq -r --arg d "$d" '.resource_changes[] | select(.change.actions != ["no-op"]) | "\($d) \(.address) \(.change.actions|join(","))"')
done | sort | uniq -c
```
Group by resource type and action; any `delete`/`create` is a red flag.

**Fix →** Approve state-only updates; for real changes, set arguments explicitly or pin the old version for affected stacks and report upstream.

**Prevent →** Small, frequent provider bumps and a plan-summary bot that flags destroys.

</details>

### Q197. An apply in CI fails with `Error: Failed to install provider ... checksum list has no SHA-256 hash` or registry `429 Too Many Requests` when 50 jobs start together. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Many runners downloading the same providers simultaneously from the public registry/releases endpoint, hitting rate limits or partial downloads.

**Diagnose →** Failures correlate with burst start times; the same jobs pass on retry.

**Fix →** Retry with backoff; pre-bake providers into the runner image or use a network/filesystem mirror and CI cache keyed on the lock file.

**Prevent →** Internal provider mirror (Artifactory/Nexus or `terraform providers mirror` to S3) for all CI runners.

</details>

### Q198. `terraform destroy` in an ephemeral PR environment leaves resources behind and the cleanup job reports success. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The destroy job ran with a different workspace/backend key (so state was empty: `No changes. No objects need to be destroyed.`), or resources created outside Terraform (controllers, Lambda ENIs, log groups auto-created by services).

**Diagnose →** Check the destroy log for `0 destroyed`; find leftovers by tag:
```bash
aws resourcegroupstaggingapi get-resources --tag-filters Key=pr,Values=412
```

**Fix →** Rerun destroy with the correct workspace/key, then delete untracked leftovers.

**Prevent →** Fail the cleanup job if it destroys 0 resources when state is expected; tag everything with `pr=<number>`; manage auto-created log groups explicitly in Terraform.

</details>

### Q199. A self-hosted runner's apply fails with `Error: Failed to load plugin schemas` / `Plugin did not respond`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The provider process crashed or was OOM-killed (AWS provider can use over 1 GB on big stacks), the binary is for the wrong architecture, or `/tmp` is mounted `noexec`.

**Diagnose →**
```bash
dmesg | grep -i -E 'killed process|oom'
file .terraform/providers/registry.terraform.io/hashicorp/aws/*/linux_amd64/terraform-provider-aws_*
mount | grep /tmp
```

**Fix →** Give the runner more memory, lower `-parallelism`, or split the stack; fix arch/noexec issues and re-init.

**Prevent →** Size runners for your largest stack and monitor memory per job.

</details>

### Q200. Your CI apply worked yesterday, today `terraform init` fails with `Error: Failed to query available provider packages` for a pinned community provider version that "disappeared". 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The provider author yanked or deleted that release, or moved the provider to a new namespace; the registry no longer serves it.

**Diagnose →** Check the provider's registry page and GitHub releases; `terraform providers` shows which modules require it.

**Fix →** Move to the nearest available version (update constraints and `terraform init -upgrade`), or switch to the new namespace with `terraform state replace-provider`:
```bash
terraform state replace-provider registry.terraform.io/oldns/kubectl registry.terraform.io/newns/kubectl
```

**Prevent →** Mirror every provider you depend on internally, prefer well-maintained providers, and keep the mirror as the source of truth for CI.

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
