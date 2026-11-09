# DevSecOps — Assignments

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

> 25 hands-on assignments in three levels: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Every one produces something you can show: a repo, a policy, a report or a runbook.

[⬅ Back to DevSecOps](./README.md) · [🏠 Home](../README.md)

## How to use these

Do them in a throwaway GitHub repo and a local kind cluster unless the assignment says otherwise. Never point scanners that attack (ZAP active scans, fuzzers) at anything you do not own. For each assignment, keep a short `NOTES.md` with the commands you ran, what the tool reported and what you decided; that file is what turns a lab into an interview story. Hints are collapsed so you can try first.

Most assignments build on the [hands-on lab](./hands-on-lab.md). The [cheatsheet](./cheatsheet.md) has the commands.

## Table of contents

- 🟢 [Beginner](#-beginner) — Assignments 1–10
- 🟡 [Intermediate](#-intermediate) — Assignments 11–20
- 🔴 [Advanced](#-advanced) — Assignments 21–25

---

## 🟢 Beginner

### Assignment 1. Threat model a three-tier app with STRIDE 🟢

**Objective:** Practise the design-time activity that decides which controls the pipeline needs.

**Tasks**
1. Pick a simple system: a React frontend, a Flask API, a PostgreSQL database, an S3 bucket for uploads, deployed on Kubernetes.
2. Draw a data-flow diagram as a Mermaid flowchart. Mark every trust boundary (internet to ingress, pod to database, pod to cloud API, CI to cluster).
3. For each boundary, list at least one threat per STRIDE category that applies.
4. For each threat, write a mitigation and the tool or control that implements it (for example "Tampering of image in registry → Cosign signing + Kyverno verifyImages").
5. Rank the top five threats by likelihood times impact.

**Expected outcome:** A `threat-model.md` with a diagram, a table of at least 15 threats with mitigations, and a ranked top five.

<details>
<summary>Hints</summary>

STRIDE: Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege. Ask four questions: what are we building, what can go wrong, what are we going to do about it, did we do a good job. Repudiation threats are usually answered by audit logs (CloudTrail, Kubernetes audit log). OWASP Threat Dragon is a free tool if you prefer drawing.

</details>

**Bonus:** Map each mitigation to the SDLC phase where it runs (plan, code, build, test, release, deploy, operate).

### Assignment 2. Stop secrets before they leave the laptop 🟢

**Objective:** Catch secrets at the cheapest possible point and scan existing history.

**Tasks**
1. Clone any repo you own with real history, and run `gitleaks git -v` over it. Record what it finds.
2. Install gitleaks as a pre-commit hook with the `pre-commit` framework.
3. Try to commit a fake AWS secret access key pattern and a fake GitHub token. Confirm both are blocked.
4. Write a `.gitleaks.toml` that extends the default rules and adds one custom rule for an internal token format, for example `acme_[a-z0-9]{32}`.
5. Turn on GitHub secret scanning and push protection in the repo settings, then explain in two sentences how it differs from gitleaks.

**Expected outcome:** The hook blocks commits with secrets, the custom rule fires on a test string, and you have a written finding list for the existing repo.

<details>
<summary>Hints</summary>

`.pre-commit-config.yaml` with `repo: https://github.com/gitleaks/gitleaks`, `hooks: [{id: gitleaks}]`, then `pre-commit install`. A custom rule needs `id`, `regex` and ideally `keywords` so gitleaks only runs the regex on lines containing the keyword. `[extend] useDefault = true` keeps the built-in rules. Push protection is server-side and validates some token types with the provider; gitleaks is client-side and pattern-based.

</details>

**Bonus:** Run TruffleHog with `--only-verified` on the same repo and compare: which tool had more false positives, and why?

### Assignment 3. Read a Trivy image report like an owner 🟢

**Objective:** Understand what an image scanner reports and how base image choice changes it.

**Tasks**
1. Scan `python:3.13`, `python:3.13-slim`, `python:3.13-alpine` and `gcr.io/distroless/python3-debian12` with `trivy image --severity HIGH,CRITICAL`.
2. Record total findings and fixable findings (`--ignore-unfixed`) for each in a table.
3. For one finding, explain every column: library, vulnerability, severity, status, installed version, fixed version.
4. Explain why the full image has so many more findings than slim, and what you lose with distroless.

**Expected outcome:** A comparison table and a recommendation for which base you would use for a Python API, with one paragraph of reasoning.

<details>
<summary>Hints</summary>

`trivy image --format json` plus `jq '[.Results[].Vulnerabilities[]?] | length'` counts findings. `--ignore-unfixed` hides CVEs with no fixed version. Distroless has no shell or package manager, which helps security but makes `kubectl exec` debugging harder; ephemeral debug containers (`kubectl debug`) solve that.

</details>

**Bonus:** Repeat the scan with Grype and explain any difference in counts (different databases, different matching for OS vs language packages).

### Assignment 4. Harden a Dockerfile until the scanners are quiet 🟢

**Objective:** Apply container image best practices and prove them with tools.

**Tasks**
1. Start from a naive Dockerfile: single stage, `FROM node:latest`, `COPY . .`, `npm install`, running as root, no health check.
2. Run `checkov -f Dockerfile` and `trivy config .`, and record every failed check.
3. Rewrite it: multi-stage, pinned base by digest, `npm ci --omit=dev`, non-root `USER` with a UID of 10000 or above, `HEALTHCHECK`, a `.dockerignore`.
4. Re-run both scanners until they pass, and compare image size before and after with `docker images`.

**Expected outcome:** Both scanners pass, the image runs as non-root (`docker run --rm img id`), and the final image is noticeably smaller.

<details>
<summary>Hints</summary>

Get a digest with `docker buildx imagetools inspect node:22-slim`. Common checks: `CKV_DOCKER_2` (HEALTHCHECK), `CKV_DOCKER_3` (USER), `CKV_DOCKER_7` (latest tag). The official Node images already contain a `node` user with UID 1000; Checkov's Kubernetes check for high UIDs is why many teams create their own user at 10001.

</details>

**Bonus:** Add `hadolint` and fix its findings too. Explain one rule where hadolint and Checkov disagree.

### Assignment 5. Semgrep: fix, suppress and write your own rule 🟢

**Objective:** Use SAST without drowning in noise.

**Tasks**
1. Clone a deliberately vulnerable app such as OWASP's PyGoat or a Flask app you write with an SQL injection and a `yaml.load` call.
2. Run `semgrep scan --config p/python --config p/owasp-top-ten --metrics=off`.
3. Fix two true positives in code.
4. Suppress one finding you judge to be a false positive with an inline `nosemgrep` comment that includes the rule ID and a reason.
5. Write a custom rule in `.semgrep/no-print-secrets.yml` that flags `print()` or `logging` calls whose argument contains a variable named like `password`, `token` or `secret`. Add a test file and run `semgrep --test`.

**Expected outcome:** Fewer findings, one documented suppression, and a custom rule with passing tests.

<details>
<summary>Hints</summary>

A rule needs `id`, `message`, `severity`, `languages` and a `pattern`, `patterns` or `pattern-either`. Use metavariables with `metavariable-regex` to match variable names, for example `$X` where `regex: (?i).*(password|token|secret).*`. Test files use `# ruleid: <id>` and `# ok: <id>` comments above the lines.

</details>

**Bonus:** Run Semgrep with `--baseline-commit origin/main` so only new findings appear, and explain why that matters for adoption.

### Assignment 6. Checkov on Terraform: fix five, skip one 🟢

**Objective:** Find and fix infrastructure misconfigurations before `terraform apply`.

**Tasks**
1. Write Terraform for an S3 bucket, a security group allowing `0.0.0.0/0` on port 22, and an RDS instance with `publicly_accessible = true` and no encryption.
2. Run `checkov -d .` and note the check IDs.
3. Fix at least five findings in code (public access block, encryption, versioning, restricted SSH, private RDS).
4. Skip one remaining check inline with `#checkov:skip=<ID>:<reason>` and justify it.
5. Run `trivy config .` on the same code and compare findings.

**Expected outcome:** A clean run except for one documented skip, and a short comparison of Checkov and Trivy config results.

<details>
<summary>Hints</summary>

Useful IDs: `CKV_AWS_24` (SSH from 0.0.0.0/0), `CKV_AWS_17` (RDS publicly accessible), `CKV_AWS_16` (RDS encryption), `CKV2_AWS_6` (S3 public access block), `CKV_AWS_21` (S3 versioning). The skip comment goes inside the resource block. No AWS account is needed: Checkov scans code, not cloud state.

</details>

**Bonus:** Generate a plan JSON (`terraform show -json tfplan > plan.json`) and scan that instead. Find a misconfiguration that only appears in the plan because it comes from a variable default.

### Assignment 7. SBOMs in two formats 🟢

**Objective:** Generate, inspect and reuse SBOMs.

**Tasks**
1. Build any image and generate SBOMs with Syft in SPDX JSON and CycloneDX JSON.
2. Use `jq` to answer: how many packages, which version of OpenSSL, which licenses appear.
3. Scan the SBOM file (not the image) with Grype.
4. Generate an SBOM for a source directory (`syft dir:.`) and compare it with the image SBOM. Explain why they differ.

**Expected outcome:** Two SBOM files, a `NOTES.md` with the answers, and a Grype report produced from the SBOM alone.

<details>
<summary>Hints</summary>

`syft IMAGE -o spdx-json=sbom.spdx.json -o cyclonedx-json=sbom.cdx.json`. SPDX lists packages in `.packages[]` with `versionInfo`; CycloneDX uses `.components[]` with `version`. `grype sbom:./sbom.spdx.json`. A source SBOM sees only declared dependencies; an image SBOM also sees OS packages and anything installed at build time.

</details>

**Bonus:** Run `trivy image --format cyclonedx` and diff the package list with Syft's.

### Assignment 8. OWASP Top 10 on OWASP Juice Shop 🟢

**Objective:** Turn the OWASP list from vocabulary into things you have seen.

**Tasks**
1. Run Juice Shop locally: `docker run -d -p 3000:3000 bkimminich/juice-shop`.
2. Find and document one example for at least six OWASP Top 10 categories (for example a SQL injection login bypass, an exposed admin page, a weak password reset).
3. For each, write which pipeline control could have caught it (SAST, SCA, DAST, IaC, code review, threat modeling) and which could not.
4. Run a ZAP baseline scan against it and match ZAP alerts to your categories.

**Expected outcome:** A table of six or more categories with an exploited example, the detecting control, and the matching ZAP alert where there is one.

<details>
<summary>Hints</summary>

Juice Shop has a built-in score board that lists challenges by category. Broken access control and insecure design are mostly invisible to scanners, which is a good interview point. ZAP baseline: `docker run --rm --network host ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t http://localhost:3000`.

</details>

**Bonus:** Write which categories changed between the 2017 and 2021 editions and why SSRF got its own entry.

### Assignment 9. Automated dependency updates 🟢

**Objective:** Make patching continuous instead of a quarterly project.

**Tasks**
1. In a repo with a Dockerfile, a `requirements.txt` and a workflow, add `.github/dependabot.yml` covering `pip`, `docker` and `github-actions`, weekly.
2. Enable Dependabot alerts and security updates in the repo settings.
3. Pin one old vulnerable dependency on purpose and confirm Dependabot opens a security pull request.
4. Group minor and patch updates into one weekly pull request per ecosystem.
5. Make sure your secure pipeline runs on Dependabot pull requests and explain what permissions those runs have.

**Expected outcome:** Dependabot opens grouped update pull requests and a security pull request, and the pipeline runs on them.

<details>
<summary>Hints</summary>

`version: 2`, then `updates:` entries with `package-ecosystem`, `directory`, `schedule.interval`, and `groups:` with `update-types: ["minor", "patch"]`. Workflows triggered by Dependabot get a read-only `GITHUB_TOKEN` and only Dependabot secrets, which is why signing and pushing should only happen on `main`.

</details>

**Bonus:** Configure the same thing with Renovate and compare: digest pinning of base images, automerge rules, dashboard issue.

### Assignment 10. Pod Security Admission in a namespace 🟢

**Objective:** Use the built-in Kubernetes control before reaching for a policy engine.

**Tasks**
1. In kind, create namespaces `psa-baseline` and `psa-restricted` with `enforce` labels for each level.
2. Try to run: a plain `nginx` pod, a pod with `privileged: true`, a pod with `hostNetwork: true`, and the hardened deployment from the lab.
3. Record which pods each namespace accepts and the exact error text for the rejections.
4. Make a plain `nginx` pod pass `restricted` by changing only its spec and image.
5. Preview what would break if you enforced `baseline` on every namespace, without changing anything.

**Expected outcome:** A table of pod versus namespace with accept or reject, and a `restricted`-compliant nginx manifest.

<details>
<summary>Hints</summary>

Labels: `pod-security.kubernetes.io/enforce=restricted` plus optional `warn` and `audit`. Restricted needs `runAsNonRoot`, `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]` and `seccompProfile.type: RuntimeDefault`. `nginxinc/nginx-unprivileged` listens on 8080 and runs as non-root. Preview: `kubectl label --dry-run=server --overwrite ns --all pod-security.kubernetes.io/enforce=baseline`.

</details>

**Bonus:** Explain what PSA cannot do that Kyverno or Gatekeeper can (image verification, registry allow-lists, mutation, custom fields).

---

## 🟡 Intermediate

### Assignment 11. Build and break the full secure pipeline 🟡

**Objective:** Complete the [hands-on lab](./hands-on-lab.md) and make its results visible to developers.

**Tasks**
1. Build the ten-stage pipeline from the lab in your own repo.
2. Break each of stages 1 to 5 and stage 10 on purpose in separate pull requests, and save a screenshot or log excerpt of each failure.
3. Make sure gitleaks, Semgrep and Trivy all upload SARIF so findings show under **Security**, **Code scanning**.
4. Add a job summary (`$GITHUB_STEP_SUMMARY`) that prints a one-line count of findings per scanner.
5. Make the required status checks enforce on `main`.

**Expected outcome:** Six failing pull requests with clear failure reasons, a green `main`, and findings visible in code scanning.

<details>
<summary>Hints</summary>

Trivy SARIF: `format: sarif`, `output: trivy.sarif`, and set `exit-code: "0"` in a second, reporting-only invocation so the upload still happens when the gate fails. Upload with `github/codeql-action/upload-sarif@v3` and a distinct `category` per tool. Branch protection: `gh api -X PUT repos/OWNER/REPO/branches/main/protection`.

</details>

**Bonus:** Measure total pipeline time and get the security stages under three minutes with caching and parallel jobs.

### Assignment 12. Key-based versus keyless signing 🟡

**Objective:** Understand what keyless signing removes and what key-based signing still offers.

**Tasks**
1. Generate a key pair with `cosign generate-key-pair`, sign an image by digest with the private key, and verify with the public key.
2. Sign the same image keylessly from a GitHub Actions workflow and verify with identity and issuer.
3. Store the private key in a cloud KMS (AWS KMS, GCP KMS or Azure Key Vault) and sign with the KMS reference.
4. Write a table comparing the three: where the trust root is, what an attacker must steal, how rotation works, what is published publicly.

**Expected outcome:** Three verified signatures on one image (`cosign tree` shows them) and the comparison table.

<details>
<summary>Hints</summary>

`cosign sign --key cosign.key IMAGE@DIGEST`, `cosign verify --key cosign.pub IMAGE@DIGEST`. KMS: `cosign generate-key-pair --kms awskms:///alias/cosign-lab`, then `cosign sign --key awskms:///alias/cosign-lab`. Keyless publishes the signer identity to the public Rekor log, which matters for private repositories.

</details>

**Bonus:** Configure Kyverno to accept either the keyless identity or the KMS key, using two attestor entries with `count: 1`.

### Assignment 13. A tested Kyverno policy set 🟡

**Objective:** Treat cluster policy like application code: written, tested, rolled out safely.

**Tasks**
1. Write Kyverno policies that: require `runAsNonRoot`, require CPU and memory limits, disallow the `latest` tag, restrict images to `ghcr.io/<you>/` and `registry.k8s.io/`, and add a default `seccompProfile` via mutation.
2. Write a `kyverno-test.yaml` with passing and failing resources for each policy and run `kyverno test .`.
3. Deploy them in `Audit` mode, read the PolicyReports, fix workloads, then switch to `Enforce`.
4. Add a `PolicyException` for one workload with a written reason.
5. Run the tests in CI on every pull request that touches `policies/`.

**Expected outcome:** Five policies, a passing test suite, and evidence of the audit-to-enforce rollout in PolicyReports.

<details>
<summary>Hints</summary>

The test file uses `apiVersion: cli.kyverno.io/v1alpha1`, `kind: Test`, with `policies`, `resources` and `results` listing `policy`, `rule`, `resources` and `result: pass|fail|skip`. `kubectl get policyreport -A -o wide` shows pass and fail counts. PolicyExceptions must be enabled in the Kyverno Helm values and are namespaced.

</details>

**Bonus:** Write the same five rules as Kubernetes `ValidatingAdmissionPolicy` objects with CEL and compare readability.

### Assignment 14. Rego for Terraform plans with Conftest 🟡

**Objective:** Enforce organisation-specific rules that off-the-shelf scanners do not know about.

**Tasks**
1. Write Rego policies that deny: any `aws_s3_bucket` without a `CostCenter` tag, any `aws_instance` type outside an allowed list, and any `aws_security_group_rule` opening port 22 to `0.0.0.0/0`.
2. Run them against `terraform show -json` output with `conftest test`.
3. Write unit tests for each rule and run `conftest verify`.
4. Add a CI job that fails the pull request when Conftest denies the plan.

**Expected outcome:** Three Rego rules with unit tests, failing on a bad plan and passing on a good one.

<details>
<summary>Hints</summary>

Iterate `input.resource_changes[]`, filter on `.type` and on `.change.actions` containing `create` or `update`, then inspect `.change.after`. Current Conftest uses OPA 1.x, so write `deny contains msg if { ... }` (Rego v1 syntax). Tests are rules named `test_...` that call `deny` with a mocked `input` using `with input as {...}`.

</details>

**Bonus:** Return structured results with a severity field, and only fail CI on `high` while printing the rest as warnings.

### Assignment 15. External Secrets with no static credentials 🟡

**Objective:** Sync secrets into Kubernetes using workload identity, not a token in a Secret.

**Tasks**
1. Run Vault (dev mode is fine) and enable the Kubernetes auth method pointing at your kind cluster.
2. Create a Vault policy that can read only `secret/data/secure-app` and a role bound to the `secure-app` service account in the `secure-app` namespace.
3. Configure an ESO `SecretStore` that uses Kubernetes auth with that role, and an `ExternalSecret` that creates `app-secret`.
4. Prove that another service account in the same namespace cannot read the secret through the store.
5. Rotate the value in Vault and measure how long until the pod sees it.

**Expected outcome:** A synced Secret with no Vault token stored anywhere in the cluster, and a failed read from the wrong service account.

<details>
<summary>Hints</summary>

`vault auth enable kubernetes`, then `vault write auth/kubernetes/config kubernetes_host=https://kubernetes.default.svc` from inside the cluster. Role: `vault write auth/kubernetes/role/secure-app bound_service_account_names=secure-app bound_service_account_namespaces=secure-app policies=secure-app ttl=15m`. ESO store: `auth.kubernetes.mountPath: kubernetes`, `role: secure-app`, `serviceAccountRef.name: secure-app`. A mounted Secret volume updates in place; environment variables do not.

</details>

**Bonus:** Do the same against AWS Secrets Manager on EKS with IRSA or EKS Pod Identity.

### Assignment 16. Remove static cloud keys from CI 🟡

**Objective:** Replace long-lived CI credentials with GitHub OIDC federation.

**Tasks**
1. Create an IAM OIDC identity provider for `token.actions.githubusercontent.com` in a sandbox AWS account.
2. Create a role whose trust policy only allows your repo's `main` branch, with `sts.amazonaws.com` as audience.
3. Use `aws-actions/configure-aws-credentials` with `role-to-assume` in a workflow and run `aws sts get-caller-identity`.
4. Show that a run from another branch or a pull request is denied.
5. Delete any `AWS_ACCESS_KEY_ID` secrets from the repo.

**Expected outcome:** CI authenticates to AWS with a one-hour session, only from `main`, with no stored keys.

<details>
<summary>Hints</summary>

The job needs `permissions: id-token: write`. Trust policy conditions: `StringEquals` on `token.actions.githubusercontent.com:aud` = `sts.amazonaws.com` and on `token.actions.githubusercontent.com:sub` = `repo:OWNER/REPO:ref:refs/heads/main`. Using `StringLike` with `repo:OWNER/*` is the classic overly broad mistake. Pull request runs have a `sub` of `repo:OWNER/REPO:pull_request`.

</details>

**Bonus:** Use GitHub environments and make the `sub` claim `repo:OWNER/REPO:environment:prod`, with a required reviewer on that environment.

### Assignment 17. Deeper DAST: API scan and authenticated scan 🟡

**Objective:** Go beyond the baseline scan safely.

**Tasks**
1. Run a ZAP API scan against Juice Shop or your own app using an OpenAPI definition.
2. Configure an authenticated scan: a context with a login, a logged-in indicator and one test user.
3. Compare findings from baseline, API and authenticated scans in a table.
4. Write a `rules.tsv` that fails the build on high-risk alerts and ignores two documented false positives.
5. Schedule the full scan nightly against a disposable environment, not on every pull request, and explain why.

**Expected outcome:** Three reports, a comparison table, and a tuned rules file with justifications.

<details>
<summary>Hints</summary>

`zap-api-scan.py -t <openapi-url> -f openapi -r api.html`. Authenticated scans are easiest with the ZAP Automation Framework (`zap.sh -cmd -autorun plan.yaml`) using an `authentication` block in the context. Use `on: schedule` with a cron in GitHub Actions for the nightly run.

</details>

**Bonus:** Feed the ZAP JSON report into DefectDojo and deduplicate findings across runs.

### Assignment 18. Runtime detection with Falco 🟡

**Objective:** Detect what slipped past every static control.

**Tasks**
1. Install Falco with the modern eBPF driver and Falcosidekick in kind.
2. Trigger at least three default rules: a shell in a container, reading `/etc/shadow`, and writing below `/etc`.
3. Write a custom rule that alerts when any process other than `gunicorn` or `python` starts in the `secure-app` namespace.
4. Route alerts to a Slack webhook or the Falcosidekick UI.
5. Tune one noisy rule with an exception rather than disabling it.

**Expected outcome:** Alerts for the triggered events, a working custom rule, and one documented tuning change.

<details>
<summary>Hints</summary>

`helm install falco falcosecurity/falco -n falco --create-namespace --set driver.kind=modern_ebpf --set falcosidekick.enabled=true --set falcosidekick.webui.enabled=true`. Custom rules go in `customRules` in the Helm values. Use `k8s.ns.name` and `proc.name` in the condition, and `spawned_process` and `container` macros. Watch with `kubectl -n falco logs -l app.kubernetes.io/name=falco -f`.

</details>

**Bonus:** Compare Falco with Tetragon: detection versus in-kernel enforcement, and when you would kill a process instead of alerting.

### Assignment 19. Vulnerability triage with exceptions and VEX 🟡

**Objective:** Run vulnerability management like an owner, not a ticket generator.

**Tasks**
1. Scan three images you use and export results to JSON.
2. Triage every CRITICAL and HIGH: fix now, fix with SLA, or not affected.
3. For one not-affected finding, write an OpenVEX document with a justification and show that `trivy image --vex` hides it.
4. For one finding you must accept temporarily, add it to `.trivyignore` with an `exp:` date and an owner comment.
5. Produce a one-page summary: counts by severity before and after, decisions, and the next review date.

**Expected outcome:** A VEX file, an expiring ignore entry, and a triage summary a manager could read.

<details>
<summary>Hints</summary>

`vexctl create --product="pkg:oci/..." --vuln="CVE-..." --status="not_affected" --justification="vulnerable_code_not_in_execute_path"`. Trivy: `trivy image --vex openvex.json IMAGE`. Typical SLAs: critical 7 days, high 30, medium 90. Reachability (is the vulnerable function called?) is the strongest argument for not affected.

</details>

**Bonus:** Make the pipeline fail when a `.trivyignore` entry has expired, so exceptions cannot be forgotten.

### Assignment 20. Dynamic database credentials with Vault 🟡

**Objective:** Replace a shared database password with per-pod, short-lived credentials.

**Tasks**
1. Run PostgreSQL and Vault in kind.
2. Enable the database secrets engine and configure a role that creates a user with `SELECT` on one schema and a one-hour TTL.
3. Read credentials twice and show that each read creates a different user in PostgreSQL.
4. Revoke a lease and show the user is dropped.
5. Deliver the credentials to a pod with the Vault Agent Injector or the Vault CSI provider.

**Expected outcome:** Pods connect with unique, expiring database users, and revocation is demonstrated.

<details>
<summary>Hints</summary>

`vault secrets enable database`, `vault write database/config/app plugin_name=postgresql-database-plugin connection_url="postgresql://{{username}}:{{password}}@postgres:5432/app?sslmode=disable" allowed_roles=app username=vault password=...`, then a role with `creation_statements` and `default_ttl=1h`. `vault read database/creds/app`, `vault lease revoke <lease_id>`. In PostgreSQL, `\du` lists the generated users.

</details>

**Bonus:** Rotate the root credential with `vault write -f database/rotate-root/app` and explain why after this nobody, including you, knows it.

---

## 🔴 Advanced

### Assignment 21. SLSA provenance verified at admission 🔴

**Objective:** Prove not just who built an image, but how, and enforce it in the cluster.

**Tasks**
1. Add build provenance to the lab pipeline with `actions/attest-build-provenance` (or the SLSA GitHub generator for containers).
2. Verify it from your laptop with `gh attestation verify` (or `slsa-verifier verify-image`), pinning the source repo and branch.
3. Extend the Kyverno `verify-signed-images` policy with an `attestations` block that requires provenance and checks a condition on it, such as the source repository or the builder ID.
4. Show that an image signed by your workflow but missing provenance is rejected.
5. Write half a page mapping your setup to SLSA Build levels, stating honestly which level you reach and what is missing for the next.

**Expected outcome:** Provenance attached to every `main` image, verified locally and at admission, with a written SLSA assessment.

<details>
<summary>Hints</summary>

`attest-build-provenance` needs `id-token: write` and `attestations: write`, with `subject-name`, `subject-digest` and `push-to-registry: true`. Verify: `gh attestation verify oci://IMAGE@DIGEST --repo OWNER/REPO`. Check the predicate type your tool emits (`https://slsa.dev/provenance/v1` for GitHub attestations) and use it as the `type` in the Kyverno attestation. Build L3 requires that user-defined build steps cannot influence the provenance, which is why the generator runs in a separate reusable workflow.

</details>

**Bonus:** Also require the SPDX SBOM attestation and deny images whose SBOM contains a package on a deny list.

### Assignment 22. Admission policy tests inside CI 🔴

**Objective:** Catch broken policies and non-compliant manifests in the pull request, not at deploy time.

**Tasks**
1. Add a job to the lab workflow that creates a kind cluster with `helm/kind-action`, installs Kyverno, and applies the policies from the repo.
2. Deploy the freshly signed image and assert the rollout succeeds.
3. Apply `tests/policy/*.yaml` and assert each is rejected, failing the job if any is admitted.
4. Run `kyverno apply` against every manifest in `k8s/` before the cluster step, as a fast pre-check.
5. Keep the whole job under five minutes.

**Expected outcome:** A CI job that fails if a policy stops blocking what it should, or starts blocking a legitimate workload.

<details>
<summary>Hints</summary>

`if kubectl apply -f tests/policy/privileged-pod.yaml; then echo "should have been denied"; exit 1; fi`. Use `helm install ... --wait` and `kubectl wait --for=condition=Ready clusterpolicy --all`. This job can only verify signatures for images produced on `main`, so on pull requests either test the privileged rule only or sign PR images with a separate identity that a PR-only policy trusts.

</details>

**Bonus:** Run the same assertions against Gatekeeper with equivalent Rego constraints and compare the developer experience.

### Assignment 23. Design a vulnerability management program 🔴

**Objective:** Show you can own security outcomes across many teams, not just run tools.

**Tasks**
1. Write a policy document: severity definitions, remediation SLAs, who owns a finding, how exceptions are requested, approved and expired.
2. Choose a central place for findings (DefectDojo, GitHub code scanning, a SIEM) and send results from at least three scanners to it.
3. Define five metrics, such as mean time to remediate by severity, open criticals per service, percentage of images signed, SLA breach rate and exception count, and build a dashboard for them.
4. Describe how you would roll the program out to 30 teams without a revolt: audit first, phased gates, office hours.
5. Present it as a one-page summary for an engineering director.

**Expected outcome:** A policy document, a working central findings view with real data, and a metrics dashboard.

<details>
<summary>Hints</summary>

DefectDojo has importers for Trivy, Semgrep, gitleaks, ZAP and Checkov, and deduplicates across runs. Count only what an engineer can act on in the headline metrics. Tie SLAs to exposure: an internet-facing service with a critical gets a shorter clock than an internal batch job.

</details>

**Bonus:** Add EPSS scores or the CISA KEV catalogue to prioritisation and show how it reorders your top ten.

### Assignment 24. CIS audit and cluster posture 🔴

**Objective:** Audit a cluster against the CIS Kubernetes Benchmark and fix what matters.

**Tasks**
1. Run kube-bench as a Job in kind and save the results.
2. Pick five failed checks and remediate them on the control plane or node (kind lets you edit static pod manifests with `docker exec`).
3. Run `trivy k8s --report summary` and fix the top three workload misconfigurations.
4. Enable Kubernetes audit logging in kind with a policy that records `Metadata` for Secrets and `RequestResponse` for RBAC changes.
5. Write which checks do not apply to managed Kubernetes (EKS, GKE, AKS) and which benchmark variant you would use there.

**Expected outcome:** Before-and-after kube-bench and Trivy results, an audit log with real entries, and a short note on managed clusters.

<details>
<summary>Hints</summary>

`kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml`, then `kubectl logs job/kube-bench`. kind cluster config supports `kubeadmConfigPatches` and `extraMounts` to pass an audit policy file to the API server. On managed services the control plane is not yours to audit; there are CIS benchmarks for EKS, GKE and AKS, and kube-bench has targets for them.

</details>

**Bonus:** Schedule kube-bench as a CronJob and alert when the failed-check count goes up.

### Assignment 25. Incident drill: compromised dependency and leaked key 🔴

**Objective:** Practise the response, not just the prevention.

**Tasks**
1. Scenario A: a popular package you depend on published a malicious version yesterday. Use your SBOM store to find every image containing it within ten minutes, and write the exact commands.
2. Block that version from being deployed with a Kyverno policy or a Grype or Trivy gate, and roll back affected services to the last good digest.
3. Scenario B: a cloud access key was pushed to a public repo. Write a timed runbook: revoke, check CloudTrail for use, rotate dependants, scope the blast radius, communicate.
4. Run scenario B for real in a sandbox: create a key with no permissions, push it to a test repo, and measure detection time from GitHub secret scanning, gitleaks and your provider.
5. Write a blameless post-incident review with a timeline, root cause and three preventive actions.

**Expected outcome:** Two runbooks, measured detection times, and a post-incident review.

<details>
<summary>Hints</summary>

With SBOMs stored as attestations you can loop over images with `cosign verify-attestation --type spdxjson ... | jq` and search for the package and version. CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIA...`. AWS often attaches a quarantine policy to keys found in public GitHub repos within minutes; note when that happened.

</details>

**Bonus:** Turn scenario A into a scheduled job that checks every stored SBOM against newly published advisories and opens an issue automatically.

---

[⬅ Back to DevSecOps](./README.md) · [🏠 Home](../README.md)

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
