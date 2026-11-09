# DevSecOps — Cheatsheet

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

> Dense command and config reference for the DevSecOps toolchain: secret scanning, SAST, SCA, IaC and image scanning, SBOMs, signing, admission policy, CIS audits, runtime detection and secrets management. Plus an OWASP Top 10 table and the one-liners interviewers expect.

[⬅ Back to DevSecOps](./README.md) · [🏠 Home](../README.md)

## Table of contents

- [Which tool for which job](#which-tool-for-which-job)
- [gitleaks: secret scanning](#gitleaks-secret-scanning)
- [Semgrep: SAST](#semgrep-sast)
- [Trivy: images, filesystems, IaC, Kubernetes](#trivy-images-filesystems-iac-kubernetes)
- [Checkov: IaC scanning](#checkov-iac-scanning)
- [Syft: SBOMs](#syft-sboms)
- [Grype: vulnerability scanning from images or SBOMs](#grype-vulnerability-scanning-from-images-or-sboms)
- [Cosign: signing and attestations](#cosign-signing-and-attestations)
- [Kyverno: admission policy](#kyverno-admission-policy)
- [Conftest and OPA: Rego policy](#conftest-and-opa-rego-policy)
- [kube-bench: CIS Kubernetes Benchmark](#kube-bench-cis-kubernetes-benchmark)
- [Falco: runtime detection](#falco-runtime-detection)
- [Vault basics](#vault-basics)
- [Kubernetes secrets delivery: Sealed Secrets and ESO](#kubernetes-secrets-delivery-sealed-secrets-and-eso)
- [GitHub Actions hardening snippets](#github-actions-hardening-snippets)
- [OWASP Top 10 quick table](#owasp-top-10-quick-table)
- [Severity, SLAs and gate defaults](#severity-slas-and-gate-defaults)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## Which tool for which job

| Job | Open-source default | Alternatives | Runs at |
|-----|---------------------|--------------|---------|
| Secrets in code and history | gitleaks | TruffleHog, GitHub push protection, detect-secrets | pre-commit, PR |
| SAST | Semgrep | CodeQL, SonarQube, Bandit (Python), gosec (Go) | PR |
| SCA (dependency CVEs) | Trivy fs, Grype | Snyk, Dependabot alerts, OWASP Dependency-Check, OSV-Scanner | PR, build |
| IaC misconfiguration | Checkov | Trivy config (absorbed tfsec), KICS, Terrascan | PR |
| Image vulnerabilities | Trivy image | Grype, Docker Scout, cloud registry scanning | build, registry, scheduled |
| SBOM | Syft | Trivy (`--format cyclonedx`), cdxgen, BuildKit `--sbom` | build |
| Signing and provenance | Cosign (Sigstore) | Notation (Notary v2), GitHub artifact attestations | release |
| Admission policy | Kyverno | OPA Gatekeeper, ValidatingAdmissionPolicy (CEL), Pod Security Admission | deploy |
| Policy-as-code in CI | Conftest (Rego), `kyverno apply` | `opa eval`, Checkov custom policies | PR |
| CIS cluster audit | kube-bench | Trivy k8s, Kubescape | scheduled |
| Runtime detection | Falco | Tetragon, Tracee, commercial CNAPP agents | runtime |
| Secrets store | Vault / OpenBao | AWS Secrets Manager, Azure Key Vault, GCP Secret Manager | runtime |
| DAST | OWASP ZAP | Burp Suite, Nuclei | test, staging |

## gitleaks: secret scanning

| Task | Command |
|------|---------|
| Scan full Git history of the current repo | `gitleaks git -v .` |
| Scan with secrets redacted in output | `gitleaks git --redact -v .` |
| Scan only a commit range | `gitleaks git --log-opts="origin/main..HEAD" .` |
| Scan commits since a date | `gitleaks git --log-opts="--since=2025-01-01" .` |
| Scan a directory or file, no Git | `gitleaks dir ./path` |
| Scan piped input | `cat build.log \| gitleaks stdin` |
| Pre-commit (staged changes only) | `gitleaks git --pre-commit --staged --redact -v` |
| SARIF report for code scanning | `gitleaks git --report-format sarif --report-path gitleaks.sarif .` |
| JSON report | `--report-format json --report-path leaks.json` |
| Custom config | `gitleaks git --config .gitleaks.toml .` |
| Only report new findings against a baseline | `gitleaks git --baseline-path baseline.json .` |
| Exit code on findings (default 1) | `--exit-code 1` |
| Skip big files | `--max-target-megabytes 5` |

Older 8.x releases used `gitleaks detect` (history) and `gitleaks protect --staged` (pre-commit); both still work but are deprecated in favour of `git`, `dir` and `stdin`.

```toml
# .gitleaks.toml
[extend]
useDefault = true                      # keep built-in rules

[[rules]]
id = "acme-internal-token"
description = "ACME internal API token"
regex = '''acme_[a-z0-9]{32}'''
keywords = ["acme_"]                   # cheap prefilter before the regex

[allowlist]
description = "Encrypted or test material"
paths = ['''k8s/sealed-.*\.yaml''', '''testdata/''']
regexes = ['''EXAMPLE''']
```

Suppress one line in code: append `# gitleaks:allow`. Accept a rotated historical finding: add its `Fingerprint` (`commit:file:rule:line`) to `.gitleaksignore`. A pushed secret is fixed by **rotation**, not by deleting the commit.

## Semgrep: SAST

| Task | Command |
|------|---------|
| Scan with a registry ruleset | `semgrep scan --config p/python` |
| Several rulesets, no telemetry | `semgrep scan --config p/python --config p/owasp-top-ten --metrics=off` |
| Default broad ruleset | `semgrep scan --config p/default` |
| Auto-detect rules (needs metrics on) | `semgrep scan --config auto` |
| Fail with exit 1 on findings | `--error` |
| Only ERROR-severity rules | `--severity ERROR` |
| Only findings new since a commit | `semgrep scan --baseline-commit origin/main --config p/python` |
| SARIF and text at the same time | `--sarif-output=semgrep.sarif` |
| JSON output | `--json --output semgrep.json` |
| Exclude paths | `--exclude 'tests/' --exclude '*.min.js'` |
| Skip one rule | `--exclude-rule python.lang.security.audit.eval-detected.eval-detected` |
| Local custom rules | `semgrep scan --config .semgrep/` |
| Validate rule files | `semgrep --validate --config .semgrep/` |
| Run rule tests | `semgrep --test .semgrep/` |
| Preview autofixes | `semgrep scan --config rules.yml --autofix --dryrun` |
| Semgrep AppSec Platform in CI | `SEMGREP_APP_TOKEN=... semgrep ci` |

```yaml
# .semgrep/no-shell-true.yml
rules:
  - id: subprocess-shell-true-with-variable
    languages: [python]
    severity: ERROR
    message: "subprocess called with shell=True and a non-constant command: command injection risk"
    patterns:
      - pattern: subprocess.$FUNC($CMD, ..., shell=True, ...)
      - pattern-not: subprocess.$FUNC("...", ..., shell=True, ...)
```

Suppress one finding: `# nosemgrep: <rule-id>` on the line, with a reason in the same comment. Ignore paths permanently in `.semgrepignore`.

## Trivy: images, filesystems, IaC, Kubernetes

| Task | Command |
|------|---------|
| Scan an image | `trivy image nginx:1.27` |
| The standard CI gate | `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 IMAGE` |
| Scan a local tarball | `trivy image --input image.tar` |
| Only OS or only language packages | `--pkg-types os` / `--pkg-types library` |
| Choose scanners | `--scanners vuln,secret,misconfig,license` |
| Dependencies in a repo (SCA) | `trivy fs --scanners vuln .` |
| Remote Git repo | `trivy repo https://github.com/org/repo` |
| IaC and Dockerfile misconfig | `trivy config .` |
| Scan an existing SBOM | `trivy sbom sbom.cdx.json` |
| Generate an SBOM | `trivy image --format cyclonedx --output sbom.cdx.json IMAGE` |
| Kubernetes cluster summary | `trivy k8s --report summary` |
| One namespace, all findings | `trivy k8s --include-namespaces prod --report all` |
| SARIF for code scanning | `--format sarif --output trivy.sarif` |
| JSON, then convert later | `--format json -o r.json` then `trivy convert --format sarif r.json` |
| Apply VEX statements | `trivy image --vex openvex.json IMAGE` |
| Custom ignore file | `--ignorefile .trivyignore` |
| Use a DB mirror (rate limits) | `--db-repository mirror.gcr.io/aquasec/trivy-db` |
| Pre-download DB for air gap | `trivy image --download-db-only` then `--skip-db-update` |
| Clear caches | `trivy clean --all` |
| Server mode for many clients | `trivy server --listen 0.0.0.0:4954` / `trivy image --server http://trivy:4954 IMAGE` |

```text
# .trivyignore: one ID per line, optional expiry, comment the owner and reason
CVE-2024-12345 exp:2026-12-31   # platform-team: no fix in Debian yet, not reachable
AVD-DS-0002                     # misconfig ID: root user in a build-only stage
```

Read a row as: library, vulnerability ID, severity, status (`fixed`, `affected`, `will_not_fix`), installed version, fixed version. Fixed version is the remediation.

## Checkov: IaC scanning

| Task | Command |
|------|---------|
| Scan a directory | `checkov -d .` |
| Scan one file | `checkov -f main.tf` |
| Limit frameworks | `checkov -d . --framework terraform,kubernetes,dockerfile,github_actions` |
| Terraform plan (resolved values) | `terraform show -json tfplan > plan.json && checkov -f plan.json` |
| Run only specific checks | `checkov -d . --check CKV_AWS_20,CKV_AWS_57` |
| Skip checks globally | `checkov -d . --skip-check CKV_AWS_18` |
| Skip paths | `checkov -d . --skip-path tests/` |
| Report but never fail | `--soft-fail` |
| Compact, failures only | `--compact --quiet` |
| SARIF and CLI output | `-o cli -o sarif --output-file-path results/` |
| List all checks | `checkov --list` |
| Create a baseline, then only fail on new | `checkov -d . --create-baseline` then `checkov -d . --baseline .checkov.baseline` |
| Scan remote Terraform modules too | `--download-external-modules true` |
| Custom checks (Python or YAML) | `--external-checks-dir ./checkov-policies` |
| Config file | `checkov --config-file .checkov.yaml` |

Inline skips, always with a reason:

```hcl
resource "aws_s3_bucket" "logs" {
  # checkov:skip=CKV_AWS_18:This is the access-log target bucket itself
  bucket = "acme-access-logs"
}
```

```yaml
metadata:
  annotations:
    checkov.io/skip1: CKV_K8S_43=Digest pinned at admission by Kyverno
```

```dockerfile
# checkov:skip=CKV_DOCKER_2:Health handled by Kubernetes probes
```

## Syft: SBOMs

| Task | Command |
|------|---------|
| Table of packages in an image | `syft nginx:1.27` |
| SPDX JSON to file | `syft IMAGE -o spdx-json=sbom.spdx.json` |
| CycloneDX JSON to file | `syft IMAGE -o cyclonedx-json=sbom.cdx.json` |
| Two formats in one run | `syft IMAGE -o spdx-json=sbom.spdx.json -o cyclonedx-json=sbom.cdx.json` |
| Source directory | `syft dir:. -o cyclonedx-json` |
| Pull straight from registry, no Docker | `syft registry:ghcr.io/org/app@sha256:...` |
| Image tarball | `syft docker-archive:image.tar` |
| Include packages from all layers | `syft IMAGE --scope all-layers` |
| Convert between formats | `syft convert sbom.spdx.json -o cyclonedx-json=sbom.cdx.json` |
| Exclude paths | `syft dir:. --exclude './node_modules/**/test/**'` |
| List catalogers | `syft cataloger list` |

Query an SBOM with `jq`:

```bash
jq -r '.packages[] | "\(.name) \(.versionInfo)"' sbom.spdx.json | sort | head      # SPDX
jq -r '.components[] | "\(.name) \(.version) \(.purl)"' sbom.cdx.json | head       # CycloneDX
jq -r '.components[] | select(.name=="log4j-core") | .version' sbom.cdx.json      # "are we affected?"
```

## Grype: vulnerability scanning from images or SBOMs

| Task | Command |
|------|---------|
| Scan an image | `grype nginx:1.27` |
| Scan an SBOM (fast, no pull) | `grype sbom:./sbom.spdx.json` |
| Scan a directory | `grype dir:.` |
| Gate: fail at or above a severity | `grype IMAGE --fail-on high` |
| Only findings that have a fix | `grype IMAGE --only-fixed` |
| Output formats | `-o table\|json\|sarif\|cyclonedx-json` |
| Report by CVE instead of GHSA | `grype IMAGE --by-cve` |
| Explain one finding | `grype IMAGE -o json \| grype explain --id CVE-2024-12345` |
| Update or check the DB | `grype db update` / `grype db status` |
| Apply OpenVEX | `grype IMAGE --vex openvex.json` |

```yaml
# .grype.yaml
fail-on-severity: high
only-fixed: true
ignore:
  - vulnerability: CVE-2024-12345
    reason: "not reachable, owner platform-team, review 2026-12-31"
  - fix-state: wont-fix
```

## Cosign: signing and attestations

| Task | Command |
|------|---------|
| Keyless sign (CI with OIDC) | `cosign sign --yes ghcr.io/org/app@sha256:...` |
| Verify keyless, pinned identity | `cosign verify IMAGE@DIGEST --certificate-identity "https://github.com/org/app/.github/workflows/release.yml@refs/heads/main" --certificate-oidc-issuer https://token.actions.githubusercontent.com` |
| Verify with an identity regex | `--certificate-identity-regexp '^https://github.com/org/app/\.github/workflows/.+@refs/heads/main$'` |
| Generate a key pair | `cosign generate-key-pair` |
| Key pair in cloud KMS | `cosign generate-key-pair --kms awskms:///alias/cosign` |
| Sign and verify with a key | `cosign sign --key cosign.key IMAGE@DIGEST` / `cosign verify --key cosign.pub IMAGE@DIGEST` |
| Sign every image in a multi-arch index | `cosign sign --recursive --yes IMAGE@DIGEST` |
| Attest an SBOM | `cosign attest --yes --type spdxjson --predicate sbom.spdx.json IMAGE@DIGEST` |
| Attest provenance | `cosign attest --yes --type slsaprovenance --predicate provenance.json IMAGE@DIGEST` |
| Verify an attestation | `cosign verify-attestation --type spdxjson --certificate-identity ... --certificate-oidc-issuer ... IMAGE@DIGEST` |
| Decode the attested predicate | `... \| jq -r .payload \| base64 -d \| jq .predicate` |
| Show what is attached to an image | `cosign tree IMAGE@DIGEST` |
| Where the signature is stored | `cosign triangulate IMAGE@DIGEST` |
| Sign and verify a file (blob) | `cosign sign-blob --yes --bundle app.bundle app.tar.gz` / `cosign verify-blob --bundle app.bundle --certificate-identity ... --certificate-oidc-issuer ... app.tar.gz` |
| Copy image with signatures | `cosign copy SRC@DIGEST DST` |
| Remove signatures and attestations | `cosign clean IMAGE@DIGEST` |
| Search the transparency log | `rekor-cli search --sha sha256:...` |

Always sign and verify by **digest**. Keyless needs `permissions: id-token: write` in GitHub Actions. Common OIDC issuers: GitHub `https://token.actions.githubusercontent.com`, GitLab `https://gitlab.com`, Google `https://accounts.google.com`.

## Kyverno: admission policy

| Task | Command |
|------|---------|
| Install | `helm repo add kyverno https://kyverno.github.io/kyverno/ && helm install kyverno kyverno/kyverno -n kyverno --create-namespace` |
| List cluster policies and status | `kubectl get cpol` |
| Namespaced policies | `kubectl get pol -A` |
| Policy reports (audit results) | `kubectl get polr -A` / `kubectl get cpolr` |
| Blocked requests | `kubectl get events -A --field-selector reason=PolicyViolation` |
| Exceptions | `kubectl get policyexception -A` |
| Test a policy offline | `kyverno apply policy.yaml --resource pod.yaml` |
| Test against live cluster resources | `kyverno apply policy.yaml --cluster` |
| Run a policy test suite | `kyverno test ./policies` |
| Admission controller logs | `kubectl -n kyverno logs deploy/kyverno-admission-controller` |
| Webhooks Kyverno registered | `kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations \| grep kyverno` |

```yaml
# Require signed images from one registry, verified by keyless identity
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  validationFailureAction: Enforce        # Audit to roll out safely first
  background: false
  webhookTimeoutSeconds: 30
  rules:
    - name: keyless-github
      match:
        any:
          - resources:
              kinds: ["Pod"]
      exclude:
        any:
          - resources:
              namespaces: ["kube-system", "kyverno"]
      verifyImages:
        - imageReferences: ["ghcr.io/org/*"]
          mutateDigest: true
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/org/*/.github/workflows/release.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev
```

```yaml
# Enforce Pod Security Standards "baseline" through Kyverno
    - name: pss-baseline
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        podSecurity:
          level: baseline
          version: latest
```

Pattern anchors: `=(field)` if present must match, `X(field)` must not exist, `^(field)` global anchor, `?*` any non-empty value. Rollout: Audit, read PolicyReports, fix or except, then Enforce. Run three admission replicas with a PDB; the webhook fails closed.

## Conftest and OPA: Rego policy

| Task | Command |
|------|---------|
| Test Kubernetes YAML | `conftest test deploy.yaml -p policy/` |
| Test a Terraform plan | `terraform show -json tfplan > plan.json && conftest test plan.json -p policy/terraform` |
| Parse HCL directly | `conftest test --parser hcl2 main.tf` |
| Test a Dockerfile | `conftest test Dockerfile -p policy/docker` |
| Use a specific package | `conftest test --namespace kubernetes.security deploy.yaml` |
| All packages | `conftest test --all-namespaces deploy.yaml` |
| Unit-test policies | `conftest verify -p policy/` |
| Output for CI | `-o table\|json\|tap\|junit\|github` |
| Share policies as OCI | `conftest push ghcr.io/org/policies:v1 policy/` / `conftest pull oci://ghcr.io/org/policies:v1` |
| OPA unit tests | `opa test policy/ -v` |
| Evaluate a query | `opa eval -i input.json -d policy/ "data.main.deny"` |
| Format and check | `opa fmt -w policy/` / `opa check policy/` |

```rego
# policy/kubernetes/security.rego  (Rego v1 syntax, OPA 1.x and current Conftest)
package main

deny contains msg if {
  input.kind == "Deployment"
  c := input.spec.template.spec.containers[_]
  c.securityContext.privileged == true
  msg := sprintf("container %q must not be privileged", [c.name])
}

deny contains msg if {
  input.kind == "Deployment"
  c := input.spec.template.spec.containers[_]
  not c.resources.limits.memory
  msg := sprintf("container %q has no memory limit", [c.name])
}

# policy/kubernetes/security_test.rego
test_privileged_denied if {
  count(deny) == 1 with input as {"kind": "Deployment", "spec": {"template": {"spec": {"containers": [
    {"name": "a", "securityContext": {"privileged": true}, "resources": {"limits": {"memory": "64Mi"}}}
  ]}}}}
}
```

## kube-bench: CIS Kubernetes Benchmark

| Task | Command |
|------|---------|
| Run as a Job on a self-managed cluster | `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml` |
| Read results | `kubectl logs job/kube-bench` |
| EKS variant | `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-eks.yaml` |
| Run directly on a node | `kube-bench run --targets node` |
| Control plane checks | `kube-bench run --targets master,etcd,controlplane,policies` |
| Only certain checks | `kube-bench run --check 1.2.1,1.2.2` |
| Skip checks | `kube-bench run --skip 1.1.12` |
| JSON for dashboards | `kube-bench run --json --outputfile kb.json` |
| Run on a node via Docker | `docker run --rm --pid=host -v /etc:/etc:ro -v /var:/var:ro aquasec/kube-bench:latest run --targets node` |
| Cleanup | `kubectl delete job kube-bench` |

Results are `PASS`, `FAIL`, `WARN` (manual check) and `INFO`, with a remediation line per failed check. On managed Kubernetes you only own the node and policy sections; use the provider-specific benchmark.

## Falco: runtime detection

| Task | Command |
|------|---------|
| Install with modern eBPF and Falcosidekick | `helm install falco falcosecurity/falco -n falco --create-namespace --set driver.kind=modern_ebpf --set falcosidekick.enabled=true --set falcosidekick.webui.enabled=true` |
| Add the chart repo | `helm repo add falcosecurity https://falcosecurity.github.io/charts` |
| Watch alerts | `kubectl -n falco logs -l app.kubernetes.io/name=falco -f` |
| Trigger "Terminal shell in container" | `kubectl exec -it deploy/app -- sh` |
| Generate many test events | `docker run --rm falcosecurity/event-generator run syscall` |
| Validate a rules file | `falco -V /etc/falco/rules.d/custom.yaml` |
| List supported fields | `falco --list` |
| Update rules from the registry | `falcoctl artifact install falco-rules:latest` |
| Add custom rules via Helm | `--set-file customRules."custom\.yaml"=custom.yaml` or `customRules:` in values |

```yaml
# custom.yaml
- list: app_allowed_processes
  items: [gunicorn, python, python3]

- rule: Unexpected process in secure-app
  desc: A process outside the allow-list started in the secure-app namespace
  condition: >
    spawned_process and container
    and k8s.ns.name = "secure-app"
    and not proc.name in (app_allowed_processes)
  output: >
    Unexpected process (proc=%proc.name cmd=%proc.cmdline user=%user.name
    pod=%k8s.pod.name ns=%k8s.ns.name image=%container.image.repository)
  priority: WARNING
  tags: [container, mitre_execution]
```

Tune with `exceptions:` or by appending conditions to a macro, not by disabling a rule. Priorities run from `EMERGENCY` down to `DEBUG`; route `WARNING` and above to on-call.

## Vault basics

| Task | Command |
|------|---------|
| Dev server (in-memory, unsealed, lab only) | `vault server -dev -dev-root-token-id=root` |
| Point the CLI at it | `export VAULT_ADDR=http://127.0.0.1:8200 VAULT_TOKEN=root` |
| Status and seal state | `vault status` |
| Initialise and unseal production Vault | `vault operator init` / `vault operator unseal` (repeat to threshold) |
| Enable KV v2 | `vault secrets enable -path=secret kv-v2` |
| Write, read, one field | `vault kv put secret/app API_TOKEN=abc` / `vault kv get secret/app` / `vault kv get -field=API_TOKEN secret/app` |
| Versions and rollback | `vault kv metadata get secret/app` / `vault kv rollback -version=1 secret/app` |
| Write a policy | `vault policy write app app-policy.hcl` |
| Token with a policy and TTL | `vault token create -policy=app -ttl=1h` |
| Inspect the current token | `vault token lookup` |
| Kubernetes auth | `vault auth enable kubernetes` then `vault write auth/kubernetes/role/app bound_service_account_names=app bound_service_account_namespaces=prod policies=app ttl=15m` |
| Dynamic database credentials | `vault secrets enable database` then `vault read database/creds/app` |
| Revoke leases | `vault lease revoke -prefix database/creds/app` |
| Enable audit log | `vault audit enable file file_path=/var/log/vault_audit.log` |
| Encrypt with Transit (no key leaves Vault) | `vault write transit/encrypt/app plaintext=$(echo -n hi \| base64)` |

```hcl
# app-policy.hcl  (KV v2 paths include /data/ and /metadata/)
path "secret/data/app/*" {
  capabilities = ["read"]
}
path "secret/metadata/app/*" {
  capabilities = ["list"]
}
```

## Kubernetes secrets delivery: Sealed Secrets and ESO

| Task | Command |
|------|---------|
| Fetch the sealing certificate | `kubeseal --fetch-cert > pub-cert.pem` |
| Seal without writing plaintext | `kubectl create secret generic db -n prod --from-literal=PASSWORD=$(openssl rand -hex 16) --dry-run=client -o yaml \| kubeseal --format yaml > sealed-db.yaml` |
| Seal offline with the cert | `kubeseal --cert pub-cert.pem --format yaml < secret.yaml > sealed.yaml` |
| Back up controller keys | `kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > keys.yaml` |
| Re-encrypt with the newest key | `kubeseal --re-encrypt < sealed.yaml > sealed-new.yaml` |
| ESO status | `kubectl get secretstore,clustersecretstore,externalsecret -A` |
| Force an ESO refresh | `kubectl annotate externalsecret app force-sync=$(date +%s) --overwrite` |
| Read a Secret value safely | `kubectl get secret app -o jsonpath='{.data.KEY}' \| base64 -d` |

Kubernetes Secrets are base64, not encrypted. Turn on encryption at rest (KMS provider), restrict `get`/`list` on Secrets with RBAC, mount as files rather than env vars, and prefer workload identity (IRSA, EKS Pod Identity, GKE Workload Identity, Azure Workload Identity) over static keys.

## GitHub Actions hardening snippets

```yaml
permissions:
  contents: read                  # top-level default; widen per job only

jobs:
  release:
    permissions:
      contents: read
      packages: write
      id-token: write             # OIDC for cloud login and Cosign keyless
    steps:
      - uses: actions/checkout@<full-commit-sha>   # pin third-party actions by SHA
        with:
          persist-credentials: false
      - name: Safe use of untrusted input
        env:
          TITLE: ${{ github.event.pull_request.title }}   # never inline ${{ }} in run:
        run: echo "$TITLE"
```

```jsonc
// AWS IAM trust policy for GitHub OIDC, main branch only
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:org/app:ref:refs/heads/main"
  }
}
```

Avoid `pull_request_target` with a checkout of the PR head, never echo secrets, use environments with required reviewers for production, and keep actions current with Dependabot `package-ecosystem: github-actions`.

## OWASP Top 10 quick table

The 2021 list is what most interviewers still quote. The 2025 edition reshuffles it; the last column shows where each category moved.

| 2021 | Category | Typical example | Pipeline control that helps | 2025 |
|------|----------|-----------------|-----------------------------|------|
| A01 | Broken Access Control | IDOR: `/api/orders/1043` returns another user's order | Threat modeling, authz tests, code review, DAST authenticated scans | A01 |
| A02 | Cryptographic Failures | Passwords hashed with MD5, TLS 1.0 allowed | SAST weak-crypto rules, TLS scanners, IaC checks for encryption at rest | A04 |
| A03 | Injection (SQL, OS command, XSS) | `f"SELECT * FROM users WHERE id={uid}"` | SAST (Semgrep, CodeQL), DAST, parameterised queries, output encoding | A05 |
| A04 | Insecure Design | Password reset with guessable security questions | Threat modeling, abuse cases, security requirements | A06 |
| A05 | Security Misconfiguration | Debug mode on, default creds, public S3 bucket | IaC scanning (Checkov, Trivy config), CIS benchmarks, ZAP baseline headers | A02 |
| A06 | Vulnerable and Outdated Components | Log4Shell in a transitive dependency | SCA (Trivy, Grype), SBOMs, Dependabot or Renovate | A03, widened to Software Supply Chain Failures |
| A07 | Identification and Authentication Failures | No rate limit on login, session not invalidated | DAST, MFA, rate limiting, secure session config | A07 (Authentication Failures) |
| A08 | Software and Data Integrity Failures | Unsigned updates, CI pulling unpinned actions, insecure deserialisation | Cosign signing, SLSA provenance, admission verification, SHA-pinned actions | A08 |
| A09 | Security Logging and Monitoring Failures | No audit trail for admin actions, alerts nobody reads | Centralised logs, Falco, CloudTrail, alert routing to on-call | A09 (Logging and Alerting Failures) |
| A10 | Server-Side Request Forgery | Image-fetch endpoint reaches `169.254.169.254` | URL allow-lists, IMDSv2, egress network policies, DAST | Merged into A01 |
| — | Mishandling of Exceptional Conditions (new in 2025) | Fail-open error handling, stack traces leaked to users | Code review, SAST, chaos and negative testing | A10 |

## Severity, SLAs and gate defaults

| Item | Common default | Why |
|------|----------------|-----|
| Image and SCA gate | Fail on fixable HIGH and CRITICAL | Actionable; unfixed CVEs go to the dashboard |
| Secret scanning gate | Fail on any finding | A secret has no "low" severity |
| SAST gate | Fail on new ERROR findings (baseline) | Avoid blocking on legacy debt |
| IaC gate | Fail on any failed check, inline skips with reasons | Skips stay visible in review |
| DAST gate | Fail on chosen rules only, warn on the rest | Baseline scans are noisy otherwise |
| Remediation SLA | Critical 7 days, High 30, Medium 90, Low best effort | Shorter for internet-facing or KEV-listed CVEs |
| Exceptions | Owner, reason, expiry date, reviewed | Exceptions without expiry become permanent |
| CVSS bands (v3.1) | Low 0.1–3.9, Medium 4.0–6.9, High 7.0–8.9, Critical 9.0–10.0 | Severity is not risk; add EPSS, KEV and reachability |
| Fulcio certificate lifetime | About 10 minutes | Why keyless needs the Rekor timestamp |
| GitHub OIDC `sub` for main | `repo:ORG/REPO:ref:refs/heads/main` | Pin it in every cloud trust policy |

## Top 15 interview one-liners

1. Shift-left means running each control at the earliest point it can work, not running everything in the IDE.
2. SAST reads code without running it, DAST attacks the running app from outside, SCA matches dependency versions against CVE databases.
3. A leaked secret is fixed by rotating it; deleting the commit only hides it from people who have not cloned yet.
4. Gate on fixable HIGH and CRITICAL with `--ignore-unfixed`; a gate nobody can satisfy gets disabled within a week.
5. An SBOM is an ingredients list in SPDX or CycloneDX; it turns "are we affected by this CVE?" into a query.
6. Sign the digest, never the tag, because tags are mutable.
7. Keyless signing swaps a long-lived private key for a short-lived Fulcio certificate bound to an OIDC identity, logged in Rekor.
8. A keyless verification that does not pin the exact identity and issuer accepts anyone's signature.
9. Admission control is the last gate before code runs, so the cluster re-verifies what CI claims.
10. Roll out policies in Audit, read the reports, fix or except, then Enforce.
11. Kubernetes Secrets are base64-encoded, not encrypted; use encryption at rest, tight RBAC and an external secret store.
12. Prefer workload identity and short-lived credentials (OIDC, IRSA, Pod Identity, Vault dynamic secrets) over static keys everywhere.
13. Falco watches syscalls at runtime to catch what static scanning cannot, such as a shell spawned inside a production container.
14. SLSA levels describe how far you can trust the build: provenance exists, it is signed by a hosted builder, the build is isolated from tampering.
15. Vulnerability management is SLAs, owners, expiring exceptions and trend metrics, not a spreadsheet of 4,000 CVEs.

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
