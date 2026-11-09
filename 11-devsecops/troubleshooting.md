# DevSecOps — Troubleshooting Drills

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

> Real DevSecOps failures with exact error text: scanners, signing, admission policies, secrets, cloud identity, runtime and incident response · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to DevSecOps](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [SAST, SCA and secrets scanning in CI](#sast-sca-and-secrets-scanning-in-ci) — Q1–Q25
- [Container image and IaC scanning](#container-image-and-iac-scanning) — Q26–Q50
- [Supply chain: signing, SBOM and dependencies](#supply-chain-signing-sbom-and-dependencies) — Q51–Q75
- [Kubernetes admission control and Pod Security](#kubernetes-admission-control-and-pod-security) — Q76–Q100
- [Secrets management: Vault, External Secrets and SOPS](#secrets-management-vault-external-secrets-and-sops) — Q101–Q125
- [Cloud identity: IRSA, Workload Identity and CI OIDC](#cloud-identity-irsa-workload-identity-and-ci-oidc) — Q126–Q150
- [DAST, runtime detection and compliance scans](#dast-runtime-detection-and-compliance-scans) — Q151–Q175
- [Certificates, TLS and security incident response](#certificates-tls-and-security-incident-response) — Q176–Q200

---

## SAST, SCA and secrets scanning in CI

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. gitleaks fails the PR on `tests/fixtures/fake_rsa.pem` and `testdata/aws_creds.json`, which only contain dummy values 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fixtures match generic rules (`private-key`, `aws-access-token`) by pattern; gitleaks has no idea they are fake.

**Diagnose →**
```bash
gitleaks detect --source . --report-format json --report-path gl.json -v
jq -r '.[] | "\(.RuleID) \(.File):\(.StartLine) \(.Fingerprint)"' gl.json
```

**Fix →** Add a path allowlist in `.gitleaks.toml` for fixture folders only, or put specific fingerprints in `.gitleaksignore`:
```toml
[extend]
useDefault = true
[allowlist]
description = "test fixtures"
paths = ['''^tests/fixtures/''', '''^testdata/''']
```

**Prevent →** Generate fake keys at test runtime instead of committing them, and review every allowlist change in CODEOWNERS so nobody allowlists `src/`.

</details>

### Q2. gitleaks passes locally but the CI job reports 40 old leaks nobody touched in this PR 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CI runs `gitleaks detect` over the full history while local runs use `protect --staged` (or `git` mode on a shallow clone). Old commits contain historical secrets.

**Diagnose →**
```bash
git rev-list --count HEAD           # shallow vs full clone
gitleaks detect --log-opts="origin/main..HEAD" -v   # only PR commits
```

**Fix →** Scan only the PR range in CI (`--log-opts="$BASE_SHA..$HEAD_SHA"`), and handle the historical findings separately: rotate them, then baseline with `--baseline-path gl-baseline.json`.

**Prevent →** Run a full-history scan on a schedule, not on every PR, and track legacy findings as tickets until each is rotated.

</details>

### Q3. `fatal: ambiguous argument 'origin/main..HEAD': unknown revision` in the secrets-scan step 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `actions/checkout` defaults to `fetch-depth: 1`, so the base branch ref does not exist in the runner.

**Diagnose →**
```bash
git branch -a
git log --oneline | wc -l
```

**Fix →**
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
```
Or fetch just the base: `git fetch origin main --depth=50`.

**Prevent →** Any tool that diffs against a base (gitleaks, Semgrep diff-aware, SonarQube new-code) needs history; set `fetch-depth: 0` in the shared workflow template.

</details>

### Q4. Semgrep flags `subprocess.run(cmd, shell=True)` in a script where `cmd` is a hard-coded constant 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The rule is a pattern match on `shell=True`, not a taint rule, so it cannot tell a constant from user input.

**Diagnose →**
```bash
semgrep --config p/python --json path/to/script.py | jq '.results[] | {check_id, line: .start.line}'
```

**Fix →** If genuinely safe, suppress inline with a reason:
```python
subprocess.run(CMD, shell=True)  # nosemgrep: python.lang.security.audit.subprocess-shell-true.subprocess-shell-true -- constant, no input
```
Better: drop `shell=True` and pass a list.

**Prevent →** Prefer taint-mode rules for injection classes, and require a justification string on every `nosemgrep` (enforce with a review rule).

</details>

### Q5. Semgrep in CI reports findings in `node_modules/`, `vendor/` and minified `dist/*.js` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No `.semgrepignore`, or a custom one that replaced the defaults. Build output is committed or produced before the scan.

**Diagnose →**
```bash
semgrep --config auto --json . | jq -r '.results[].path' | cut -d/ -f1 | sort | uniq -c
```

**Fix →** Add `.semgrepignore`:
```
node_modules/
vendor/
dist/
*.min.js
```
Run the SAST job before the build step.

**Prevent →** Third-party code belongs to SCA, not SAST. Keep ignore files in the repo template.

</details>

### Q6. Semgrep job takes 35 minutes and gets killed with `exit code 137` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** OOM on the runner from `--config auto` across a huge monorepo, very large generated files, or too many jobs in parallel.

**Diagnose →**
```bash
semgrep --config p/default --time --json . > out.json
jq '.time.targets | sort_by(-.run_time) | .[:10] | .[] | {path, run_time}' out.json
```

**Fix →** Exclude generated code, cap file size and memory, and scan only changed files on PRs:
```bash
semgrep ci --max-target-bytes 1000000 --max-memory 4000 -j 2
```
`semgrep ci` is diff-aware on PRs when `SEMGREP_BASELINE_REF` / PR context is set.

**Prevent →** Full scans nightly, diff scans on PRs, and a curated ruleset instead of everything.

</details>

### Q7. SonarQube Quality Gate fails on "Security Hotspots Reviewed is less than 100%" but there are no vulnerabilities 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Hotspots are not bugs; they are code that needs a human review (crypto use, regexes, cookies). The default "Sonar way" gate requires all new hotspots reviewed.

**Diagnose →** Project → Security Hotspots → filter "New code", status "To review". Or via API:
```bash
curl -su "$SONAR_TOKEN:" "$SONAR_HOST/api/hotspots/search?projectKey=myapp&status=TO_REVIEW&inNewCodePeriod=true"
```

**Fix →** Review each and mark **Safe**, **Fixed** or **Acknowledged** with a comment. Do not lower the gate to make it green.

**Prevent →** Make hotspot review part of PR review; a security champion triages hotspots weekly.

</details>

### Q8. SonarQube marks a real SQL injection as "fixed" after a refactor, but the code still concatenates input 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A false negative: the data flow now crosses a wrapper method, reflection, or a framework the analyzer does not model, so taint tracking lost the source.

**Diagnose →** Trace source to sink manually, then confirm with a second engine:
```bash
semgrep --config p/sql-injection src/
codeql database create db --language=java && codeql database analyze db java-security-extended.qls --format=sarif-latest -o out.sarif
```

**Fix →** Fix the code with parameterised queries. Add a custom rule (Semgrep taint mode or a CodeQL source/sink model) for the wrapper so it is detected next time.

**Prevent →** Do not treat "0 findings" as proof. Pair SAST with code review on security-sensitive modules and DAST on the running endpoint.

</details>

### Q9. `sonar-scanner` fails with `Not authorized. Please check the user token in the property 'sonar.token' or the environment variable 'SONAR_TOKEN'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Token expired or revoked, secret not exposed to the job (forks do not get secrets), wrong host URL, or the token lacks "Execute Analysis" on that project.

**Diagnose →**
```bash
curl -su "$SONAR_TOKEN:" "$SONAR_HOST_URL/api/authentication/validate"
# {"valid":false} means a bad token
```

**Fix →** Generate a project analysis token, store it as `SONAR_TOKEN` in CI secrets, and confirm the job actually receives it (`[ -n "$SONAR_TOKEN" ] || exit 1`).

**Prevent →** Use project-scoped tokens with an expiry and a calendar reminder; alert when analyses stop arriving.

</details>

### Q10. SonarQube shows 0% coverage and the "Coverage on New Code" condition fails, although tests pass 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The coverage report is not produced, is produced after the scanner runs, or the path property is wrong (`sonar.javascript.lcov.reportPaths`, `sonar.coverage.jacoco.xmlReportPaths`, `sonar.python.coverage.reportPaths`). Paths inside the report do not match repo paths (containers, `/app/src`).

**Diagnose →**
```bash
ls -la coverage/lcov.info target/site/jacoco/jacoco.xml
grep -m3 '^SF:' coverage/lcov.info
```
Check scanner logs for `Could not resolve N file paths`.

**Fix →** Run tests with coverage first, point the property at the file, and rewrite paths if tests ran in a container (`sed -i 's#/app/#./#' coverage/lcov.info`).

**Prevent →** Keep scanner config in `sonar-project.properties` in the repo and fail the job if the report file is missing.

</details>

### Q11. OWASP Dependency-Check reports CVEs for `jquery` in a Java project that does not use jQuery 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Dependency-Check identifies components by CPE matching on file names and metadata, which produces false positives (a webjar, a bundled docs folder, or a similarly named artifact).

**Diagnose →** Open the HTML report, check the "Evidence" for the match and which file it came from.

**Fix →** Suppress the specific false match with a scoped suppression file:
```xml
<suppress>
  <notes>Bundled in javadoc, not shipped at runtime</notes>
  <packageUrl regex="true">^pkg:javascript/jquery@.*$</packageUrl>
  <cve>CVE-2020-11023</cve>
</suppress>
```
Run with `--suppression suppression.xml`.

**Prevent →** Prefer purl-based SCA (Trivy, Grype, OSV-Scanner) over CPE matching where possible; review suppressions every quarter.

</details>

### Q12. Dependency-Check job fails with `NVD Returned Status Code: 403` or hangs for an hour updating the database 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The NVD API 2.0 heavily rate-limits callers without an API key, and every CI run downloads the full database from scratch.

**Diagnose →** Look for `NVD API request failures` or `Rate limit exceeded` in logs; check the job runs with an empty data directory each time.

**Fix →** Request an NVD API key and pass it, and cache the data directory:
```bash
dependency-check.sh --nvdApiKey "$NVD_API_KEY" --data /cache/odc-data --scan .
```

**Prevent →** Run a scheduled job that updates a shared data directory or mirror and have PR jobs run with `--noupdate`.

</details>

### Q13. `npm audit` fails the build with a critical in a devDependency used only for tests 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `npm audit` checks all dependencies by default, including dev tooling that never ships to production.

**Diagnose →**
```bash
npm audit --omit=dev --audit-level=high
npm ls <package>
```

**Fix →** Gate on production deps: `npm audit --omit=dev --audit-level=high`. Still update the dev package when a fix exists (`npm update <pkg>` or an `overrides` entry).

**Prevent →** Separate gates: block on prod-reachable highs, report dev-only findings as warnings. Let Renovate keep dev tooling current.

</details>

### Q14. `npm audit fix --force` "fixed" the CVE and broke the app at runtime 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--force` installs semver-major upgrades of direct dependencies to reach a patched transitive version, bringing breaking API changes.

**Diagnose →**
```bash
git diff package.json package-lock.json
npm ls vulnerable-pkg
```

**Fix →** Revert, then target only the vulnerable transitive package with `overrides`:
```json
"overrides": { "semver": "^7.5.4" }
```
Run the test suite and `npm audit` again.

**Prevent →** Never run `audit fix --force` in CI. Use Renovate/Dependabot PRs that go through tests.

</details>

### Q15. pip-audit or Trivy reports a CVE in a Python package, but `pip install -U` says it is already the latest 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A pinned upper bound in another dependency holds it back, or the scanner reads a stale lockfile (`requirements.txt`, `poetry.lock`) rather than the installed env.

**Diagnose →**
```bash
pip-audit -r requirements.txt
pipdeptree -r -p cryptography
pip index versions cryptography
```

**Fix →** Bump the constraining parent, regenerate the lock (`pip-compile --upgrade-package cryptography` or `poetry update cryptography`), and rescan.

**Prevent →** Scan the lockfile that is actually deployed, and keep locks regenerated by automation.

</details>

### Q16. GitHub code scanning shows the same CodeQL alert twice, once per workflow 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two workflows upload SARIF for the same tool without distinct categories, or a renamed workflow left old analyses alive.

**Diagnose →**
```bash
gh api repos/:owner/:repo/code-scanning/analyses --jq '.[] | {tool: .tool.name, category, ref}' | head
```

**Fix →** Give each upload a unique `category`:
```yaml
- uses: github/codeql-action/analyze@v3
  with:
    category: "/language:javascript"
```
Delete stale analyses with `gh api -X DELETE .../analyses/<id>`.

**Prevent →** One code-scanning workflow per tool, maintained centrally.

</details>

### Q17. `upload-sarif` fails with `Resource not accessible by integration` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `GITHUB_TOKEN` lacks `security-events: write`, often because the org default is read-only, or the run is from a fork.

**Diagnose →** Check the workflow's `permissions:` block and the run's "Set up job" log, which lists granted permissions.

**Fix →**
```yaml
permissions:
  contents: read
  security-events: write
```
For fork PRs, upload from a `workflow_run` workflow instead of granting write to untrusted code.

**Prevent →** Declare least-privilege `permissions:` per job in every workflow.

</details>

### Q18. SARIF upload rejected: `SARIF file is too large` / `too many results` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The tool output exceeds GitHub limits (compressed size and result count per run), usually because vendored code or every severity is included.

**Diagnose →**
```bash
jq '[.runs[].results | length] | add' results.sarif
ls -lh results.sarif
```

**Fix →** Exclude vendored paths, filter severity (`trivy fs --severity HIGH,CRITICAL --format sarif`), or split into multiple categories.

**Prevent →** Treat SARIF as a curated signal, not a dump; keep low-severity output as build artifacts.

</details>

### Q19. A developer pushed `--no-verify` and the pre-commit secret hook never ran; the secret is now on a feature branch 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Client-side hooks are optional by design and can always be skipped.

**Diagnose →**
```bash
gitleaks detect --log-opts="origin/main..feature/x" -v
```

**Fix →** Treat it as leaked: revoke and rotate the credential first, then remove it from the branch. Do not just delete the line.

**Prevent →** Back the hook with server-side controls: GitHub push protection / GitLab secret push protection plus a required CI secret-scan check. Hooks are convenience, not enforcement.

</details>

### Q20. GitHub push protection blocks a push: `GH013: Repository rule violations found ... Push cannot contain secrets` for a revoked key 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Push protection pattern-matches known provider token formats; it does not know the key is revoked.

**Diagnose →** The push output links to the blocked secret and commit SHA.

**Fix →** Remove the secret from the commits (`git rebase -i`, amend, push again). Only use the "allow secret" link if it is truly a test value and justify it; bypasses are audited.

**Prevent →** Keep tokens out of code entirely; load from env or a secrets manager even in tests.

</details>

### Q21. trufflehog reports a "verified" Slack webhook in a 2019 commit 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** trufflehog tested the credential against the provider and it still works, meaning a real live secret sits in history.

**Diagnose →**
```bash
trufflehog git file://. --only-verified --json | jq '{detector: .DetectorName, commit: .SourceMetadata.Data.Git.commit, file: .SourceMetadata.Data.Git.file}'
```

**Fix →** Revoke and reissue the webhook immediately, update consumers, then decide on history rewriting separately.

**Prevent →** Prioritise verified findings over unverified ones; run `--only-verified` org-wide on a schedule.

</details>

### Q22. Semgrep custom rule matches nothing although you can see the vulnerable code 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong `languages`, a pattern that does not parse as code in that language, missing `...` for arguments, or paths excluded by `.semgrepignore`.

**Diagnose →**
```bash
semgrep --validate --config rules/my-rule.yml
semgrep --config rules/my-rule.yml --debug src/target.py 2>&1 | grep -i -E "skipp|parse"
semgrep --test rules/
```

**Fix →** Use the Playground-style iteration: start with the exact snippet, generalise with metavariables, e.g. `pattern: requests.get($URL, ..., verify=False, ...)`.

**Prevent →** Ship every custom rule with a test file containing `# ruleid:` and `# ok:` annotations and run `semgrep --test` in CI.

</details>

### Q23. SCA gate blocks a release for a critical CVE that exists in a function your code never calls 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SCA is version-based; it flags the package, not reachable code paths.

**Diagnose →** Check reachability: `govulncheck ./...` for Go reports only called vulnerable symbols; for other ecosystems use a reachability-capable SCA or inspect call sites with grep/IDE.

**Fix →** If unreachable and no fix exists, record a time-boxed exception with evidence (VEX `not_affected` with justification `vulnerable_code_not_in_execute_path`). Upgrade as soon as a fix ships.

**Prevent →** Gate on reachable + exploitable (EPSS/KEV) criticals, and expire exceptions automatically.

</details>

### Q24. The secret scanner flags high-entropy strings in `package-lock.json` integrity hashes 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Generic entropy rules match `sha512-...` integrity values and base64 blobs.

**Diagnose →** Check the `RuleID` (e.g. `generic-api-key`) and the file path in the report.

**Fix →** Allowlist lockfiles by path or a regex for `integrity` values in `.gitleaks.toml`:
```toml
[allowlist]
paths = ['''(^|/)package-lock\.json$''', '''(^|/)yarn\.lock$''']
```

**Prevent →** Tune generic rules once in a shared config and extend it per repo instead of each team inventing their own.

</details>

### Q25. Security scans pass on PRs from forks without reporting anything, but fail on main 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fork PRs get no secrets, so SonarQube/Snyk tokens are empty and the step is skipped or `continue-on-error` swallows the auth failure.

**Diagnose →** Look for empty tokens or `401` in fork run logs; check `if:` conditions and `continue-on-error: true`.

**Fix →** Run token-free scanners (Semgrep OSS, gitleaks, Trivy fs) on `pull_request`, and token-based ones in a `workflow_run` job that checks out the PR code read-only. Never use `pull_request_target` with a checkout of PR code and secrets.

**Prevent →** Make a missing token a hard failure on trusted branches, and audit for silent `continue-on-error`.

</details>

## Container image and IaC scanning

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. Trivy fails the build on 14 HIGH/CRITICAL CVEs in the base image, all with "Fixed Version" empty 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The distro has not shipped patches (status `affected` / `will_not_fix`), so no upgrade will clear them and the gate blocks forever.

**Diagnose →**
```bash
trivy image --severity HIGH,CRITICAL --format json myapp:1.4 \
  | jq -r '.Results[].Vulnerabilities[]? | select(.FixedVersion==null or .FixedVersion=="") | "\(.VulnerabilityID) \(.PkgName) \(.Status)"'
```

**Fix →** Gate only on fixable issues and track the rest:
```bash
trivy image --ignore-unfixed --severity HIGH,CRITICAL --exit-code 1 myapp:1.4
```
Consider a slimmer base (distroless, Chainguard/Wolfi, `-slim`) to drop unused packages entirely.

**Prevent →** Policy: block on fixable HIGH/CRITICAL, report unfixable, rebuild images weekly so patches land automatically.

</details>

### Q27. `FATAL ... failed to download vulnerability DB: ... TOOMANYREQUESTS: retry-after` from Trivy in CI 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every job pulls the DB from `ghcr.io/aquasecurity/trivy-db` anonymously and hits GHCR rate limits, especially behind a shared NAT IP.

**Diagnose →** Error appears in the DB download phase; count how many parallel jobs start Trivy at once.

**Fix →** Use a mirror and cache:
```bash
trivy image --db-repository public.ecr.aws/aquasecurity/trivy-db:2 \
  --java-db-repository public.ecr.aws/aquasecurity/trivy-java-db:1 \
  --cache-dir .trivycache myapp:1.4
```
Cache `.trivycache` between runs (`actions/cache` keyed by date).

**Prevent →** Mirror the DB into your own registry with a scheduled job (`oras copy`), point all jobs at it, and run PR scans with `--skip-db-update` against the cached copy.

</details>

### Q28. Trivy in an air-gapped runner fails with `failed to download vulnerability DB` even though you copied `db.tar.gz` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The DB is not extracted into the expected cache layout, metadata is too old, or the Java DB is still being fetched for JAR scanning.

**Diagnose →**
```bash
ls ~/.cache/trivy/db/        # expect trivy.db and metadata.json
cat ~/.cache/trivy/db/metadata.json
```

**Fix →**
```bash
mkdir -p ~/.cache/trivy/db && tar -xzf db.tar.gz -C ~/.cache/trivy/db
trivy image --skip-db-update --skip-java-db-update --offline-scan myapp:1.4
```

**Prevent →** Automate a daily DB export (`oras pull ghcr.io/aquasecurity/trivy-db:2`) into the internal artifact store and alert when `metadata.json` `UpdatedAt` is older than 48 hours.

</details>

### Q29. Trivy and Grype report different CVE counts for the same image 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Different advisory sources (distro feeds vs NVD vs GHSA), different handling of unfixed and disputed CVEs, and different package detection (language binaries, JARs inside JARs).

**Diagnose →**
```bash
trivy image -f json -o t.json myapp:1.4
grype myapp:1.4 -o json > g.json
jq -r '.Results[].Vulnerabilities[]?.VulnerabilityID' t.json | sort -u > t.txt
jq -r '.matches[].vulnerability.id' g.json | sort -u > g.txt
comm -3 t.txt g.txt
```

**Fix →** Pick one scanner as the gate of record and use the other for second opinion. Investigate the diff list; most differences are severity source or unfixed handling.

**Prevent →** Document the scanner, DB source and severity policy so audits and developers see one answer.

</details>

### Q30. `.trivyignore` entries are ignored and the build still fails on the same CVE 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file is not in the working directory Trivy runs from, the ID differs (GHSA vs CVE alias), or the YAML `.trivyignore.yaml` format is used without `--ignorefile`.

**Diagnose →**
```bash
pwd; ls -la .trivyignore
trivy image --ignorefile ./.trivyignore myapp:1.4 --debug 2>&1 | grep -i ignore
```

**Fix →** Pass the path explicitly and match the ID Trivy prints. Add an expiry:
```
# Not reachable, see SEC-142
CVE-2024-45337 exp:2026-12-31
```

**Prevent →** Every ignore line needs a ticket reference and an expiry date; CI fails when an expiry has passed.

</details>

### Q31. Trivy reports a critical CVE in a Go binary built from your own code, in package `stdlib` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The binary was compiled with an old Go toolchain; Trivy reads the embedded build info and flags the Go standard library version.

**Diagnose →**
```bash
go version -m ./bin/app | head -3
trivy image --format json myapp:1.4 | jq '.Results[] | select(.Type=="gobinary")'
```

**Fix →** Rebuild with a patched Go release (update `golang:1.xx` builder tag and the `go`/`toolchain` directive in `go.mod`).

**Prevent →** Renovate tracks the builder image tag and `toolchain` line; rebuild on new Go patch releases.

</details>

### Q32. Image scan passes in CI but the registry scanner (ECR enhanced scanning / Harbor) shows criticals a week later 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New CVEs were published after the build. A one-time CI scan is a point-in-time snapshot.

**Diagnose →**
```bash
aws ecr describe-image-scan-findings --repository-name myapp --image-id imageTag=1.4 \
  --query 'imageScanFindings.findingSeverityCounts'
```

**Fix →** Rebuild and redeploy with updated base layers; for running workloads, rescan what is deployed (Trivy Operator or registry continuous scanning).

**Prevent →** Continuous scanning of deployed images, plus scheduled rebuilds of all base images.

</details>

### Q33. Trivy Operator creates thousands of `VulnerabilityReport` objects and etcd size alerts fire 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Reports per ReplicaSet revision and per container, kept for old ReplicaSets, plus frequent rescans across many namespaces.

**Diagnose →**
```bash
kubectl get vulnerabilityreports -A --no-headers | wc -l
kubectl get --raw /metrics | grep apiserver_storage_objects | grep -i aquasecurity
```

**Fix →** Limit scope and history: set `revisionHistoryLimit` low on Deployments, exclude system namespaces (`excludeNamespaces`), raise the report TTL scan interval, and disable report types you do not use (config audit, RBAC) in Helm values.

**Prevent →** Export reports to an external store or dashboard instead of keeping everything in etcd; size-test operators before cluster-wide rollout.

</details>

### Q34. Hadolint fails with `DL3008 Pin versions in apt get install` on every Dockerfile 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Packages installed without versions make builds non-reproducible, so the rule fires.

**Diagnose →** `hadolint Dockerfile`

**Fix →** Pin where it matters or document an ignore:
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl=8.5.0-2ubuntu10.6 \
 && rm -rf /var/lib/apt/lists/*
```
Or in `.hadolint.yaml`: `ignored: [DL3008]` with a team decision recorded.

**Prevent →** Agree a Dockerfile standard once; pinned base image digests matter more than every apt version.

</details>

### Q35. Checkov fails 22 checks inside `.terraform/modules/` from a public registry module you call 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Checkov scans downloaded module source when `--download-external-modules true` or when `.terraform/` is present in the scanned directory.

**Diagnose →**
```bash
checkov -d . --compact --quiet -o json | jq -r '.results.failed_checks[] | "\(.check_id) \(.file_path)"' | sort | uniq -c
```

**Fix →** Decide deliberately: scan your code and the module inputs you pass, skip vendored paths (`--skip-path .terraform`), and fix issues you control by setting module variables (e.g. `enable_encryption = true`).

**Prevent →** Maintain an internal approved module that already passes policy, and pin module versions.

</details>

### Q36. Checkov `CKV_AWS_144` (cross-region replication) fails on a dev S3 bucket where it makes no sense 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default policies apply everywhere and do not know environment context.

**Diagnose →** `checkov -f s3.tf --check CKV_AWS_144`

**Fix →** Suppress inline with justification:
```hcl
resource "aws_s3_bucket" "dev_scratch" {
  # checkov:skip=CKV_AWS_144:Dev scratch bucket, no DR requirement
  bucket = "acme-dev-scratch"
}
```
Or a per-directory `.checkov.yaml` with `skip-check` for `envs/dev`.

**Prevent →** Keep skip lists per environment in config, reviewed by security, not ad hoc.

</details>

### Q37. Checkov passes on HCL, but the deployed bucket is still public 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Values come from variables, `locals`, `for_each` or data sources that static HCL scanning cannot resolve, so the check evaluated an unknown value as compliant.

**Diagnose →** Scan the plan, which has resolved values:
```bash
terraform plan -out tf.plan
terraform show -json tf.plan > tf.json
checkov -f tf.json --framework terraform_plan
```

**Fix →** Gate on the plan JSON in CI in addition to HCL scanning, and fix the variable default.

**Prevent →** Add a runtime layer too: AWS Config rules / S3 Block Public Access at account level so a missed scan cannot expose data.

</details>

### Q38. tfsec/Trivy config warns `aws-ec2-no-public-egress-sgr` on every security group 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `egress 0.0.0.0/0` is the AWS default; most workloads need outbound internet or VPC endpoints.

**Diagnose →** `trivy config --severity HIGH,CRITICAL ./infra`

**Fix →** Restrict egress where feasible (VPC endpoints, NAT with egress firewall), otherwise ignore inline with a reason:
```hcl
#trivy:ignore:aws-ec2-no-public-egress-sgr
```
(tfsec's `#tfsec:ignore:` still works in older setups; tfsec has been folded into Trivy.)

**Prevent →** Classify rules as blocking vs informational in a central config instead of hundreds of inline ignores.

</details>

### Q39. KICS or Checkov on Kubernetes manifests flags "Container should not run as root" even though the image sets `USER 10001` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Manifest scanners only see YAML; they cannot read the image `USER`. Without `runAsNonRoot` in `securityContext`, the check fails.

**Diagnose →** Look for `securityContext` at pod and container level in the rendered manifest (`helm template . | yq '.spec.template.spec'`).

**Fix →**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }
```

**Prevent →** Put the hardened `securityContext` in the shared Helm library chart so every service inherits it.

</details>

### Q40. Helm chart passes Checkov in the repo but fails in Kyverno at deploy time 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Checkov scanned the chart with default values; production values files enable a sidecar, `hostPath` or a different `securityContext`.

**Diagnose →**
```bash
helm template app ./chart -f values-prod.yaml > rendered.yaml
checkov -f rendered.yaml --framework kubernetes
kyverno apply ./policies -r rendered.yaml
```

**Fix →** Scan rendered output for every environment's values file, and run the same Kyverno policies in CI with `kyverno apply`.

**Prevent →** One policy source used in both CI (shift-left) and admission (enforcement).

</details>

### Q41. Trivy reports secrets in the image: `AWS_SECRET_ACCESS_KEY` found in `/app/.env` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `COPY . .` without a `.dockerignore` copied the local `.env` into the image.

**Diagnose →**
```bash
trivy image --scanners secret myapp:1.4
docker run --rm --entrypoint ls myapp:1.4 -la /app
```

**Fix →** Rotate the key (the image may already be in a registry), add `.dockerignore` (`.env`, `.git`, `*.pem`), rebuild and delete the old tag/digest from the registry.

**Prevent →** Secret scanning on images in CI, a template `.dockerignore`, and runtime secrets via mounted volumes or env from a secret store.

</details>

### Q42. Secret deleted in a later Dockerfile `RUN rm` step still shows up in the image scan 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Each layer is immutable; the file exists in the earlier layer even if a later layer deletes it.

**Diagnose →**
```bash
docker history --no-trunc myapp:1.4
dive myapp:1.4
```

**Fix →** Use BuildKit secret mounts so the secret never lands in a layer:
```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```
`docker build --secret id=npmrc,src=$HOME/.npmrc .` Then rotate the exposed token.

**Prevent →** Ban `ARG`/`ENV` for credentials in the Dockerfile linter policy.

</details>

### Q43. The image scan reports zero vulnerabilities for a distroless image that you know is outdated 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The scanner did not detect an OS package database (stripped image, scratch base, or custom base without `/var/lib/dpkg/status.d` or `apk` db) so it found no packages to match.

**Diagnose →**
```bash
trivy image --list-all-pkgs -f json myapp:1.4 | jq '.Results[] | {Target, Type, n: (.Packages|length)}'
syft myapp:1.4 -o table | head
```

**Fix →** Use a base that keeps package metadata (Google distroless and Chainguard images do), or generate an SBOM at build time from the source and scan that.

**Prevent →** Treat "0 packages detected" as a failure, not a pass.

</details>

### Q44. Trivy flags CVEs in `/usr/local/lib/python3.12/site-packages/pip` inside the runtime image 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The official Python image ships pip, setuptools and wheel; old versions carry CVEs, even though your app never runs pip at runtime.

**Diagnose →** `trivy image --pkg-types library myapp:1.4 | grep -E "pip|setuptools"`

**Fix →** Upgrade or remove them in the final stage:
```dockerfile
RUN pip install --no-cache-dir -U pip setuptools && pip uninstall -y pip
```
Or use a multi-stage build that copies only the venv into a distroless Python base.

**Prevent →** Minimal runtime stages; build tools stay in the builder stage.

</details>

### Q45. `terraform plan` scanning with Checkov fails: `Unable to parse plan: unsupported format version` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An old Checkov version that does not understand the plan JSON from a newer Terraform/OpenTofu release, or the file is binary `tf.plan` instead of JSON.

**Diagnose →**
```bash
checkov --version; terraform version
head -c 200 tf.json
```

**Fix →** Convert with `terraform show -json tf.plan > tf.json`, and upgrade Checkov (`pip install -U checkov`) or use the current container image.

**Prevent →** Pin scanner versions in CI and bump them alongside Terraform upgrades via Renovate.

</details>

### Q46. Conftest/OPA policy on Terraform plan always passes, even for a resource that clearly violates it 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rego path mismatch: wrong package (`package main` expected by default), rule named `deny` vs `violation`, or iterating `input.resource_changes` with the wrong field (`change.after` is null on deletes).

**Diagnose →**
```bash
conftest test tf.json -p policy --namespace main --trace | head -50
opa eval -i tf.json -d policy 'data.main.deny' --format pretty
```

**Fix →** Match the package and rule names Conftest expects and guard nulls:
```rego
package main
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket_public_access_block"
  rc.change.after.block_public_acls == false
  msg := sprintf("%s must block public ACLs", [rc.address])
}
```

**Prevent →** Unit-test policies with `opa test` using passing and failing fixtures.

</details>

### Q47. Scanning a multi-arch image gives different results on the amd64 and arm64 runner 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The tag points to an index; each runner resolves its own platform manifest, which may come from different base layers or package versions.

**Diagnose →**
```bash
docker buildx imagetools inspect myapp:1.4
trivy image --platform linux/arm64 myapp:1.4
trivy image --platform linux/amd64 myapp:1.4
```

**Fix →** Scan every platform explicitly in CI and gate on the union.

**Prevent →** Scan and sign by digest, per platform, as part of the release job.

</details>

### Q48. CI image scan fails with `unable to inspect the image: ... unauthorized: authentication required` for a private ECR repo 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Trivy is not logged in to the registry, or the ECR token (valid 12 hours) expired in a long pipeline.

**Diagnose →** `aws sts get-caller-identity` and `aws ecr get-login-password | docker login ...` result.

**Fix →**
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
trivy image 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.4
```
Or scan the local image before push.

**Prevent →** Scan in the build job right after `docker build` and before `docker push`.

</details>

### Q49. Trivy scan job sometimes takes 2 seconds and passes, sometimes 4 minutes and fails, for the same image 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A stale scan result cache keyed by image ID is reused, or the DB update failed silently with `--skip-db-update` and an old DB.

**Diagnose →**
```bash
trivy image --debug myapp:1.4 2>&1 | grep -E "DB|cache"
trivy --version   # shows DB version and UpdatedAt
```

**Fix →** Clear or scope the cache (`trivy clean --scan-cache`), refresh the DB at job start, and print DB metadata in logs.

**Prevent →** Fail the job if the DB is older than a threshold, so results are reproducible and explainable.

</details>

### Q50. After you enabled `--exit-code 1` for MEDIUM, every team's pipeline is red and they start adding blanket ignores 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The gate threshold was raised without a baseline or migration window, so the backlog blocked delivery.

**Diagnose →** Count findings per repo/severity from the last scans to size the backlog.

**Fix →** Roll back to HIGH/CRITICAL fixable blocking, report MEDIUM as warnings, and give teams a dated target. Remove blanket ignores and replace them with expiring entries.

**Prevent →** Change gates with a ratchet: block on new findings only, then tighten per quarter with metrics.

</details>

## Supply chain: signing, SBOM and dependencies

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. `cosign verify` fails with `Error: no matching signatures` for an image CI says it signed 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CI signed a different digest (the tag was re-pushed after signing), signed a per-platform manifest while you verify the index, or pushed the signature to a different repo (`COSIGN_REPOSITORY`).

**Diagnose →**
```bash
crane digest ghcr.io/acme/api:1.4
cosign tree ghcr.io/acme/api:1.4
cosign triangulate ghcr.io/acme/api:1.4   # where cosign looks for the .sig
```

**Fix →** Sign by digest straight after push using the digest output of the build step:
```bash
cosign sign --yes ghcr.io/acme/api@${DIGEST}
```
Verify the same digest.

**Prevent →** Never sign tags; pass `steps.build.outputs.digest` through the pipeline and deploy by digest.

</details>

### Q52. Keyless `cosign verify` fails: `none of the expected identities matched what was in the certificate` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--certificate-identity` does not exactly match the Fulcio cert SAN, which for GitHub Actions is the workflow ref, e.g. `https://github.com/acme/api/.github/workflows/release.yml@refs/tags/v1.4.0`, or the issuer is wrong.

**Diagnose →**
```bash
cosign verify ghcr.io/acme/api@sha256:... \
  --certificate-identity-regexp '.*' --certificate-oidc-issuer-regexp '.*' -o json \
  | jq -r '.[0].optional | {Issuer, Subject}'
```

**Fix →** Use the real values, with a constrained regexp for tags:
```bash
cosign verify ghcr.io/acme/api@sha256:... \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github.com/acme/api/.github/workflows/release.yml@refs/tags/v.*$'
```

**Prevent →** Keep the identity in one policy file used by CI verification and the admission controller. Never ship the `.*` debug regexps.

</details>

### Q53. `cosign sign` in GitHub Actions fails with `getting signer: getting key from Fulcio: ... no OIDC token` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job lacks `id-token: write`, so it cannot request the GitHub OIDC token for keyless signing.

**Diagnose →** Check `permissions:` and that `ACTIONS_ID_TOKEN_REQUEST_URL` is set in the job env.

**Fix →**
```yaml
permissions:
  contents: read
  packages: write
  id-token: write
steps:
  - uses: sigstore/cosign-installer@v3
  - run: cosign sign --yes ghcr.io/acme/api@${{ steps.build.outputs.digest }}
```

**Prevent →** Grant `id-token: write` only to the release job, not workflow-wide.

</details>

### Q54. Key-based `cosign verify --key cosign.pub` fails after a key rotation for older images 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Older images were signed with the previous key, and the policy trusts only the new public key.

**Diagnose →** Verify against both keys; check which signed which digest.

**Fix →** Trust both public keys during a transition (Kyverno `attestors` with `count: 1` over two key entries), re-sign images still in use with the new key, then remove the old one.

**Prevent →** Plan rotation with an overlap window, store keys in KMS (`cosign sign --key awskms:///alias/cosign`), or move to keyless to remove long-lived keys.

</details>

### Q55. `cosign verify-attestation --type cyclonedx` returns `none of the attestations matched the predicate type` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The attestation was created with a different predicate type (e.g. `spdxjson`, or a custom URI) or attached to a different digest.

**Diagnose →**
```bash
cosign tree ghcr.io/acme/api@sha256:...
cosign download attestation ghcr.io/acme/api@sha256:... | jq -r '.payload' | base64 -d | jq -r .predicateType
```

**Fix →** Verify with the type that was used, or regenerate:
```bash
syft ghcr.io/acme/api@sha256:... -o cyclonedx-json > sbom.cdx.json
cosign attest --yes --type cyclonedx --predicate sbom.cdx.json ghcr.io/acme/api@sha256:...
```

**Prevent →** Standardise predicate types in the release template.

</details>

### Q56. Kyverno `verifyImages` rejects pods: `failed to verify image ... : no signatures found` for a signed private ECR image 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Kyverno cannot read the registry (no pull credentials, no IRSA on the Kyverno admission controller), so it finds no signature artifact; or it cannot reach the Rekor/Fulcio public endpoints from a private cluster.

**Diagnose →**
```bash
kubectl -n kyverno logs deploy/kyverno-admission-controller | grep -i -E "verify|unauthorized|rekor"
kubectl -n kyverno get sa kyverno-admission-controller -o yaml | grep role-arn
```

**Fix →** Give Kyverno registry read access (IRSA role with `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`, or `imagePullSecrets` in the rule), and allow egress to `rekor.sigstore.dev` and `fulcio.sigstore.dev` or run a private Sigstore stack.

**Prevent →** Test policies with `kyverno test` and run a canary namespace in Audit mode before Enforce.

</details>

### Q57. After enabling signature verification, pods fail because the manifest uses `image: api:1.4` and the policy mutates it to a digest the registry rejects 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Kyverno `verifyImages` with `mutateDigest: true` resolves the tag at admission; the tag was moved, or a short name resolved to `docker.io/library/api` instead of your registry.

**Diagnose →** `kubectl get pod <p> -o jsonpath='{.spec.containers[*].image}'` and compare with `crane digest <full-ref>`.

**Fix →** Use fully-qualified references by digest in manifests (`ghcr.io/acme/api@sha256:...`), set the policy `imageReferences` to your registry only.

**Prevent →** GitOps writes digests, not tags (Renovate or Argo CD Image Updater with digest strategy).

</details>

### Q58. SBOM generated in CI does not match what scanners find in the running image 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SBOM came from source (`syft dir:.`) or the lockfile, not the built image; the image includes OS packages and build-time additions the source SBOM lacks, or the image was rebuilt after SBOM generation.

**Diagnose →**
```bash
syft ghcr.io/acme/api@sha256:... -o cyclonedx-json > image.cdx.json
jq -r '.components[] | "\(.name)@\(.version)"' sbom-ci.cdx.json | sort > a.txt
jq -r '.components[] | "\(.name)@\(.version)"' image.cdx.json | sort > b.txt
diff a.txt b.txt | head
```

**Fix →** Generate the SBOM from the final image digest and attach it as an attestation to that digest.

**Prevent →** One pipeline step: build → push → SBOM by digest → attest → sign, all keyed on the same digest.

</details>

### Q59. Grype scanning an SBOM reports far fewer vulnerabilities than scanning the image directly 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SBOM lacks the distro identity (no `os-release` info) or purls/CPEs, so OS packages cannot be matched to distro advisories.

**Diagnose →**
```bash
jq '.metadata.component, (.components[] | select(.type=="operating-system"))' sbom.cdx.json
jq '[.components[] | select(.purl==null)] | length' sbom.cdx.json
```

**Fix →** Regenerate the SBOM with Syft from the image (it records distro and purls), or pass `--distro debian:12` to Grype.

**Prevent →** Validate SBOM quality in CI: require purls on components and an OS component.

</details>

### Q60. `npm install` in CI pulled `@acme/billing-utils` from the public npm registry instead of the internal one 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Dependency confusion: the scope is not mapped to the internal registry in `.npmrc`, or a proxy resolves public before private and someone published a higher version publicly.

**Diagnose →**
```bash
npm config get @acme:registry
jq '.packages["node_modules/@acme/billing-utils"].resolved' package-lock.json
npm view @acme/billing-utils --registry https://registry.npmjs.org
```

**Fix →** Treat the build as compromised: rotate CI secrets the install scripts could read, then pin the scope:
```
@acme:registry=https://artifactory.acme.internal/api/npm/npm-internal/
```
Regenerate the lockfile and verify every `resolved` URL.

**Prevent →** Register your scope on npmjs.com, route all installs through a proxy that never resolves internal names publicly, and fail CI if `resolved` URLs point outside approved registries (`lockfile-lint --allowed-hosts`).

</details>

### Q61. pip in CI installed an attacker-controlled `acme-auth` from PyPI despite `--extra-index-url` pointing to your private index 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--extra-index-url` merges both indexes and pip picks the highest version from either; a public package with a bigger version wins.

**Diagnose →** `pip download acme-auth -v 2>&1 | grep -i "found link"`, and check which index served it.

**Fix →** Use a single `--index-url` pointing to a proxy (Artifactory/Nexus/CodeArtifact) that serves internal names only from the internal repo, and hash-pin:
```bash
pip install --require-hashes --index-url https://pypi.acme.internal/simple -r requirements.txt
```
Rotate any secrets the build had access to.

**Prevent →** Ban `--extra-index-url` in CI templates; reserve internal names on PyPI.

</details>

### Q62. `npm ci` fails with `EINTEGRITY sha512-... integrity checksum failed` after a registry migration 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The tarball served differs from the hash in the lockfile: a proxy rewrote or re-packed it, a corrupted cache, or a genuinely tampered package.

**Diagnose →**
```bash
npm cache verify
npm view <pkg>@<ver> dist.integrity --registry https://registry.npmjs.org
```
Compare with the lockfile value.

**Fix →** If the public registry hash matches the lockfile, fix or purge the proxy cache entry. Do not just delete the lockfile to "make it work".

**Prevent →** Treat integrity failures as security events until explained; alert on them.

</details>

### Q63. Renovate opened a PR bumping a GitHub Action from a SHA to a new SHA, and a reviewer cannot tell what changed 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Actions are pinned by commit SHA (correct) without a version comment.

**Diagnose →** Check `uses:` lines lack `# v4.2.2` comments.

**Fix →** Pin with a trailing version comment so Renovate updates both and the PR shows the release notes:
```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```
Enable Renovate `helpers:pinGitHubActionDigests`.

**Prevent →** Org policy requiring SHA-pinned actions, plus allowlisted action owners in repository settings.

</details>

### Q64. A third-party GitHub Action you use was compromised and printed secrets into public logs 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Actions referenced by mutable tag (`@v1`), and the attacker repointed tags to malicious code, as in the `tj-actions/changed-files` incident (2025).

**Diagnose →**
```bash
grep -rn "uses: .*@v" .github/workflows/
gh run list --workflow ci.yml --limit 50   # runs in the exposure window
```
Download logs and search for base64 blobs.

**Fix →** Rotate every secret available to affected workflows, delete exposed logs, pin to a known-good SHA or replace the action.

**Prevent →** SHA pinning, minimal `permissions:`, secrets scoped to environments with required reviewers, and egress monitoring on runners (e.g. Harden-Runner).

</details>

### Q65. SLSA provenance verification fails: `slsa-verifier: ... expected source 'github.com/acme/api', got 'github.com/acme-fork/api'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The artifact was built from a fork or a different repo, or the verifier was given the wrong `--source-uri`.

**Diagnose →**
```bash
slsa-verifier verify-image ghcr.io/acme/api@sha256:... \
  --source-uri github.com/acme/api --source-tag v1.4.0 --print-provenance | jq .predicate.invocation
```

**Fix →** If the source is wrong, do not deploy: rebuild from the canonical repo. If your parameters were wrong, correct `--source-uri`/`--source-branch`.

**Prevent →** Only release workflows in the canonical repo can push to production registries.

</details>

### Q66. `gh attestation verify` fails with `no attestations found` for a release binary 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow did not run `actions/attest-build-provenance`, lacked `attestations: write`/`id-token: write`, or you verify a re-zipped file with a different hash.

**Diagnose →**
```bash
sha256sum app-linux-amd64
gh attestation verify app-linux-amd64 --owner acme
```

**Fix →**
```yaml
permissions: { id-token: write, attestations: write, contents: read }
- uses: actions/attest-build-provenance@v2
  with: { subject-path: dist/app-linux-amd64 }
```

**Prevent →** Verify attestations as a step in the deploy job so a missing one fails early.

</details>

### Q67. Cosign signing in CI fails intermittently with `error signing ... Post "https://rekor.sigstore.dev/api/v1/log/entries": context deadline exceeded` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Network egress restrictions, proxy without TLS interception exemption, or a public-good Sigstore outage.

**Diagnose →**
```bash
curl -sv https://rekor.sigstore.dev/api/v1/log | head
echo $HTTPS_PROXY
```

**Fix →** Allow egress to `fulcio.sigstore.dev`, `rekor.sigstore.dev`, `tuf-repo-cdn.sigstore.dev`, retry with backoff, and set `COSIGN_TIMEOUT`/`--timeout`.

**Prevent →** For strict networks, run a private Sigstore (Fulcio + Rekor) or use KMS keys; monitor status.sigstore.dev.

</details>

### Q68. `cosign verify` fails with `error verifying bundle: ... tlog entry not found` in an air-gapped cluster 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Verification tries to fetch the Rekor inclusion proof online and the TUF root can't be refreshed.

**Diagnose →** Run with `--verbose` and look for outbound calls to Rekor/TUF.

**Fix →** Use offline verification with the bundle and a mirrored TUF root:
```bash
cosign verify --offline --trusted-root trusted_root.json ...
```
Or sign with your own key and `--insecure-ignore-tlog=true` only where an internal transparency log policy replaces it.

**Prevent →** Decide on transparency strategy (public vs private Rekor) before going air-gapped.

</details>

### Q69. Licence scan fails the build: `AGPL-3.0` detected in a transitive Go module 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A new dependency pulled an AGPL module; your policy blocks strong copyleft for SaaS.

**Diagnose →**
```bash
go mod why -m github.com/some/agpl-lib
trivy fs --scanners license --severity HIGH,CRITICAL .
```

**Fix →** Replace the dependency or the parent that pulls it; if it is test-only, confirm it is not linked into the shipped binary and record the decision with legal.

**Prevent →** Run licence checks on PRs that change `go.mod`, with an allowlist agreed with legal.

</details>

### Q70. Your SBOM lists component versions as `UNKNOWN` for half the JARs 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Shaded/fat JARs without `pom.properties` or manifest version info, so the generator cannot identify nested components.

**Diagnose →**
```bash
unzip -l app.jar | grep -E "pom.properties|MANIFEST.MF" | head
syft app.jar -o table | grep -i unknown
```

**Fix →** Generate the SBOM at build time with the build tool (CycloneDX Maven/Gradle plugin), which knows exact resolved versions, and merge it with the image SBOM.

**Prevent →** Build-tool SBOMs for application deps + image SBOMs for OS packages.

</details>

### Q71. Dependabot alerts stay open after you upgraded the vulnerable package 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another manifest or lockfile in the repo (a sub-project, a sample app) still references the old version, or the lockfile was not committed.

**Diagnose →** Alert details show the manifest path; `git grep "lodash@4.17.20"`.

**Fix →** Update every manifest/lockfile listed, or delete unused sample projects.

**Prevent →** Keep a single lockfile per deployable and remove dead code from repos.

</details>

### Q72. `go build` in CI fails with `verifying module: checksum mismatch` for an internal module 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Go tries to verify a private module against the public checksum DB (`sum.golang.org`), or a tag was moved after `go.sum` recorded it.

**Diagnose →**
```bash
go env GOPRIVATE GONOSUMDB GOPROXY
git ls-remote --tags https://git.acme.internal/lib/auth
```

**Fix →** `go env -w GOPRIVATE=git.acme.internal/*` so private modules bypass the public proxy and sumdb. If a tag was moved, cut a new version; never rewrite released tags.

**Prevent →** Protect tags in the Git server and set `GOPRIVATE` in CI images.

</details>

### Q73. An image signed in staging fails verification in production with the same digest 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Images were copied between registries (`crane copy`/`skopeo copy`) without the `.sig`/`.att` artifacts, or production verifies a different signer identity (prod policy expects the release workflow, staging was signed by CI).

**Diagnose →**
```bash
cosign tree prod.registry/acme/api@sha256:...
cosign verify prod.registry/acme/api@sha256:... --certificate-identity-regexp '.*' --certificate-oidc-issuer-regexp '.*'
```

**Fix →** Copy with signatures: `cosign copy staging.registry/acme/api@sha256:... prod.registry/acme/api`. If the identity differs, sign again in the promotion workflow (an extra signature for "approved for prod").

**Prevent →** Promotion tooling copies referrers/signatures, and policies are versioned per environment.

</details>

### Q74. Scanner says the image is affected by CVE-X but your VEX says `not_affected`; the gate still fails 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The scanner is not reading the VEX document, the product/purl identifiers in the VEX do not match the scanned components, or the format is unsupported.

**Diagnose →**
```bash
trivy image --vex ./vex.openvex.json --debug ghcr.io/acme/api@sha256:... 2>&1 | grep -i vex
```

**Fix →** Use exact purls (including version) in `products`/`subcomponents`, and pass the VEX file explicitly or attach it as an attestation the scanner can discover.

**Prevent →** Generate VEX with tooling (`vexctl create`) and test it in CI against the gate.

</details>

### Q75. A typosquatted package `reqeusts` was found in a developer's requirements file 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A typo in an install command or copy-paste; malicious lookalikes often steal environment variables on install.

**Diagnose →**
```bash
pip show reqeusts; pip download reqeusts --no-deps -d /tmp/x && unzip -l /tmp/x/*.whl
```
Check where it ran (laptop, CI) and what credentials were present.

**Fix →** Remove it, treat the machine/runner as compromised, rotate credentials it could read, and report the package to PyPI.

**Prevent →** Resolve through a curated proxy with malicious-package blocking, and review new dependencies in PRs.

</details>

## Kubernetes admission control and Pod Security

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. `Error from server: admission webhook "validate.kyverno.svc-fail" denied the request: policy Deployment/prod/api for resource violation: require-run-as-nonroot` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A Kyverno ClusterPolicy in `Enforce` mode requires `runAsNonRoot: true` and the Deployment does not set it.

**Diagnose →**
```bash
kubectl get clusterpolicy require-run-as-nonroot -o yaml | yq '.spec.rules[].validate'
kyverno apply policies/require-run-as-nonroot.yaml -r deploy.yaml
```

**Fix →** Set the `securityContext` at pod level (`runAsNonRoot: true`, `runAsUser: 10001`) and make sure the image actually runs as a non-root UID.

**Prevent →** Run `kyverno apply` against rendered manifests in CI so developers see the same message before deploy.

</details>

### Q77. Pods do not appear, but `kubectl apply` of the Deployment succeeded 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Admission policies checked the Pod (created by the ReplicaSet controller), not the Deployment, so the rejection is recorded on the ReplicaSet, not shown to you.

**Diagnose →**
```bash
kubectl describe rs -l app=api | grep -A5 Events
kubectl get events -n prod --field-selector reason=FailedCreate
```
Look for `admission webhook ... denied the request` or `violates PodSecurity`.

**Fix →** Fix the pod template as the message says.

**Prevent →** Kyverno auto-generates rules for controllers (`autogen`); for Gatekeeper and PSA, validate workloads in CI too.

</details>

### Q78. `pods "api-7d9" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false, unrestricted capabilities, seccompProfile` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The namespace has `pod-security.kubernetes.io/enforce=restricted` and the pod spec misses required fields.

**Diagnose →**
```bash
kubectl get ns prod --show-labels
kubectl label --dry-run=server --overwrite ns prod pod-security.kubernetes.io/enforce=restricted
```

**Fix →**
```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile: { type: RuntimeDefault }
containers:
- name: api
  securityContext:
    allowPrivilegeEscalation: false
    capabilities: { drop: ["ALL"] }
```

**Prevent →** Roll out PSA with `warn` and `audit` labels before `enforce`.

</details>

### Q79. Enabling `enforce=restricted` on a namespace did not evict the existing non-compliant pods 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pod Security Admission is admission-time only; it never touches running pods. They fail only when recreated.

**Diagnose →** The `kubectl label` command printed warnings about existing pods; or dry-run it again with `--dry-run=server`.

**Fix →** Fix the workloads and roll them (`kubectl rollout restart deploy -n prod`) in a controlled window so you find failures now, not during a node drain at 3 a.m.

**Prevent →** Audit first with `audit`/`warn` labels and fix everything before enforcing.

</details>

### Q80. `Error from server (InternalError): Internal error occurred: failed calling webhook "validation.gatekeeper.sh": ... context deadline exceeded` blocks every deploy 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Gatekeeper pods are down, overloaded, or unreachable from the API server (network policy, private cluster firewall on port 8443), and the webhook is `failurePolicy: Fail`.

**Diagnose →**
```bash
kubectl -n gatekeeper-system get pods
kubectl -n gatekeeper-system get endpoints gatekeeper-webhook-service
kubectl get validatingwebhookconfiguration gatekeeper-validating-webhook-configuration -o yaml | grep -E "failurePolicy|timeoutSeconds"
```

**Fix →** Restore the pods (scale, resources), open API-server-to-webhook traffic (GKE private clusters need a firewall rule for the webhook port).

**Prevent →** Run 3 replicas with a PDB and anti-affinity, exempt `kube-system`, set reasonable `timeoutSeconds`, and alert on webhook latency/rejections.

</details>

### Q81. Kyverno is down and now the whole cluster, including Kyverno's own namespace, cannot be fixed because writes are rejected 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fail-closed webhooks that match all namespaces, including the one Kyverno needs to recover.

**Diagnose →**
```bash
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations | grep kyverno
kubectl -n kyverno get pods
```

**Fix →** Break glass: temporarily delete the Kyverno webhook configurations (Kyverno recreates them when healthy), recover the pods, and confirm webhooks return.
```bash
kubectl delete validatingwebhookconfiguration kyverno-resource-validating-webhook-cfg
kubectl delete mutatingwebhookconfiguration kyverno-resource-mutating-webhook-cfg
```

**Prevent →** Exclude `kube-system` and `kyverno` namespaces in webhook config, HA replicas, and a documented break-glass runbook with audited access.

</details>

### Q82. Gatekeeper constraint is created but violations are not blocked; `kubectl get constraint` shows `TOTAL-VIOLATIONS: 12` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The constraint's `enforcementAction` is `dryrun` or `warn`, or its `match` does not include the kinds/namespaces you expect.

**Diagnose →**
```bash
kubectl get k8srequiredlabels require-team -o yaml | yq '.spec.enforcementAction, .spec.match'
kubectl get k8srequiredlabels require-team -o jsonpath='{.status.violations}' | jq
```

**Fix →** After fixing existing violators, set `enforcementAction: deny` and ensure `match.kinds` has the right `apiGroups` (`apps` for Deployments, `""` for Pods).

**Prevent →** Promote constraints dryrun → warn → deny with violation count at zero as the gate.

</details>

### Q83. Gatekeeper constraint fails to apply: `no matches for kind "K8sRequiredLabels" in version "constraints.gatekeeper.sh/v1beta1"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ConstraintTemplate was not applied yet or failed to compile, so the CRD for the constraint kind does not exist. In GitOps, ordering applied them together.

**Diagnose →**
```bash
kubectl get constrainttemplate k8srequiredlabels -o jsonpath='{.status}' | jq
kubectl get crd | grep constraints.gatekeeper.sh
```

**Fix →** Apply the template first, fix Rego compile errors shown in `status.byPod[].errors`, then the constraint.

**Prevent →** Argo CD sync waves (templates wave 0, constraints wave 1) or `SkipDryRunOnMissingResource=true`.

</details>

### Q84. Gatekeeper Rego for "images must come from our registry" blocks images from `registry.acme.com.evil.io` correctly, but allows `registry.acme.comevil.io/x` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The policy uses `startswith(image, "registry.acme.com")` without a trailing `/`, so lookalike hostnames pass.

**Diagnose →** Write a unit test with lookalike hostnames: `opa test ./policies -v`.

**Fix →**
```rego
violation[{"msg": msg}] {
  c := input_containers[_]
  not startswith(c.image, "registry.acme.com/")
  msg := sprintf("image %v not from approved registry", [c.image])
}
```
Also include `initContainers` and `ephemeralContainers` in `input_containers`.

**Prevent →** Policy unit tests for bypass cases, reviewed like code.

</details>

### Q85. Pods pass admission, but someone exec'd in with `kubectl debug` and an ephemeral container that runs privileged 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Policies check `containers` and `initContainers` only; ephemeral containers are added via the `pods/ephemeralcontainers` subresource, which the webhook does not match.

**Diagnose →**
```bash
kubectl get pod api-7d9 -o jsonpath='{.spec.ephemeralContainers}' | jq
kubectl get clusterpolicy -o yaml | grep -n ephemeral
```

**Fix →** Extend policies to `ephemeralContainers` and include the subresource in rule matches (Kyverno pod-security policies cover it; Gatekeeper needs `pods/ephemeralcontainers` operations). PSA already evaluates ephemeral containers.

**Prevent →** RBAC: restrict `pods/ephemeralcontainers` and `pods/exec` to on-call roles with audit logging.

</details>

### Q86. Kyverno mutate policy that adds default `securityContext` does not apply to pods created before the policy existed 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Mutating admission only runs on create/update requests. Existing objects are unchanged.

**Diagnose →** Check pod `creationTimestamp` vs policy creation; `kubectl get pod -o yaml | grep -A5 securityContext`.

**Fix →** Restart workloads to pass them through admission, or use `mutateExistingOnPolicyUpdate: true` for supported targets (not for immutable pod fields).

**Prevent →** Prefer fixing manifests in Git; mutation is a safety net, not the source of truth.

</details>

### Q87. Argo CD sync fails with `admission webhook "validate.kyverno.svc-fail" denied the request` only for the Job hook resource 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Argo CD hook Jobs (migrations) are created by a different template, without the hardened `securityContext` or required labels the policy expects.

**Diagnose →** Argo CD UI → sync result message; `kubectl get events -n prod | grep -i denied`.

**Fix →** Fix the hook Job template in the chart to meet policy. Do not add a blanket exclusion for Jobs.

**Prevent →** Render the whole chart, hooks included, and run `kyverno apply` in CI.

</details>

### Q88. Kyverno `PolicyReport` shows 400 `fail` results, but nothing was blocked 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Policies run with `validationFailureAction: Audit` (or `failureAction: Audit` in newer versions), and background scans report existing violations.

**Diagnose →**
```bash
kubectl get policyreport -A
kubectl get policyreport -n prod -o json | jq -r '.results[] | select(.result=="fail") | "\(.policy) \(.resources[0].name)"' | sort | uniq -c
```

**Fix →** That is expected in audit. Use the reports to drive fixes per team, then switch to Enforce.

**Prevent →** Publish policy reports to a dashboard (Policy Reporter) so teams see their own debt.

</details>

### Q89. A Kyverno policy excluded `kube-system` by namespace, yet a system DaemonSet is still blocked after a Helm upgrade 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The upgraded chart installs into a different namespace (e.g. `aws-observability`, `monitoring`), or the exclusion uses `namespaces` under `match` instead of `exclude`.

**Diagnose →**
```bash
kubectl get clusterpolicy disallow-host-path -o yaml | yq '.spec.rules[] | {match, exclude}'
kubectl get ds -A | grep <name>
```

**Fix →** Add a narrowly scoped `exclude` (namespace + workload name) or a `PolicyException` resource with the exact justification.

**Prevent →** Use Kyverno `PolicyException` objects in Git with owners and expiry instead of editing policies.

</details>

### Q90. Pod creation fails with `container has runAsNonRoot and image will run as root` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The kubelet check: the pod requires non-root but the image has no numeric `USER` (or `USER root`), and no `runAsUser` is set.

**Diagnose →**
```bash
kubectl describe pod api-7d9 | grep -A3 "Error"
docker inspect --format '{{.Config.User}}' ghcr.io/acme/api:1.4
```

**Fix →** Add `USER 10001` (numeric) in the Dockerfile, or set `runAsUser: 10001` and ensure file permissions allow that UID.

**Prevent →** Lint Dockerfiles for a numeric non-root `USER`.

</details>

### Q91. App crashes with `EACCES: permission denied, open '/app/cache/x'` after `readOnlyRootFilesystem: true` and non-root were enforced 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app writes to its image filesystem, which is now read-only, or the directory is owned by root.

**Diagnose →** `kubectl logs`, then find write paths: `kubectl debug -it pod/api-7d9 --image=busybox --target=api -- ls -ld /app/cache`.

**Fix →** Mount writable `emptyDir` volumes at the write paths, and set `fsGroup` if volume ownership matters:
```yaml
volumeMounts: [{ name: cache, mountPath: /app/cache }]
volumes: [{ name: cache, emptyDir: {} }]
```

**Prevent →** Test images with `docker run --read-only --user 10001` in CI.

</details>

### Q92. A policy requiring resource limits blocks Istio-injected pods because the sidecar has no limits 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Mutating webhooks (Istio) run before validating ones; the injected `istio-proxy` container fails your validation.

**Diagnose →** `kubectl get events` shows the container name `istio-proxy` in the denial.

**Fix →** Set sidecar resources via the mesh config or pod annotations (`sidecar.istio.io/proxyCPULimit`), or scope the policy to exclude that container by name with a documented reason.

**Prevent →** Test policies against fully mutated pods (server-side dry-run: `kubectl apply --dry-run=server`).

</details>

### Q93. `kubectl apply --dry-run=server` passes, but the real apply is denied a minute later 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Webhooks marked `sideEffects: Unknown/Some` are skipped on dry-run, or the policy uses external data (image signature lookup, API calls, ConfigMap context) that changed or timed out.

**Diagnose →**
```bash
kubectl get validatingwebhookconfiguration -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.webhooks[*].sideEffects}{"\n"}{end}'
```

**Fix →** Make webhooks declare `sideEffects: None` (and support dry-run), and stabilise external lookups (caching, timeouts).

**Prevent →** Treat dry-run as indicative, and rely on CI policy tests plus canary deploys.

</details>

### Q94. Gatekeeper `audit` reports violations for resources that no longer exist 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Audit results are refreshed on an interval (default 60s) and capped by `--constraint-violations-limit`; status can lag, or audit pods are failing.

**Diagnose →**
```bash
kubectl -n gatekeeper-system logs deploy/gatekeeper-audit | tail
kubectl get k8srequiredlabels require-team -o jsonpath='{.status.auditTimestamp}'
```

**Fix →** Fix the audit pod (memory limits are a common cause on big clusters) and wait for the next cycle.

**Prevent →** Size audit resources for cluster object count and use `--audit-from-cache` with sync config for what you query.

</details>

### Q95. Gatekeeper policy that checks "Ingress host must be unique" never fires 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Policies referencing other objects use `data.inventory`, which is empty unless you configure Gatekeeper to sync those kinds.

**Diagnose →** `kubectl get config config -n gatekeeper-system -o yaml` and check `spec.sync.syncOnly`.

**Fix →**
```yaml
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata: { name: config, namespace: gatekeeper-system }
spec:
  sync:
    syncOnly:
    - { group: "networking.k8s.io", version: "v1", kind: "Ingress" }
```

**Prevent →** Document data dependencies next to each ConstraintTemplate and test with replicated inventory in `gator verify`.

</details>

### Q96. After upgrading to Kubernetes 1.30, a ValidatingAdmissionPolicy fails with `expression ... no such key: securityContext` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CEL expressions access optional fields without `has()`, so pods without that field cause evaluation errors.

**Diagnose →**
```bash
kubectl get validatingadmissionpolicy require-nonroot -o yaml | yq '.spec.validations'
kubectl get validatingadmissionpolicy require-nonroot -o jsonpath='{.status.typeChecking}'
```

**Fix →** Guard optional fields:
```yaml
expression: >
  object.spec.containers.all(c, has(c.securityContext) &&
    has(c.securityContext.runAsNonRoot) && c.securityContext.runAsNonRoot == true)
```
Also set `failurePolicy` deliberately.

**Prevent →** Test VAPs in a kind cluster in CI with good and bad pod fixtures.

</details>

### Q97. A namespace label `pod-security.kubernetes.io/enforce=baseline` is silently removed after every Argo CD sync 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Namespace is managed in Git without the label (or created via `CreateNamespace=true` without `managedNamespaceMetadata`), so sync reverts manual labels.

**Diagnose →** `argocd app diff <app>` shows the label removal.

**Fix →** Put PSA labels in the Namespace manifest in Git, or in `syncPolicy.managedNamespaceMetadata.labels`.

**Prevent →** No manual security labels; enforce namespace labels with a policy that requires them.

</details>

### Q98. Developers run `kubectl apply` and get a warning `would violate PodSecurity "restricted:latest"` but the pod is created 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The namespace has `warn=restricted` but `enforce` is baseline or unset. Warnings are informational.

**Diagnose →** `kubectl get ns <ns> -o jsonpath='{.metadata.labels}'`

**Fix →** Fix the fields listed in the warning now, before enforcement is raised.

**Prevent →** Make CI fail on PSA warnings by running `kubectl apply --dry-run=server` and grepping for `Warning:`.

</details>

### Q99. Kyverno `generate` policy for a default-deny NetworkPolicy created it in new namespaces, but someone deleted it and it did not come back 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `synchronize: false` in the generate rule, so Kyverno does not keep the downstream resource in sync.

**Diagnose →** `kubectl get clusterpolicy add-default-deny -o yaml | yq '.spec.rules[].generate.synchronize'`

**Fix →** Set `synchronize: true`; Kyverno recreates deleted resources and reverts edits.

**Prevent →** Restrict RBAC on NetworkPolicy deletion in tenant namespaces and alert on deletes from audit logs.

</details>

### Q100. Policy webhooks add 2–3 seconds to every pod creation and HPA scale-ups lag 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Image verification hitting remote registries on every admission, expensive Rego/JMESPath over large lists, or too few webhook replicas.

**Diagnose →**
```bash
kubectl get --raw /metrics | grep apiserver_admission_webhook_admission_duration_seconds_bucket | grep -E "kyverno|gatekeeper" | tail
```

**Fix →** Enable image verification caching, scope `verifyImages` to your registries, scale webhooks horizontally, and narrow `match` to the kinds that need checks.

**Prevent →** Track webhook p99 latency SLOs and load-test policies before rollout.

</details>

## Secrets management: Vault, External Secrets and SOPS

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `vault kv get secret/app/db` returns `Code: 403. Errors: * permission denied` although the policy allows `secret/app/*` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The mount is KV v2. The CLI hides it, but the real API paths are `secret/data/app/db` (read) and `secret/metadata/app/db` (list). A policy written for `secret/app/*` matches nothing.

**Diagnose →**
```bash
vault secrets list -detailed | grep '^secret/'        # options: map[version:2]
vault token capabilities secret/data/app/db           # shows "deny"
```

**Fix →**
```hcl
path "secret/data/app/*"     { capabilities = ["read"] }
path "secret/metadata/app/*" { capabilities = ["list", "read"] }
```

**Prevent →** Write policies with the `data/` and `metadata/` prefixes and test each one with `vault token capabilities` in CI before applying.

</details>

### Q102. Vault pod restarted and `vault status` shows `Sealed: true`; every app now fails to fetch secrets 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Vault was set up with Shamir unseal keys, so every restart needs manual unsealing. Or auto-unseal is configured but the KMS key or Vault's IAM permission for it is gone.

**Diagnose →**
```bash
kubectl exec -n vault vault-0 -- vault status     # Seal Type: shamir / awskms
kubectl logs -n vault vault-0 | grep -i -E "seal|kms"
```

**Fix →** Short term: `vault operator unseal` with the quorum of key holders on each pod. Long term: switch to auto-unseal (`seal "awskms"`, `gcpckms`, `azurekeyvault` or `transit`) using `vault operator unseal -migrate`.

**Prevent →** In production we always use auto-unseal, run 3 or 5 Raft nodes across zones, and alert on `vault_core_unsealed == 0`.

</details>

### Q103. Pod stuck in `Init:0/1`; the `vault-agent-init` container logs `error authenticating: ... Code: 403 ... * permission denied` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Kubernetes auth role does not match the pod: wrong `bound_service_account_names`, wrong `bound_service_account_namespaces`, a wrong `vault.hashicorp.com/role` annotation, or a role that has no policy attached.

**Diagnose →**
```bash
kubectl get pod <pod> -o jsonpath='{.spec.serviceAccountName}{"\n"}{.metadata.annotations}'
vault read auth/kubernetes/role/api
```

**Fix →**
```bash
vault write auth/kubernetes/role/api \
  bound_service_account_names=api \
  bound_service_account_namespaces=prod \
  policies=api-read ttl=1h
```

**Prevent →** Manage roles in Terraform next to the Deployment's ServiceAccount, and give each workload its own SA. Never use `default`.

</details>

### Q104. After a cluster upgrade, Vault Kubernetes auth fails for every pod with `error validating token: invalid issuer (iss) claim` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bound projected tokens carry an issuer such as `https://oidc.eks.<region>.amazonaws.com/id/...` or `https://kubernetes.default.svc.cluster.local`, and it does not match the `issuer` set in the auth config. A related failure: the `token_reviewer_jwt` was a legacy SA token secret that no longer exists.

**Diagnose →**
```bash
kubectl create token api -n prod | cut -d. -f2 | base64 -d 2>/dev/null | jq .iss
vault read auth/kubernetes/config
```

**Fix →** When Vault runs in-cluster, drop `token_reviewer_jwt` so Vault uses its own pod token, and let it validate through the TokenReview API:
```bash
vault write auth/kubernetes/config \
  kubernetes_host=https://kubernetes.default.svc:443 \
  disable_local_ca_jwt=false
```
For an external Vault, give it a long-lived reviewer token from a `kubernetes.io/service-account-token` Secret and set `issuer` correctly.

**Prevent →** Include a Vault login smoke test in the cluster-upgrade runbook.

</details>

### Q105. Pods annotated with `vault.hashicorp.com/agent-inject: true` start without a Vault sidecar, and the app fails on a missing file 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The injector's mutating webhook is not running or does not select the namespace. Or the annotation sits on the Deployment metadata instead of the pod template. Or the value is not a quoted string. Because the webhook uses `failurePolicy: Ignore`, the pod is admitted unmutated.

**Diagnose →**
```bash
kubectl get mutatingwebhookconfiguration vault-agent-injector-cfg -o yaml | yq '.webhooks[].namespaceSelector'
kubectl logs -n vault deploy/vault-agent-injector | tail
kubectl get deploy api -o yaml | yq '.spec.template.metadata.annotations'
```

**Fix →** Put the annotations under `spec.template.metadata.annotations`, write `"true"` with quotes, then roll out the Deployment again. Mutations only happen at pod creation.

**Prevent →** Add a policy that fails pods carrying the inject annotation but no `vault-agent` container.

</details>

### Q106. ExternalSecret shows `SecretSyncedError` with `AccessDeniedException: ... is not authorized to perform: secretsmanager:GetSecretValue` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ESO controller (or the SA named in the SecretStore's `auth.jwt.serviceAccountRef`) has no IRSA role, or the role's policy scopes the wrong ARN. Secret ARNs end in a random 6-character suffix. If the secret uses a customer-managed KMS key, the role also needs `kms:Decrypt`.

**Diagnose →**
```bash
kubectl describe externalsecret db -n prod
kubectl get sa -n external-secrets external-secrets -o yaml | grep role-arn
aws iam simulate-principal-policy --policy-source-arn <role> \
  --action-names secretsmanager:GetSecretValue --resource-arns <secret-arn>
```

**Fix →** Grant `secretsmanager:GetSecretValue` on `arn:aws:secretsmanager:<r>:<acct>:secret:prod/api/*` and `kms:Decrypt` on the CMK. Add the role to the key policy if the policy does not delegate to IAM.

**Prevent →** Use one SecretStore per namespace with its own narrowly scoped role, not one cluster-wide all-powerful role.

</details>

### Q107. ExternalSecret reports `could not get provider client: SecretStore "aws-sm" is not ready` or `SecretStore.external-secrets.io "aws-sm" not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ExternalSecret references `kind: SecretStore` (namespaced) while the store is a `ClusterSecretStore`, or the reverse. Another cause: the store itself failed validation (bad region, bad auth).

**Diagnose →**
```bash
kubectl get secretstore,clustersecretstore -A
kubectl describe clustersecretstore aws-sm      # Conditions: Ready=False, reason
```

**Fix →**
```yaml
spec:
  secretStoreRef:
    name: aws-sm
    kind: ClusterSecretStore
```
Then fix whatever the store's status condition says.

**Prevent →** Template ExternalSecrets in a Helm library chart so `kind` and `name` are not hand-typed.

</details>

### Q108. The DB password was rotated in AWS Secrets Manager, the K8s Secret shows the new value, but the pods still use the old one 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app reads the secret from environment variables. Env vars are fixed when the container starts and never refresh. Volume-mounted secrets do refresh, but only after the kubelet sync delay, and only if the app re-reads the file.

**Diagnose →**
```bash
kubectl get externalsecret db -o jsonpath='{.status.refreshTime}'
kubectl exec deploy/api -- printenv DB_PASSWORD | sha256sum
kubectl get secret db -o jsonpath='{.data.password}' | base64 -d | sha256sum
```

**Fix →** Roll the Deployment automatically when the Secret changes, for example with Stakater Reloader (`reloader.stakater.com/auto: "true"`), or mount the secret as a file and have the app watch it.

**Prevent →** Rotate in two phases: the old and new credentials both stay valid until every pod has restarted.

</details>

### Q109. Two ExternalSecrets write to the same target Secret and its keys flip every refresh cycle 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both use `creationPolicy: Owner`, so each one replaces the full Secret data with its own keys.

**Diagnose →**
```bash
kubectl get externalsecret -n prod -o custom-columns=NAME:.metadata.name,TARGET:.spec.target.name,POLICY:.spec.target.creationPolicy
kubectl get secret app -o yaml | yq '.metadata.ownerReferences'
```

**Fix →** Merge the data into one ExternalSecret with several `data` entries, or set the second one to `creationPolicy: Merge` so it only adds its keys to a Secret owned by the first.

**Prevent →** Use the convention of one target Secret per ExternalSecret, enforced by a policy that rejects duplicate `spec.target.name` values within a namespace.

</details>

### Q110. `sops -d secrets.enc.yaml` fails with `Failed to get the data key required to decrypt the SOPS file` and `no identity matched any of the recipients` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your age private key is not where SOPS looks for it, or the file was encrypted to recipients that do not include your public key.

**Diagnose →**
```bash
yq '.sops.age[].recipient' secrets.enc.yaml
age-keygen -y ~/.config/sops/age/keys.txt        # your public key
echo "$SOPS_AGE_KEY_FILE"
```

**Fix →** Point SOPS at your key with `export SOPS_AGE_KEY_FILE=~/keys/age.txt`. On Linux the default path is `~/.config/sops/age/keys.txt`. If your recipient is missing from the file, someone who can decrypt must add you (see Q112).

**Prevent →** Keep a team recipient list in `.sops.yaml` and document key setup in the repo README.

</details>

### Q111. A new engineer runs `sops -d` on a KMS-encrypted file and gets `AccessDeniedException: ... is not authorized to perform: kms:Decrypt` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Their IAM role or SSO permission set has no `kms:Decrypt` on that key, the key policy does not delegate to IAM, or they are logged into the wrong account or profile.

**Diagnose →**
```bash
aws sts get-caller-identity
yq '.sops.kms[].arn' secrets.enc.yaml
aws kms get-key-policy --key-id <arn> --policy-name default
```

**Fix →** Grant `kms:Decrypt` (and `kms:Encrypt`/`kms:GenerateDataKey` for editors) to the team role. Or set `AWS_PROFILE` to the profile mapped to the right account.

**Prevent →** Grant KMS access to groups or roles, never to individual users, and keep a separate key per environment so dev engineers cannot decrypt prod.

</details>

### Q112. You added a teammate's age key to `.sops.yaml`, but they still cannot decrypt the existing files 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `.sops.yaml` creation rules only apply when a file is encrypted. Existing files keep the data key wrapped for the old recipients.

**Diagnose →** `yq '.sops.age[].recipient' file.enc.yaml` does not list the new key.

**Fix →** Someone who can decrypt re-wraps the data key:
```bash
find . -name '*.enc.yaml' -exec sops updatekeys -y {} \;
git commit -am "sops: add recipient for new team member"
```
When someone **leaves**, `updatekeys` is not enough because they may already hold the data key. Run `sops rotate -i` and rotate the actual secret values too.

**Prevent →** Make recipient changes PR-driven with a script that runs `updatekeys` across the repo.

</details>

### Q113. Flux Kustomization fails with `decryption failed for 'db-secret': ... failed to get the data key required to decrypt the SOPS file` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `spec.decryption` is missing from the Kustomization, the referenced secret is in the wrong namespace, or the age key in it is stored under a key name that does not end in `.agekey`.

**Diagnose →**
```bash
flux get kustomizations -A
kubectl get kustomization apps -n flux-system -o yaml | yq '.spec.decryption'
kubectl get secret sops-age -n flux-system -o jsonpath='{.data}' | jq 'keys'
```

**Fix →**
```bash
kubectl create secret generic sops-age -n flux-system \
  --from-file=age.agekey=./age.agekey
```
```yaml
spec:
  decryption:
    provider: sops
    secretRef: { name: sops-age }
```

**Prevent →** Bootstrap the decryption secret as part of cluster provisioning, and keep its private key in a cloud secret manager.

</details>

### Q114. After rebuilding the cluster, every SealedSecret fails with `no key could decrypt secret` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The new sealed-secrets controller generated a fresh key pair. The old private key that sealed the Git contents was lost with the old cluster.

**Diagnose →**
```bash
kubectl logs -n kube-system deploy/sealed-secrets-controller | grep -i decrypt
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key
```

**Fix →** Restore the old key secrets from backup into `kube-system` and restart the controller. With no backup, you have to re-seal every secret from the source values with `kubeseal --fetch-cert` against the new controller.

**Prevent →** Back up the controller keys right after install and after each key renewal:
```bash
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-keys.yaml
```
Store that file encrypted in a secret manager, never in Git.

</details>

### Q115. A SealedSecret works in `staging` but fails in `prod` with `no key could decrypt secret` after being copied there 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The default scope is `strict`. The ciphertext is bound to the original name and namespace, so changing either breaks decryption, even on the same controller.

**Diagnose →** `kubectl get sealedsecret db -n prod -o yaml | yq '.metadata.annotations'` shows no cluster-wide or namespace-wide annotation.

**Fix →** Re-seal for the target namespace:
```bash
kubectl create secret generic db --from-literal=password=... --dry-run=client -o yaml \
  | kubeseal --namespace prod --format yaml > prod/db-sealed.yaml
```

**Prevent →** Seal per environment from source values. Use `--scope cluster-wide` only when you deliberately want the ciphertext portable.

</details>

### Q116. An app using Vault dynamic database credentials starts failing with `password authentication failed for user "v-kubernetes-api-x7f..."` after a few hours 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The lease hit its `max_ttl` and Vault revoked the DB user, but the app only read the credentials once at startup. Or the agent renewed the lease but the app's connection pool kept stale credentials.

**Diagnose →**
```bash
vault read database/roles/api            # default_ttl, max_ttl
vault list sys/leases/lookup/database/creds/api
```

**Fix →** Have Vault Agent re-render the credentials file and signal the app (`exec` with `restart_on_secret_changes`), or let the app re-read credentials and rebuild the pool. Make `max_ttl` longer than the longest pod lifetime, or restart pods well before it.

**Prevent →** Load-test credential rotation in staging with a short `max_ttl` such as 10 minutes.

</details>

### Q117. Vault becomes slow, requests time out with `context deadline exceeded`, and `vault.expire.num_leases` is at 4 million 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A lease explosion. Some client logs in or requests dynamic credentials on every request (a CronJob, Lambda or app with no token caching). Each request creates a token or lease that lives until its TTL expires.

**Diagnose →**
```bash
vault read sys/internal/counters/activity
vault list sys/leases/lookup/auth/kubernetes/login | wc -l
vault read sys/metrics -format=json | jq '.Gauges[] | select(.Name|test("num_leases"))'
```

**Fix →** Find the offending role in the audit log and fix the client to reuse its token (Vault Agent auto-auth with caching). Shorten that role's `token_ttl`. Revoke the excess with `vault lease revoke -prefix auth/kubernetes/login`, carefully and in batches.

**Prevent →** Set lease count quotas (`vault write sys/quotas/lease-count/k8s max_leases=50000 path=auth/kubernetes`) and alert on lease growth.

</details>

### Q118. Vault writes fail with `local node not active but active cluster node not found`; `vault operator raft list-peers` shows 2 of 5 voters 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Raft lost quorum. 3 of 5 nodes are gone (a zone outage, or PVCs deleted during a node pool replacement), so no leader can be elected.

**Diagnose →**
```bash
kubectl get pods,pvc -n vault
kubectl logs -n vault vault-0 | grep -i -E "raft|election"
```

**Fix →** If the lost nodes can come back with their data, bring them back. If not, do manual recovery: stop the remaining nodes, write `raft/peers.json` on a surviving node listing only the healthy peers, start it, unseal it, and re-join the others with `vault operator raft join`. As a last resort, restore from a `vault operator raft snapshot save` backup.

**Prevent →** Spread voters across 3 zones, use `PodDisruptionBudget` and pod anti-affinity, enable Autopilot (`vault operator raft autopilot set-config -cleanup-dead-servers=true`), and take scheduled snapshots.

</details>

### Q119. A GitHub Actions secret appears in plain text in the job log even though secrets are supposed to be masked 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Masking only matches the exact stored string. A value that was base64-encoded, URL-encoded, split into lines (a multi-line JSON key) or derived from the secret is not masked.

**Diagnose →** Search the log for the transformed form, and check for steps that do `echo $SECRET | base64`, `cat key.json` or `set -x`.

**Fix →** Delete the log run (Actions UI, or `gh run delete <id>`) and rotate the secret. Mask derived values before using them:
```bash
TOKEN=$(echo "$RAW" | base64 -d)
echo "::add-mask::$TOKEN"
```

**Prevent →** Never print secrets, avoid `set -x` in steps that handle them, and store structured credentials as a single-line value.

</details>

### Q120. A security review finds the RDS master password in plain text in the Terraform state file in S3 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Terraform stores every attribute in state, including `sensitive` ones. `sensitive = true` only hides values from CLI output.

**Diagnose →**
```bash
terraform state pull | jq '.resources[] | select(.type=="aws_db_instance") | .instances[].attributes.password'
```

**Fix →** Rotate the password. Then remove it from Terraform entirely: `manage_master_user_password = true` makes RDS generate and store it in Secrets Manager. On Terraform 1.11+ with a recent provider you can also use write-only arguments such as `password_wo`, which are never persisted.

**Prevent →** Encrypt the state bucket with KMS, restrict `s3:GetObject` on state to CI roles, enable access logging, and treat state as a secret.

</details>

### Q121. Developers with only the namespace `edit` role can read production Secrets through `kubectl get secret -o yaml` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The built-in `edit` (and `admin`) ClusterRoles include `get/list/watch` on Secrets. Even without them, any role that can create pods can mount any Secret in the namespace and read it.

**Diagnose →**
```bash
kubectl auth can-i get secrets -n prod --as=jane@acme.com
kubectl auth can-i create pods -n prod --as=jane@acme.com
kubectl get rolebinding -n prod -o wide
```

**Fix →** Give developers `view` (which excludes Secrets) in prod, plus a custom role for what they really need. Deploys should go through CI or GitOps, not humans holding `edit`.

**Prevent →** Keep real credentials in Vault or a cloud secret manager with per-workload access, and audit `get`/`list` on secrets with an audit policy at `Metadata` level.

</details>

### Q122. You enabled etcd encryption at rest, but `etcdctl get /registry/secrets/prod/db` still returns readable plaintext 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Encryption applies only on write. Secrets created before the change stay in plaintext until they are rewritten. Another cause: `identity {}` is listed first in the `EncryptionConfiguration` providers, so new writes are also unencrypted.

**Diagnose →**
```bash
ETCDCTL_API=3 etcdctl get /registry/secrets/prod/db | hexdump -C | head
# Encrypted values start with k8s:enc:aescbc:v1: or k8s:enc:kms:v2:
```

**Fix →** Put the real provider (`kms` v2 preferred, or `aescbc`/`secretbox`) first and keep `identity {}` last only for reading. Restart the API servers, then rewrite every Secret:
```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

**Prevent →** On managed clusters, turn on envelope encryption (EKS KMS secrets encryption, GKE application-layer secrets encryption) at creation, and verify it with an etcd read in a test.

</details>

### Q123. Vault Agent renders the secrets file, but the values in it are `<no value>` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** KV v2 nests the payload one level deeper. The template uses `.Data.password` instead of `.Data.data.password`, or the path is missing `data/`.

**Diagnose →** `vault kv get -format=json secret/app/db | jq .data` shows `data` and `metadata` keys.

**Fix →**
```yaml
vault.hashicorp.com/agent-inject-template-db: |
  {{- with secret "secret/data/app/db" -}}
  DB_PASSWORD={{ .Data.data.password }}
  {{- end }}
```

**Prevent →** Add a startup check that fails fast if any config value equals `<no value>` or is empty. Silent empty passwords are worse than a crash.

</details>

### Q124. A long-running service gets `permission denied` from Vault exactly 32 days after deployment 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The service authenticated once and kept renewing its token until it reached the default `max_lease_ttl` of 768h (32 days). Renewal cannot extend past max TTL.

**Diagnose →**
```bash
vault token lookup <accessor-or-token>   # look at creation_time, expire_time, explicit_max_ttl
vault read sys/mounts/auth/kubernetes/tune
```

**Fix →** Have the client log in again when renewal fails. Vault Agent auto-auth and the official SDK lifetime watchers do this. For services that cannot log in again, use a periodic token (`period=24h`), which can be renewed indefinitely.

**Prevent →** Use Vault Agent or a CSI provider instead of hand-written token handling, and alert on Vault 403s per service.

</details>

### Q125. A Vault root token is found pasted in a Slack channel 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone generated a root token for setup or break-glass work and never revoked it.

**Diagnose →**
```bash
vault token lookup <token>               # policies: [root], creation_time
# Search audit logs for the token's HMAC
vault write sys/audit-hash/file input=<token>
grep <hmac> /vault/audit/audit.log
```

**Fix →** Revoke it immediately with `vault token revoke <token>`. Review everything it touched in the audit log, and rotate any secrets it read or policies and auth methods it changed. Delete the Slack message and check Slack's export or retention settings.

**Prevent →** No standing root tokens. Generate one only when needed with `vault operator generate-root` under the unseal-key quorum, and revoke it in the same session. Admins should use OIDC login with named policies.

</details>

## Cloud identity: IRSA, Workload Identity and CI OIDC

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. A pod configured for IRSA runs `aws sts get-caller-identity` and gets the node role (`eks-node-group-role`) instead of its own role 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ServiceAccount has no `eks.amazonaws.com/role-arn` annotation, the pod uses a different SA, or the pod was created before the annotation was added. The pod identity webhook only injects at pod creation. Without the injected env vars, the SDK falls back to the node's instance profile.

**Diagnose →**
```bash
kubectl get sa api -n prod -o yaml | grep role-arn
kubectl exec deploy/api -- env | grep -E 'AWS_ROLE_ARN|AWS_WEB_IDENTITY_TOKEN_FILE'
```

**Fix →** Annotate the SA, then run `kubectl rollout restart deploy/api`.

**Prevent →** Block pod access to IMDS (Q130) so a missing annotation fails loudly instead of silently borrowing node permissions.

</details>

### Q127. `An error occurred (AccessDenied) when calling the AssumeRoleWithWebIdentity operation: Not authorized to perform sts:AssumeRoleWithWebIdentity` from an IRSA pod 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role's trust policy does not match the token. The `sub` condition names a different namespace or SA, the OIDC provider ID belongs to another cluster, or `aud` is not `sts.amazonaws.com`.

**Diagnose →**
```bash
kubectl exec deploy/api -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token \
  | cut -d. -f2 | base64 -d 2>/dev/null | jq '{iss,sub,aud}'
aws iam get-role --role-name api-prod --query Role.AssumeRolePolicyDocument
```

**Fix →**
```json
"Condition": { "StringEquals": {
  "oidc.eks.ap-south-1.amazonaws.com/id/ABC123:sub": "system:serviceaccount:prod:api",
  "oidc.eks.ap-south-1.amazonaws.com/id/ABC123:aud": "sts.amazonaws.com"
}}
```

**Prevent →** Generate trust policies from the SA name and namespace in a Terraform module, never by hand.

</details>

### Q128. `InvalidIdentityToken: No OpenIDConnect provider found in your account for https://oidc.eks.ap-south-1.amazonaws.com/id/ABC123` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cluster's OIDC issuer was never registered as an IAM identity provider in this account, or it was registered in a different account than the role.

**Diagnose →**
```bash
aws eks describe-cluster --name prod --query cluster.identity.oidc.issuer --output text
aws iam list-open-id-connect-providers
```

**Fix →** `eksctl utils associate-iam-oidc-provider --cluster prod --approve`, or an `aws_iam_openid_connect_provider` resource with `client_id_list = ["sts.amazonaws.com"]`.

**Prevent →** Create the OIDC provider in the same Terraform stack as the cluster.

</details>

### Q129. IRSA env vars are present in the pod, but one legacy service still uses the node role 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The service uses an old AWS SDK that predates web-identity support in the default credential chain, or the code builds credentials explicitly (for example `InstanceProfileCredentialsProvider`), or it runs as a user that cannot read the token file.

**Diagnose →**
```bash
kubectl exec deploy/legacy -- ls -l /var/run/secrets/eks.amazonaws.com/serviceaccount/token
kubectl exec deploy/legacy -- id
# check the SDK version in the dependency manifest
```

**Fix →** Upgrade the SDK and use the default provider chain. If the container runs as non-root, set `securityContext.fsGroup` so the token file is readable.

**Prevent →** Keep a minimum SDK version in your base images, and turn on IMDS blocking so fallback becomes an error.

</details>

### Q130. Pods without IRSA can still call AWS with the node's permissions through `169.254.169.254` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IMDSv1 is enabled, or IMDSv2's hop limit is 2 or more, which lets containers in pod network namespaces reach the metadata service.

**Diagnose →**
```bash
kubectl run t --rm -it --image=curlimages/curl -- \
  sh -c 'curl -s -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: 60" http://169.254.169.254/latest/api/token'
aws ec2 describe-instances --instance-ids <id> --query 'Reservations[].Instances[].MetadataOptions'
```

**Fix →** In the launch template set `http_tokens = "required"` and `http_put_response_hop_limit = 1`. Pods on host networking can still reach IMDS, so also add a NetworkPolicy egress deny to `169.254.169.254/32`.

**Prevent →** Keep node roles minimal (ECR pull, CNI, SSM), give every workload IRSA or Pod Identity, and enforce the metadata options with an SCP or AWS Config rule.

</details>

### Q131. With EKS Pod Identity, the pod gets `Unable to locate credentials` even though a pod identity association exists 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `eks-pod-identity-agent` add-on is not installed or not running on that node, the pod predates the association, or the role trust policy is written for IRSA instead of Pod Identity.

**Diagnose →**
```bash
aws eks list-pod-identity-associations --cluster-name prod --namespace prod
kubectl get ds -n kube-system eks-pod-identity-agent
kubectl exec deploy/api -- env | grep AWS_CONTAINER_CREDENTIALS_FULL_URI
```

**Fix →** Install the add-on and restart the pod. The trust policy must be:
```json
{ "Effect": "Allow",
  "Principal": { "Service": "pods.eks.amazonaws.com" },
  "Action": ["sts:AssumeRole", "sts:TagSession"] }
```

**Prevent →** Pick one mechanism per cluster (Pod Identity for new EKS clusters) and manage associations in Terraform.

</details>

### Q132. `aws-actions/configure-aws-credentials` fails with `Credentials could not be loaded, please check your action inputs: Could not load credentials from any providers` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow did not request an OIDC token. Without `permissions: id-token: write`, GitHub never sets `ACTIONS_ID_TOKEN_REQUEST_URL`. Declaring any `permissions:` block also resets every unlisted permission to none.

**Diagnose →** Check the workflow or job `permissions:`. The action log usually hints at the missing `id-token` permission.

**Fix →**
```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::111122223333:role/gha-deploy
      aws-region: ap-south-1
```

**Prevent →** Use a reusable workflow template with the right permissions, and pin the action by SHA.

</details>

### Q133. GitHub OIDC to AWS works on pushes to `main` but fails with `Not authorized to perform sts:AssumeRoleWithWebIdentity` in the deploy job that uses `environment: prod` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `sub` claim changes format depending on context. On a branch push it is `repo:acme/api:ref:refs/heads/main`. When the job targets an environment it becomes `repo:acme/api:environment:prod`. Pull requests use `repo:acme/api:pull_request`.

**Diagnose →** Print the claims once in a debug job:
```bash
curl -sH "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
  "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq -r .value | cut -d. -f2 | base64 -d | jq .sub
```

**Fix →** Trust `repo:acme/api:environment:prod` for the prod role, and protect the `prod` environment with required reviewers.

**Prevent →** One role per environment, each trusting exactly one `sub` value. Use environments as the gate rather than branch names.

</details>

### Q134. An audit finds an IAM role whose GitHub OIDC trust policy checks only `token.actions.githubusercontent.com:aud = sts.amazonaws.com` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone copied a minimal example. With no `sub` condition, **any GitHub Actions workflow in any repository in the world** can assume the role, because every workflow can get a token with that audience from the same issuer.

**Diagnose →**
```bash
aws iam get-role --role-name gha-deploy --query 'Role.AssumeRolePolicyDocument.Statement[].Condition'
# CloudTrail: check who actually assumed it
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
  --query 'Events[].CloudTrailEvent' --output text | jq -r '.userIdentity.userName // .requestParameters.roleSessionName'
```

**Fix →** Add a strict `sub` condition right away (`StringEquals` on `repo:acme/api:environment:prod`). Review CloudTrail for the `sub` values that have used the role, and treat unknown ones as an incident.

**Prevent →** An IAM Access Analyzer custom policy check or a Checkov/OPA rule in CI that rejects GitHub OIDC trust policies without a `sub` condition, or with `repo:*` wildcards.

</details>

### Q135. GitLab CI to AWS OIDC fails with `InvalidIdentityToken: Incorrect token audience` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `aud` in the job's `id_tokens` does not match any client ID on the IAM OIDC provider for `gitlab.com` (or your self-managed host).

**Diagnose →**
```bash
aws iam get-open-id-connect-provider --open-id-connect-provider-arn <arn> --query ClientIDList
```
Compare that list with the `aud:` in `.gitlab-ci.yml`.

**Fix →**
```yaml
deploy:
  id_tokens:
    AWS_TOKEN:
      aud: https://gitlab.com
  script:
    - >
      aws sts assume-role-with-web-identity --role-arn "$ROLE_ARN"
      --role-session-name "gl-$CI_JOB_ID" --web-identity-token "$AWS_TOKEN"
```
Make the provider's client ID list contain the same value.

**Prevent →** Restrict the trust policy on `gitlab.com:sub` (for example `project_path:acme/api:ref_type:branch:ref:main`) and keep the job template central.

</details>

### Q136. A GKE pod gets `403 ... does not have storage.objects.get access`, and the logs show it is using the node's Compute Engine default service account 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Workload Identity is not enabled on the node pool (metadata mode is not `GKE_METADATA`), or the KSA is not mapped to an IAM identity.

**Diagnose →**
```bash
gcloud container node-pools describe default --cluster prod --region asia-south1 \
  --format='value(config.workloadMetadataConfig.mode)'
kubectl run t --rm -it --image=google/cloud-sdk:slim --overrides='{"spec":{"serviceAccountName":"api"}}' -n prod \
  -- curl -s -H 'Metadata-Flavor: Google' http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email
```

**Fix →** Update the node pool with `--workload-metadata=GKE_METADATA`, then grant the permission either to the KSA principal directly (`principal://iam.googleapis.com/projects/<num>/locations/global/workloadIdentityPools/<proj>.svc.id.goog/subject/ns/prod/sa/api`) or to a GSA that the KSA impersonates.

**Prevent →** Create clusters with Workload Identity on, and remove broad roles from the default compute SA.

</details>

### Q137. GKE pod fails with `Unable to generate access token; IAM returned 403 Forbidden: Permission 'iam.serviceAccounts.getAccessToken' denied on resource` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The KSA is annotated with `iam.gke.io/gcp-service-account`, but the GSA lacks the `roles/iam.workloadIdentityUser` binding for that exact KSA, or the namespace or name in the member string is wrong.

**Diagnose →**
```bash
kubectl get sa api -n prod -o jsonpath='{.metadata.annotations.iam\.gke\.io/gcp-service-account}'
gcloud iam service-accounts get-iam-policy api@proj.iam.gserviceaccount.com
```

**Fix →**
```bash
gcloud iam service-accounts add-iam-policy-binding api@proj.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:proj.svc.id.goog[prod/api]"
```
Expect a few minutes of propagation delay.

**Prevent →** Manage the KSA, GSA and binding in one Terraform module.

</details>

### Q138. `google-github-actions/auth` fails with `The given credential is rejected by the attribute condition` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Workload Identity Pool provider has an `attribute-condition` (as it should) that this workflow's claims do not satisfy, such as a different repository owner, a fork, or a non-main ref.

**Diagnose →**
```bash
gcloud iam workload-identity-pools providers describe github \
  --location=global --workload-identity-pool=ci --format='value(attributeCondition,attributeMapping)'
```

**Fix →** If the run is legitimate, adjust the condition, for example `assertion.repository_owner == 'acme' && assertion.ref == 'refs/heads/main'`. Then grant `roles/iam.workloadIdentityUser` (or direct resource roles) to `principalSet://.../attribute.repository/acme/api`.

**Prevent →** Always keep an attribute condition, because a provider without one trusts every GitHub repo. Scope bindings to specific repositories.

</details>

### Q139. After recreating an AKS cluster, pods fail with `AADSTS700211: No matching federated identity record found for presented assertion issuer` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every AKS cluster has its own OIDC issuer URL. The federated identity credentials on the managed identity still point at the old cluster's issuer.

**Diagnose →**
```bash
az aks show -g rg -n prod --query oidcIssuerProfile.issuerUrl -o tsv
az identity federated-credential list -g rg --identity-name api-id -o table
```

**Fix →**
```bash
az identity federated-credential create -g rg --identity-name api-id --name prod-api \
  --issuer "$(az aks show -g rg -n prod --query oidcIssuerProfile.issuerUrl -o tsv)" \
  --subject system:serviceaccount:prod:api --audiences api://AzureADTokenExchange
```

**Prevent →** Make federated credentials depend on the cluster's issuer output in Terraform, so a rebuild updates them. For blue/green clusters, add a credential for each.

</details>

### Q140. An AKS app using `DefaultAzureCredential` fails with `ManagedIdentityCredential authentication unavailable` / `WorkloadIdentityCredential authentication unavailable` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The pod is missing the label `azure.workload.identity/use: "true"`, so the webhook never injected `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` and `AZURE_FEDERATED_TOKEN_FILE`. The SA may also lack the `azure.workload.identity/client-id` annotation.

**Diagnose →**
```bash
kubectl get pod <pod> -o jsonpath='{.metadata.labels}'
kubectl exec <pod> -- env | grep AZURE_
```

**Fix →** Add the label to the pod template (not just the Deployment), annotate the SA with the identity's client ID, and restart the pods.

**Prevent →** Validate both with a Kyverno policy for namespaces that use workload identity.

</details>

### Q141. `azure/login` in GitHub Actions fails with `AADSTS700213: No matching federated identity record found for presented assertion subject 'repo:acme/api:environment:prod'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The federated credential on the app registration or managed identity was created for "Branch: main" (`repo:acme/api:ref:refs/heads/main`), but the job runs with `environment: prod`, which changes the subject.

**Diagnose →** Compare the error's subject with `az ad app federated-credential list --id <app-id>`.

**Fix →** Add a federated credential with subject `repo:acme/api:environment:prod`, and keep `permissions: id-token: write` on the job.

**Prevent →** Document the subject format per job type in your CI template and create the credentials from it.

</details>

### Q142. Cross-account deploy fails with `User: arn:aws:sts::111111111111:assumed-role/ci/... is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::222222222222:role/deploy` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A cross-account assume needs both sides. The target role's trust policy must trust the source principal, and the source role must have `sts:AssumeRole` on the target ARN. One side is missing, or an `sts:ExternalId` condition is not met.

**Diagnose →**
```bash
aws iam get-role --role-name deploy --query Role.AssumeRolePolicyDocument   # run in 222...
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::111111111111:role/ci \
  --action-names sts:AssumeRole --resource-arns arn:aws:iam::222222222222:role/deploy
```

**Fix →** Trust the specific role ARN (`arn:aws:iam::111111111111:role/ci`), not the account root. Add the `sts:AssumeRole` allow on the source side.

**Prevent →** Use a deploy-role module per target account and test assumption in a pipeline smoke step.

</details>

### Q143. A role with `AdministratorAccess` gets `AccessDenied ... with an explicit deny in a service control policy` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An AWS Organizations SCP denies the action in that account or OU, for example a region restriction, a ban on `iam:CreateUser`, or protection for CloudTrail. SCPs cap what any identity in the account can do, admins included. A permissions boundary has a similar effect (`... with an explicit deny in a permissions boundary`).

**Diagnose →** The error message names the policy type. Then:
```bash
aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY
aws iam get-role --role-name ci --query Role.PermissionsBoundary
```

**Fix →** If the action is legitimate, request an SCP exception (often a condition on `aws:PrincipalArn` for a break-glass or pipeline role). Do not try to work around the guardrail.

**Prevent →** Publish your SCP guardrails to teams, and test pipelines in an account under the same OU as prod.

</details>

### Q144. An IRSA role has `s3:GetObject` on the bucket, the policy simulator says allowed, yet the pod gets `AccessDenied` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A deny or a missing grant in another policy layer. The bucket policy may deny unless `aws:SourceVpce` matches, the object may be SSE-KMS encrypted and the role lacks `kms:Decrypt`, a VPC endpoint policy may restrict buckets, or the object may be owned by another account with ACLs.

**Diagnose →**
```bash
aws s3api get-bucket-policy --bucket data-prod | jq -r .Policy | jq .
aws s3api head-object --bucket data-prod --key path/file   # ServerSideEncryption, SSEKMSKeyId
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.ap-south-1.s3 --query 'VpcEndpoints[].PolicyDocument'
```
CloudTrail's `errorMessage` for the call often names the layer.

**Fix →** Grant `kms:Decrypt` on the key (and add the role to the key policy if needed). Align the bucket and endpoint policies, and set Object Ownership to `BucketOwnerEnforced`.

**Prevent →** Test access paths end to end from inside the cluster in CI, not just in the simulator, which does not evaluate every layer.

</details>

### Q145. A 2-hour Terraform apply from GitHub Actions dies midway with `ExpiredToken: The security token included in the request is expired` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `configure-aws-credentials` defaults to a 1-hour session. If the job chains into another role, role chaining is hard-capped at 1 hour regardless of settings.

**Diagnose →**
```bash
aws iam get-role --role-name gha-deploy --query Role.MaxSessionDuration
```
Check whether the Terraform provider uses `assume_role`, which is chaining.

**Fix →** Raise the role's `MaxSessionDuration` (up to 43200), set `role-duration-seconds: 10800` in the action, and assume the target role directly from OIDC instead of chaining.

**Prevent →** Split huge applies into smaller state files. Long-running applies are a reliability risk on their own.

</details>

### Q146. A CI pipeline creates a ServiceAccount for deploys but `kubectl get secret` shows no token secret for it on Kubernetes 1.29 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Since 1.24, Kubernetes no longer auto-creates long-lived `kubernetes.io/service-account-token` Secrets. Tokens are short-lived and come from the TokenRequest API.

**Diagnose →** `kubectl get sa deployer -o yaml` shows no `secrets:` entry.

**Fix →** Prefer a short-lived token:
```bash
kubectl create token deployer -n ci --duration=1h
```
If you truly need a long-lived one, create a Secret of type `kubernetes.io/service-account-token` with the annotation `kubernetes.io/service-account.name: deployer`.

**Prevent →** Better still, authenticate CI through cloud identity (EKS access entries, GKE IAM) or GitOps pull, so no Kubernetes token is stored in CI at all.

</details>

### Q147. A kubeconfig with a static cluster-admin ServiceAccount token is found committed in a public repo 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A long-lived SA token secret was created for CI and copied into a file for convenience.

**Diagnose →**
```bash
echo "$TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq   # SA name, namespace, secret name
kubectl get clusterrolebinding -o wide | grep <sa>
```
Search the audit logs for that SA's username (`system:serviceaccount:<ns>:<sa>`) since the commit date.

**Fix →** Delete the token Secret (this invalidates the token at once), remove the cluster-admin binding, review audit logs for actions taken, and rotate any secrets the SA could read. Also purge the file from Git history, though rotation is what actually protects you. If the API endpoint is public, restrict it.

**Prevent →** No long-lived tokens. Use OIDC-based CI auth, a private API endpoint or an authorized network list, and secret scanning with push protection.

</details>

### Q148. After switching an EKS cluster to `authenticationMode: API`, the CI role gets `error: You must be logged in to the server (Unauthorized)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CI role was only mapped in the `aws-auth` ConfigMap, which is ignored in `API` mode. It needs an access entry.

**Diagnose →**
```bash
aws eks describe-cluster --name prod --query cluster.accessConfig
aws eks list-access-entries --cluster-name prod
```

**Fix →**
```bash
aws eks create-access-entry --cluster-name prod --principal-arn arn:aws:iam::111122223333:role/gha-deploy
aws eks associate-access-policy --cluster-name prod --principal-arn arn:aws:iam::111122223333:role/gha-deploy \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=prod
```

**Prevent →** Migrate through `API_AND_CONFIG_MAP`, create the entries, verify, and only then switch to `API`.

</details>

### Q149. IRSA works in the blue cluster but the same Deployment in the new green cluster fails with `Not authorized to perform sts:AssumeRoleWithWebIdentity` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Each EKS cluster has its own OIDC issuer. The role trusts only the blue cluster's provider, and the green cluster's provider may not even be registered yet.

**Diagnose →**
```bash
for c in blue green; do aws eks describe-cluster --name $c --query cluster.identity.oidc.issuer --output text; done
aws iam get-role --role-name api-prod --query 'Role.AssumeRolePolicyDocument.Statement[].Principal'
```

**Fix →** Register the green provider and add a second statement for it to the trust policy (same `sub`). Remove the blue statement after cutover.

**Prevent →** Parameterize trust policies with a list of cluster issuers, or move to EKS Pod Identity, where the trust is on `pods.eks.amazonaws.com` and associations are per cluster.

</details>

### Q150. Pods in a private EKS cluster hang and then fail with `dial tcp 52.94.x.x:443: i/o timeout` when the SDK calls STS 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The nodes have no NAT route, and there is no STS VPC endpoint, or the SDK targets the global endpoint `sts.amazonaws.com`, which the regional interface endpoint does not serve.

**Diagnose →**
```bash
kubectl exec deploy/api -- env | grep AWS_STS_REGIONAL_ENDPOINTS
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.ap-south-1.sts
kubectl exec deploy/api -- nslookup sts.ap-south-1.amazonaws.com
```

**Fix →** Create an interface endpoint for `com.amazonaws.<region>.sts` with private DNS enabled, and set `AWS_STS_REGIONAL_ENDPOINTS=regional` (the IRSA webhook can inject this; current SDKs default to regional). Add endpoints for every other AWS API the app uses (ECR, S3 gateway, Secrets Manager, KMS).

**Prevent →** Keep a checklist of required VPC endpoints for private clusters and test with a pod that calls each service.

</details>

## DAST, runtime detection and compliance scans

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. The ZAP baseline scan of your React SPA finishes in 40 seconds with only header warnings and no real findings 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The traditional spider only follows `<a href>` links in HTML. A client-rendered SPA serves one near-empty `index.html`, so ZAP never discovers the routes or the API calls behind them. Baseline is also passive only.

**Diagnose →** Check the "URLs found" count in the report or the `-d` debug output. If it shows a handful of static assets, the crawl failed.

**Fix →** Enable the AJAX spider (`-j`), which drives a headless browser. Better still, test the API directly from its OpenAPI spec:
```bash
docker run --rm -v "$PWD:/zap/wrk" ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py -t https://staging.acme.com/openapi.json -f openapi -r api-report.html
```

**Prevent →** Treat API scanning (from the spec) as the main DAST signal for SPAs, and keep the baseline scan for headers and cookies.

</details>

### Q152. The authenticated ZAP full scan only ever tests the login page and `/logout`, and the report shows mostly `302` responses 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The session is lost early. The spider hits `/logout`, the token expires, or ZAP was never told how to authenticate or how to recognise a logged-out state.

**Diagnose →** In the ZAP session history, look for redirects to `/login` after the first few requests. Confirm whether the app uses cookies or a bearer token.

**Fix →** Define authentication in an Automation Framework plan: a context with the auth method (form, JSON or script), a verification strategy using a logged-in or logged-out indicator regex, and a test user. Exclude `.*logout.*` from the spider and scanner. For bearer-token APIs, inject the header with the `ZAP_AUTH_HEADER_VALUE` env var.

**Prevent →** Keep the ZAP automation YAML in the repo, and add a report check that fails if fewer than N authenticated URLs were scanned.

</details>

### Q153. `zap-baseline.py` fails the pipeline with exit code 2 even though the report has no high-risk alerts 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ZAP packaged scans exit with 1 when there are FAIL-level alerts, 2 when there are only WARN-level alerts, and 3 on errors. By default every rule is WARN, so any finding returns 2.

**Diagnose →** Read the summary at the end of the log, for example `FAIL-NEW: 0 WARN-NEW: 7`.

**Fix →** Generate a rules file, decide per rule, and run with it:
```bash
zap-baseline.py -t https://staging.acme.com -g gen.conf      # generate defaults
# edit to: 10038  FAIL  (Content Security Policy Header Not Set) ; 10027 IGNORE ...
zap-baseline.py -t https://staging.acme.com -c rules.tsv -I  # -I: don't fail on WARN
```

**Prevent →** Version the rules file and review IGNORE entries the way you review scanner suppressions.

</details>

### Q154. The DAST job reports nothing, and its log shows mostly `403` and `429` responses 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The WAF, CDN bot protection or app rate limiter is blocking the scanner, so it is testing block pages instead of your app.

**Diagnose →** `grep -c ' 403 \| 429 ' zap.log`, and check the WAF logs for the scanner's source IP.

**Fix →** In staging, allowlist the scanner's egress IP or a secret header in the WAF, or scan the origin directly from inside the VPC. Throttle the scanner (`-z "-config scanner.threadPerHost=2"`).

**Prevent →** Run DAST against an environment with the same app config as prod but with a known WAF bypass for the scanner. Test the WAF separately.

</details>

### Q155. An active DAST scan against staging created 12,000 junk records and sent password-reset emails to real customer addresses copied from prod 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The active scanner submits every form and fuzzes every endpoint. Staging held real PII and had live email delivery.

**Diagnose →** Match the scan window to the app's write logs and the email provider's send log.

**Fix →** Stop the scan, purge the generated data, and notify through your privacy process if real customers were emailed. Exclude destructive or side-effect paths (`/password-reset`, `/invite`, payment webhooks) in the ZAP context.

**Prevent →** No production PII in staging. Use a sandbox SMTP (Mailpit or a provider sandbox), give the scanner a dedicated test tenant, and run active scans against ephemeral environments that are destroyed afterwards.

</details>

### Q156. Nuclei runs in CI against staging and reports nothing for a known exposed `/.git/config` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The template directory is stale or empty on the runner, severity or tag filters exclude the template, or the target list has the wrong scheme or host.

**Diagnose →**
```bash
nuclei -version
nuclei -tl | grep -i git-config
nuclei -u https://staging.acme.com -id git-config -debug
```

**Fix →** Update templates (`nuclei -ut`) at the start of the job, or bake a pinned templates version into the image. Remove filters that are too narrow, such as `-severity critical`.

**Prevent →** Add a canary check: a test endpoint that must always be detected, so a silent scanner fails the job.

</details>

### Q157. Falco pods are in `CrashLoopBackOff` on new nodes after an OS upgrade, logging `Unable to load the driver` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Falco uses the kernel module or the legacy eBPF probe, and no prebuilt driver exists for the new kernel. Building it at runtime fails because the headers are missing.

**Diagnose →**
```bash
kubectl logs -n falco ds/falco -c falco-driver-loader
kubectl get nodes -o wide        # KERNEL-VERSION column
```

**Fix →** Switch to the modern eBPF probe, which is compiled into Falco and needs no per-kernel driver (kernel 5.8+ with BTF):
```bash
helm upgrade falco falcosecurity/falco -n falco --reuse-values --set driver.kind=modern_ebpf
```

**Prevent →** Test Falco on new node images in a canary node pool before rolling out OS upgrades.

</details>

### Q158. Falco floods Slack with `Terminal shell in container` alerts every few minutes, all from the same CI debug job 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A legitimate tool runs `kubectl exec ... sh`, or a readiness probe runs a shell. The default rule fires on any interactive shell.

**Diagnose →** Group the alerts by `k8s.ns.name`, `k8s.pod.name`, `proc.pname` and `user.name` in Falcosidekick or your SIEM.

**Fix →** Add a narrow exception in a custom rules file instead of editing the upstream rules:
```yaml
- rule: Terminal shell in container
  exceptions:
    - name: ci_debug
      fields: [k8s.ns.name, container.image.repository]
      values:
        - [ci-tools, registry.acme.com/ci/debugger]
  override:
    exceptions: append
```

**Prevent →** Route noisy, low-priority rules to a log index instead of chat. Review exceptions quarterly so the allowlist does not hide real attacks.

</details>

### Q159. The Falco rule `Read sensitive file untrusted` fires constantly for your log-shipping DaemonSet reading `/etc/shadow` on the host 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The shipper mounts the host filesystem and reads broadly (a glob on `/host/etc/*`), so it really does read sensitive files.

**Diagnose →** Check `proc.name`, `fd.name` and the container image in the alert, then look at the shipper's input config.

**Fix →** First fix the shipper. Narrow its input paths to `/var/log/**` and mount only what it needs, read-only. Only if the read is truly required, add an exception for that image and file in the rule.

**Prevent →** Review the hostPath mounts of every DaemonSet. A Kyverno policy can allowlist which hostPaths each namespace may mount.

</details>

### Q160. `kubectl logs ds/falco` shows alerts, but nothing ever arrives in Slack 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Falcosidekick is not deployed or not wired up. Falco needs `json_output: true` and `http_output` pointing at Sidekick. The Slack webhook URL may also be wrong or the `minimumpriority` filter set too high.

**Diagnose →**
```bash
kubectl logs -n falco deploy/falco-falcosidekick | tail
kubectl get cm -n falco falco -o yaml | grep -A3 -E 'json_output|http_output'
```

**Fix →**
```bash
helm upgrade falco falcosecurity/falco -n falco --reuse-values \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="$SLACK_WEBHOOK" \
  --set falcosidekick.config.slack.minimumpriority=warning
```

**Prevent →** Trigger a test event (`kubectl exec` into a canary pod) after each Falco upgrade and confirm it reaches every output.

</details>

### Q161. A crypto-miner ran in a pod for two days and Falco raised no alert 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The default ruleset has limited outbound-connection coverage, and miner detections may live in rules that are disabled by default or in the incubating/sandbox rulesets. Falco may also have been dropping events, or the pod ran on a node where Falco was not scheduled (taints without tolerations).

**Diagnose →**
```bash
kubectl get ds falco -n falco -o wide        # DESIRED vs READY vs node count
kubectl logs -n falco ds/falco | grep -i "syscall event drop"
```
Check the loaded rules for miner and stratum detections.

**Fix →** Add tolerations so Falco runs on every node. Enable rules for outbound connections to mining pools and for known miner binaries. Combine with NetworkPolicy default-deny egress and DNS logging, which catches miners even when rules miss them.

**Prevent →** Use layered detection (Falco, GuardDuty Runtime Monitoring or cloud DNS findings, and CPU anomaly alerts), and alert on Falco DaemonSet coverage below 100%.

</details>

### Q162. Under peak load, Falco logs `Falco internal: syscall event drop. 18432 system calls dropped in last second.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ring buffer between the kernel probe and userspace fills up because the syscall rate exceeds what Falco processes. Falco may be CPU-throttled by its limits, or it collects syscalls no rule needs.

**Diagnose →**
```bash
kubectl top pod -n falco
kubectl get ds falco -n falco -o yaml | yq '.spec.template.spec.containers[0].resources'
```
Enable `metrics.enabled: true` and watch the drop counters.

**Fix →** Raise or remove the CPU limit for Falco. Increase `engine.modern_ebpf.buf_size_preset`. Let Falco collect only the syscalls the loaded rules need (the default `base_syscalls.repair: true` behaviour), and drop expensive rules you do not use.

**Prevent →** Alert on drop rate. Attackers can deliberately create syscall noise to blind runtime detection.

</details>

### Q163. A Tetragon `TracingPolicy` with a `Sigkill` action killed your payment service's legitimate `curl` health check in production 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The policy was shipped straight in enforcement mode, and its selectors (binary path, namespace, pod labels) were broader than intended.

**Diagnose →**
```bash
kubectl get tracingpolicy,tracingpolicynamespaced -A
kubectl logs -n kube-system ds/tetragon -c export-stdout | grep -i sigkill | jq '.process_kprobe.process | {binary, pod: .pod.name}'
```

**Fix →** Change the action to `Post` (observe only) or delete the policy. Narrow `podSelector` and `matchBinaries`, and add an allowlist for the health-check path.

**Prevent →** Always roll out runtime enforcement in stages: observe for one to two weeks, review hits, then enforce first in a single canary namespace.

</details>

### Q164. kube-bench on EKS reports dozens of FAILs under "1 Control Plane Security Configuration" 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You ran the generic Kubernetes benchmark. On EKS the control plane is managed by AWS and is not on your nodes, so those checks cannot pass and do not apply.

**Diagnose →** The output header shows which benchmark ran. Check whether the job used `job.yaml` or `job-eks.yaml`.

**Fix →** Run the EKS benchmark, which only covers worker node and policy sections:
```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-eks.yaml
kubectl logs job/kube-bench
```

**Prevent →** Pin the benchmark per platform in your compliance pipeline, and use the cloud provider's posture tools for control-plane settings.

</details>

### Q165. kube-bench fails `Ensure that the --protect-kernel-defaults argument is set to true` on every node, but setting it makes the kubelet refuse to start 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With `protectKernelDefaults: true`, the kubelet will not start if kernel sysctls differ from what it expects (`vm.overcommit_memory=1`, `kernel.panic=10`, `kernel.panic_on_oops=1`, `vm.panic_on_oom=0`).

**Diagnose →**
```bash
journalctl -u kubelet | grep -i "protect-kernel-defaults\|sysctl"
sysctl vm.overcommit_memory kernel.panic kernel.panic_on_oops vm.panic_on_oom
```

**Fix →** Set the sysctls in the node image or user data first (`/etc/sysctl.d/90-kubelet.conf`), then enable `protectKernelDefaults: true` in the kubelet config.

**Prevent →** Bake CIS node settings into the golden AMI and run kube-bench in the image pipeline, not only on live clusters.

</details>

### Q166. The kube-bench job completes but most checks show `WARN` or "unable to find config file" 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job does not mount the host paths where the kubelet config lives, or the distribution uses non-standard paths (k3s, RKE2, Bottlerocket), or version auto-detection picked the wrong benchmark.

**Diagnose →** `kubectl logs job/kube-bench | head -30`, and check the job's `volumeMounts` against where the kubelet config actually lives on the node (`ps -ef | grep kubelet`).

**Fix →** Use the distribution-specific job manifest, or pass the right config with `--benchmark` and `--config-dir`, and mount `/var/lib/kubelet` and `/etc/kubernetes` read-only. On Bottlerocket, use the OS's own CIS reporting.

**Prevent →** Fail the compliance job when the WARN count passes a threshold, so a broken scan is not mistaken for a clean one.

</details>

### Q167. Security Hub still shows `[CloudTrail.1]` as FAILED although you created a trail yesterday 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The control needs at least one **multi-Region** trail that logs **read and write management events**. A single-region trail, or one logging only write events, does not pass. The control may also be evaluated in an account the trail does not cover.

**Diagnose →**
```bash
aws cloudtrail describe-trails --query 'trailList[].{n:Name,multi:IsMultiRegionTrail,org:IsOrganizationTrail}'
aws cloudtrail get-event-selectors --trail-name main
```

**Fix →** `aws cloudtrail update-trail --name main --is-multi-region-trail`, and set management events to `ReadWriteType: All`. Better, create an organization trail from the management account.

**Prevent →** Deploy trails and Config through account-vending baselines (Control Tower or Terraform), not by hand.

</details>

### Q168. Security Hub shows the same failed control 40 times per account, and the dashboard is unreadable 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Several standards (CIS, AWS FSBP, PCI) are enabled with consolidated control findings turned off, so each standard emits its own finding. Findings are also not aggregated into one region.

**Diagnose →**
```bash
aws securityhub get-enabled-standards
aws securityhub describe-hub --query ControlFindingGenerator
aws securityhub list-finding-aggregators
```

**Fix →** Set `ControlFindingGenerator` to `SECURITY_CONTROL` (consolidated findings), create a cross-region finding aggregator in the delegated admin account, and disable controls that do not apply (with documented reasons).

**Prevent →** Manage Security Hub configuration centrally with central configuration policies from the delegated admin account.

</details>

### Q169. A Prowler run finishes, but hundreds of checks show `AccessDenied` errors instead of PASS or FAIL 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The scanning role lacks the read permissions Prowler needs. `ReadOnlyAccess` alone misses some APIs. Prowler documents `SecurityAudit` and `ViewOnlyAccess`, plus a small additional policy.

**Diagnose →** Check the Prowler log lines with `AccessDenied` for the denied API calls, and run `aws sts get-caller-identity` to confirm which role ran it.

**Fix →** Attach `arn:aws:iam::aws:policy/SecurityAudit` and `arn:aws:iam::aws:policy/job-function/ViewOnlyAccess` to the scan role, add Prowler's extra permissions policy, then rerun with `prowler aws`.

**Prevent →** Deploy the scan role to every account with StackSets or Terraform, and treat "check errored" as a failure in reports.

</details>

### Q170. GuardDuty raises `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS` for an EKS node role 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Temporary credentials from that node's instance profile are being used from an IP outside AWS. Something on the node, often a compromised pod reaching IMDS, stole them.

**Diagnose →**
```bash
aws guardduty get-findings --detector-id <id> --finding-ids <fid> | jq '.Findings[].Service.Action'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<ASIA...>
```

**Fix →** Revoke active sessions for the role ("Revoke active sessions" adds an `aws:TokenIssueTime` deny). Isolate the node (cordon, quarantine security group) and keep it for forensics. Find the pod that reached IMDS, and review everything the stolen credentials did.

**Prevent →** IMDSv2 with hop limit 1, minimal node roles, IRSA or Pod Identity for workloads, and GuardDuty EKS protection with runtime monitoring enabled.

</details>

### Q171. GuardDuty reports `CryptoCurrency:EC2/BitcoinTool.B!DNS` for an EKS worker node 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A process on the node resolved a known mining pool domain. It is usually a compromised container (exposed dashboard, RCE in an app, or a malicious image), rarely the node itself.

**Diagnose →**
```bash
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
kubectl top pods -A --sort-by=cpu | head
```
Check Route 53 Resolver query logs or CoreDNS logs for the pool domain and the source pod IP.

**Fix →** Isolate the pod with a deny-all NetworkPolicy, scale its owner to zero, and capture the image digest and logs before deleting anything. Then find the entry point.

**Prevent →** Default-deny egress, Route 53 Resolver DNS Firewall blocklists, signed images from trusted registries, and Falco or GuardDuty runtime monitoring.

</details>

### Q172. The InSpec CIS profile fails `PasswordAuthentication should be no` on the golden AMI, although the Ansible hardening role sets it in `/etc/ssh/sshd_config` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** On recent Ubuntu, Debian and RHEL releases, `sshd_config` starts with `Include /etc/ssh/sshd_config.d/*.conf`, and sshd uses the **first** value it reads. A drop-in like `50-cloud-init.conf` setting `PasswordAuthentication yes` wins over your line further down.

**Diagnose →**
```bash
sudo sshd -T | grep -i passwordauthentication     # effective config
grep -ri passwordauthentication /etc/ssh/sshd_config.d/
```

**Fix →** Write your hardening as a drop-in that sorts first (for example `/etc/ssh/sshd_config.d/00-hardening.conf`), or remove the conflicting drop-in, then run `sshd -t && systemctl reload ssh`.

**Prevent →** Test with the *effective* config (`sshd -T`) in the image pipeline. InSpec's `sshd_active_config` resource checks the running state.

</details>

### Q173. `oscap xccdf eval` against your RHEL 9 hosts reports almost every rule as `notapplicable` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You used the wrong SCAP datastream (for example `ssg-rhel8-ds.xml` on RHEL 9), so the CPE platform checks do not match. Running inside a container instead of on the host does the same.

**Diagnose →**
```bash
cat /etc/os-release
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml | grep -i profile
```

**Fix →**
```bash
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis \
  --results results.xml --report report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

**Prevent →** Pin the datastream to the OS version in automation, and fail the job if the `notapplicable` count is high.

</details>

### Q174. During an investigation you cannot tell who read a production Secret: EKS audit logs show nothing for `get secrets` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `audit` control-plane log type is not enabled on the cluster. On self-managed clusters, the audit policy may set `level: None` for Secrets or only log write verbs.

**Diagnose →**
```bash
aws eks describe-cluster --name prod --query 'cluster.logging.clusterLogging'
# self-managed: check --audit-policy-file rules for resources: ["secrets"]
```

**Fix →** Enable audit logging going forward:
```bash
aws eks update-cluster-config --name prod \
  --logging '{"clusterLogging":[{"types":["audit","authenticator"],"enabled":true}]}'
```
For self-managed clusters, log Secrets at `Metadata` level, never `Request`, which would write secret values into the logs. The past reads cannot be recovered, so assume the secret is exposed and rotate it.

**Prevent →** Make audit logging part of the cluster baseline, ship logs to a SIEM with retention that meets your compliance needs, and alert on unusual Secret `list` calls.

</details>

### Q175. `kubescape scan framework nsa` scores your cluster at 41% and flags `Automatic mapping of service account` on almost every workload 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** By default every pod gets its ServiceAccount token mounted, even when the app never calls the Kubernetes API. A stolen token is then a free foothold.

**Diagnose →**
```bash
kubescape scan control C-0034 -v
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {.spec.automountServiceAccountToken}{"\n"}{end}'
```

**Fix →** Set `automountServiceAccountToken: false` on the ServiceAccounts or pod specs of apps that do not need the API. Keep it on only for controllers and operators, with minimal RBAC.

**Prevent →** A Kyverno mutate policy that defaults it to `false` unless a namespace opts in, plus Kubescape in CI on rendered manifests.

</details>

## Certificates, TLS and security incident response

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. A cert-manager Certificate stays `Ready=False` and the Challenge shows `Waiting for HTTP-01 challenge propagation: wrong status code '404', expected '200'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The solver's temporary Ingress or route for `/.well-known/acme-challenge/<token>` is not served. Typical reasons: the wrong `ingressClassName` in the Issuer solver, an HTTP-to-HTTPS redirect or auth in front of the path, or DNS for the host not yet pointing at this load balancer.

**Diagnose →**
```bash
cmctl status certificate api-tls -n prod
kubectl describe challenge -n prod
kubectl get ingress -n prod | grep cm-acme-http-solver
curl -v http://api.acme.com/.well-known/acme-challenge/test
```

**Fix →** Set the solver's ingress class to the one that actually serves the host:
```yaml
solvers:
- http01:
    ingress:
      ingressClassName: nginx
```
Exempt the challenge path from forced HTTPS or auth, and fix DNS.

**Prevent →** For wildcards, internal hosts or anything behind a CDN, use DNS-01. Test new issuers against the Let's Encrypt staging endpoint.

</details>

### Q177. A DNS-01 challenge hangs forever with `DNS record for "acme.com" not yet propagated`, although `dig` from your laptop shows the TXT record 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** cert-manager checks propagation through the cluster's resolver. With split-horizon DNS, the cluster sees a private zone for `acme.com` that does not contain the public TXT record. Another cause is a delegated `_acme-challenge` CNAME the solver does not follow.

**Diagnose →**
```bash
kubectl run d --rm -it --image=busybox -- nslookup -type=TXT _acme-challenge.acme.com
kubectl logs -n cert-manager deploy/cert-manager | grep -i propagat
```

**Fix →** Make cert-manager use public resolvers for the self-check:
```bash
helm upgrade cert-manager jetstack/cert-manager -n cert-manager --reuse-values \
  --set 'extraArgs={--dns01-recursive-nameservers-only,--dns01-recursive-nameservers=1.1.1.1:53\,8.8.8.8:53}'
```

**Prevent →** Document split-horizon zones, and give cert-manager a DNS provider credential scoped only to `_acme-challenge` records where your provider supports it.

</details>

### Q178. Issuance fails with `429 urn:ietf:params:acme:error:rateLimited: too many certificates already issued` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You hit a Let's Encrypt rate limit, usually the duplicate-certificate limit (the same set of names re-issued too often) or the per-registered-domain weekly limit. Typical triggers are a CrashLooping deploy that deletes and recreates certificate Secrets, or many preview environments.

**Diagnose →**
```bash
kubectl get certificaterequest -A --sort-by=.metadata.creationTimestamp | tail -20
kubectl describe order -n prod | grep -A2 -i ratelimit
```

**Fix →** Stop the churn and wait out the window. Meanwhile, move preview environments to one wildcard certificate or to the staging issuer.

**Prevent →** Use the staging ACME endpoint for test setups, never delete TLS Secrets in deploy scripts, and use a wildcard for `*.preview.acme.com`.

</details>

### Q179. A service calling another internal service fails with `x509: certificate signed by unknown authority` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server uses a certificate from your private CA, and the client container's trust store (often a minimal or distroless image) does not include that CA.

**Diagnose →**
```bash
openssl s_client -connect billing.internal:443 -servername billing.internal -showcerts </dev/null | openssl x509 -noout -issuer
kubectl exec deploy/api -- ls /etc/ssl/certs | head
```

**Fix →** Distribute the CA bundle to workloads, for example with cert-manager's trust-manager `Bundle` synced into a ConfigMap, mounted at `/etc/ssl/certs/acme-ca.pem`, with `SSL_CERT_FILE` pointed at it. Or add it in the base image (`update-ca-certificates`). Never "fix" it with `InsecureSkipVerify` or `-k`.

**Prevent →** Ship the private CA in every base image and use one managed bundle source.

</details>

### Q180. At 05:30 every client starts failing with `x509: certificate has expired or is not yet valid` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The certificate expired because nobody renewed a manually issued one, or automated renewal was failing silently. Less often, the client clock is wrong ("not yet valid").

**Diagnose →**
```bash
echo | openssl s_client -connect api.acme.com:443 -servername api.acme.com 2>/dev/null | openssl x509 -noout -dates
date -u
```

**Fix →** Renew and deploy now (`cmctl renew api-tls -n prod`, or reissue in your CA or ACM). Then restart or reload anything that caches the certificate.

**Prevent →** Automate renewal everywhere, and alert at 21 and 7 days before expiry with blackbox exporter's `probe_ssl_earliest_cert_expiry` or cert-manager's `certmanager_certificate_expiration_timestamp_seconds`.

</details>

### Q181. Browsers load `https://api.acme.com` fine, but `curl` and Java clients fail with `unable to get local issuer certificate` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server sends only the leaf certificate, not the intermediate. Browsers often fill the gap from cache or AIA fetching. Most other clients do not.

**Diagnose →**
```bash
openssl s_client -connect api.acme.com:443 -servername api.acme.com -showcerts </dev/null | grep -c 'BEGIN CERTIFICATE'
# 1 = leaf only
```

**Fix →** Serve the full chain: leaf first, then intermediates (`fullchain.pem`, or `tls.crt` containing both). In Kubernetes, check that the TLS Secret's `tls.crt` has the full chain.

**Prevent →** Add a chain-completeness check to the synthetic monitor (`openssl verify -untrusted chain.pem leaf.pem`).

</details>

### Q182. `x509: certificate is valid for acme.com, *.acme.com, not api.eu.acme.com` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A wildcard covers exactly one label. `*.acme.com` matches `api.acme.com`, not `api.eu.acme.com`.

**Diagnose →** `openssl x509 -in tls.crt -noout -ext subjectAltName`

**Fix →** Issue a certificate with `*.eu.acme.com` or the explicit name `api.eu.acme.com` in the SANs:
```yaml
spec:
  dnsNames: ["api.eu.acme.com", "*.eu.acme.com"]
```

**Prevent →** Plan the hostname hierarchy together with the certificate strategy, and test new hostnames in staging with real certificates.

</details>

### Q183. After setting Istio `PeerAuthentication` to `STRICT`, a legacy VM client gets `curl: (56) Recv failure: Connection reset by peer` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** STRICT mode only accepts mTLS connections from mesh workloads. A plaintext client outside the mesh is rejected at the sidecar.

**Diagnose →**
```bash
istioctl x describe pod <server-pod> -n prod
kubectl get peerauthentication -A
kubectl logs <server-pod> -c istio-proxy | tail
```

**Fix →** Bring the client into the mesh (a sidecar, or WorkloadEntry for the VM), or route it through an ingress gateway that terminates TLS. As a time-boxed exception, set `PERMISSIVE` on that one workload or port with `portLevelMtls`.

**Prevent →** Before going STRICT, check telemetry for plaintext connections (`connection_security_policy="none"` in Istio metrics) and migrate those clients first.

</details>

### Q184. A Java service fails with `PKIX path building failed: unable to find valid certification path to requested target` right after the internal CA was rotated 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The JVM uses its own truststore (`$JAVA_HOME/lib/security/cacerts` or a custom `-Djavax.net.ssl.trustStore`), not the OS bundle. The new root or intermediate was never imported.

**Diagnose →**
```bash
kubectl exec deploy/orders -- sh -c 'keytool -list -cacerts -storepass changeit | grep -i acme'
kubectl exec deploy/orders -- sh -c 'echo $JAVA_TOOL_OPTIONS'
```

**Fix →**
```bash
keytool -importcert -cacerts -storepass changeit -noprompt -alias acme-root-2026 -file acme-root-2026.crt
```
Do this in the image build, or mount a managed truststore.

**Prevent →** Rotate CAs with overlap: publish the new root to every truststore weeks before the switch and keep the old one until all leaf certificates have moved.

</details>

### Q185. `kubectl` against a kubeadm cluster fails with `Unable to connect to the server: x509: certificate has expired or is not yet valid` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** kubeadm-issued control-plane certificates last one year. They are renewed automatically only during `kubeadm upgrade`, and this cluster was not upgraded for a year.

**Diagnose →**
```bash
sudo kubeadm certs check-expiration
```

**Fix →** On each control-plane node:
```bash
sudo kubeadm certs renew all
# restart static pods: move manifests out and back
sudo mkdir -p /tmp/m && sudo mv /etc/kubernetes/manifests/*.yaml /tmp/m/ && sleep 20 && sudo mv /tmp/m/*.yaml /etc/kubernetes/manifests/
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
```
Check that kubelet client certificate rotation is on (`rotateCertificates: true`).

**Prevent →** Upgrade at least yearly, and alert on `kubeadm certs check-expiration` (or `apiserver_client_certificate_expiration_seconds`) at 30 days.

</details>

### Q186. After hardening the load balancer to TLS 1.3 only, a partner's integration fails with `tls: protocol version not supported` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The partner's client (an old Java 8 build, .NET Framework or an embedded device) only speaks TLS 1.2.

**Diagnose →**
```bash
openssl s_client -connect api.acme.com:443 -tls1_2 </dev/null    # fails now
nmap --script ssl-enum-ciphers -p 443 api.acme.com
```
LB access logs show the client's TLS version.

**Fix →** Allow TLS 1.2 with strong AEAD ECDHE ciphers only (for example the AWS policy `ELBSecurityPolicy-TLS13-1-2-2021-06`). Keep TLS 1.0 and 1.1 disabled.

**Prevent →** Before tightening, measure the TLS versions in use from LB logs, notify integrators, and set a date.

</details>

### Q187. cert-manager renewed the certificate and the Secret has the new `notAfter`, but clients still see the old certificate 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app or proxy loads the certificate once at startup and never reloads it. Or the certificate was copied somewhere else (a cloud LB, a CDN, a different namespace) that cert-manager does not manage.

**Diagnose →**
```bash
kubectl get secret api-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -enddate
echo | openssl s_client -connect api.acme.com:443 -servername api.acme.com 2>/dev/null | openssl x509 -noout -enddate
```

**Fix →** Restart or reload the workload, and automate it with Reloader on the Secret or with an app that watches the mounted file. For external endpoints, push the certificate there automatically or use the provider's managed certificates.

**Prevent →** Monitor the certificate that is actually *served* (blackbox probe), not just the Secret.

</details>

### Q188. An ACM certificate for `api.acme.com` has been `Pending validation` for two days 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The DNS validation CNAME was never created, was created in the wrong hosted zone (private instead of public), or the domain is delegated to another DNS provider.

**Diagnose →**
```bash
aws acm describe-certificate --certificate-arn <arn> --query 'Certificate.DomainValidationOptions[].ResourceRecord'
dig +short CNAME _abc123.api.acme.com
dig +short NS acme.com
```

**Fix →** Create the exact CNAME in the authoritative public zone. In Terraform, use `aws_acm_certificate_validation` with `aws_route53_record` built from `domain_validation_options`.

**Prevent →** Use DNS validation in code, and keep the CNAMEs in place so ACM can renew automatically.

</details>

### Q189. AWS emails that an access key was found on public GitHub, and the IAM user now has an `AWSCompromisedKeyQuarantine` policy attached 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A long-lived IAM user key was committed. Bots scrape GitHub within minutes, so assume it has been used. The quarantine policy denies some high-risk actions, but not everything.

**Diagnose →**
```bash
aws iam list-access-keys --user-name ci-legacy
aws cloudtrail lookup-events --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIA... \
  --start-time 2026-11-01 --query 'Events[].[EventTime,EventName,EventSource]' --output table
```
Also check for new IAM users, keys, roles, EC2 instances in every region, and Lambda functions.

**Fix →** Deactivate and then delete the key. Remove any persistence the attacker created (users, keys, roles, trust policy edits), terminate rogue resources in all regions, and replace the key with a role (OIDC for CI). Open an AWS Support case for billing.

**Prevent →** No IAM user access keys for automation, secret scanning with push protection, an SCP blocking `iam:CreateAccessKey` outside break-glass, and budget alerts.

</details>

### Q190. CloudTrail shows `CreateUser`, `AttachUserPolicy` (AdministratorAccess) and `CreateAccessKey` at 03:12 from an unfamiliar IP, made by a CI role 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CI role's credentials, or the pipeline itself, were compromised, and the attacker is setting up persistence that survives revoking the CI credentials.

**Diagnose →**
```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=<new-user>
aws cloudtrail lookup-events --start-time <t> --query 'Events[?contains(CloudTrailEvent, `<ip>`)].[EventTime,EventName]'
```
Trace back to the session: which workflow run or OIDC `sub` assumed the role at that time.

**Fix →** Contain first. Deactivate the new keys, attach a deny-all to the new user, revoke sessions on the CI role, and disable the pipeline. Then remove the persistence, hunt in every region and service, and find the entry point (a malicious PR, a compromised action, a leaked token). Preserve the logs.

**Prevent →** CI roles should not have `iam:*`. Use permission boundaries, an SCP denying `iam:CreateUser` and `CreateAccessKey`, GuardDuty, and alerts on IAM write events from CI roles.

</details>

### Q191. A running pod is confirmed to be mining crypto. The team wants to `kubectl delete` it immediately 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Not a cause question. The risk is destroying the evidence you need to find the entry point, and the controller simply recreating the same compromised pod.

**Diagnose →** Capture the evidence before changing anything:
```bash
kubectl get pod miner-x -o yaml > pod.yaml
kubectl logs miner-x --all-containers > logs.txt
kubectl get pod miner-x -o jsonpath='{.status.containerStatuses[*].imageID}'
```

**Fix →** Contain without deleting. Apply a deny-all NetworkPolicy matching a quarantine label, remove the pod from Service selectors (relabel it), and cordon the node. Snapshot the node's volume, or capture memory and the filesystem with your forensics tooling. Scale the owner to zero, and only then delete. Rotate every secret the pod could access.

**Prevent →** Keep an IR runbook with these commands ready, a pre-created quarantine NetworkPolicy, and drill it.

</details>

### Q192. A production DB password used by 30 services leaked, and it has to be rotated without an outage 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A single shared static credential with no rotation design.

**Diagnose →** List every consumer: secret-manager access logs, DB connection logs (`pg_stat_activity` grouped by `usename` and `client_addr`), and code search.

**Fix →** Rotate with dual credentials. Create a new DB user or password, update the secret store, and roll the services in batches. Watch for authentication errors, then disable the old credential and check the DB logs for any remaining use. If active abuse is suspected, cut over immediately and accept some errors.

**Prevent →** One credential per service, managed rotation (Secrets Manager rotation or Vault dynamic credentials), and apps that reload credentials without a redeploy.

</details>

### Q193. A public base image tag you use (`node:20-alpine` mirrored from a community repo) suddenly ships a binary that phones home 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tags are mutable. The upstream publisher or a compromised maintainer pushed a new image under the same tag, and your builds pull by tag.

**Diagnose →**
```bash
crane digest docker.io/someorg/node:20-alpine
# compare with digests recorded in your SBOMs and build logs
grype registry.acme.com/api@sha256:... --only-fixed=false
```

**Fix →** Rebuild from a known-good digest or an official image. Find every image built from the bad digest (from your SBOMs or the registry), redeploy them, and investigate any outbound connections they made.

**Prevent →** Pin base images by digest (Renovate keeps them updated), pull only through an internal registry mirror, use official or verified publishers, and verify signatures where they exist.

</details>

### Q194. The Kubernetes audit log shows successful `list pods` requests from `system:anonymous` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Anonymous auth is on (the API server default) **and** someone bound a role to `system:anonymous` or `system:unauthenticated`, often a quick fix for a monitoring tool or a tutorial.

**Diagnose →**
```bash
kubectl get clusterrolebindings,rolebindings -A -o json | \
  jq -r '.items[] | select(.subjects[]?.name | test("system:(anonymous|unauthenticated)")) | "\(.kind) \(.metadata.namespace // "-") \(.metadata.name) -> \(.roleRef.name)"'
kubectl auth can-i --list --as=system:anonymous
```

**Fix →** Delete those bindings. Review the audit log for what anonymous callers accessed and from which IPs. Restrict API endpoint exposure (private endpoint or authorized networks).

**Prevent →** Run a policy or CI check that rejects bindings to anonymous or unauthenticated subjects. Where possible, restrict anonymous auth to health endpoints only (`AuthenticationConfiguration` anonymous conditions in recent Kubernetes versions).

</details>

### Q195. An engineer reports their laptop was stolen while unlocked; it had a kubeconfig, an active AWS SSO session and a GitHub CLI token 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every cached credential on the device is now at risk until it is revoked.

**Diagnose →** Inventory what the person can access: IdP group memberships, AWS permission sets, GitHub org roles, and kubeconfig contexts. Then check recent activity from unknown IPs in CloudTrail, GitHub audit logs and Kubernetes audit logs.

**Fix →** In order: suspend the IdP account and revoke all sessions (this kills SSO-derived access), revoke GitHub tokens and SSH keys, revoke the AWS IAM Identity Center sessions, remove static Kubernetes credentials, and rotate any secrets stored locally (`.env` files). Trigger a remote wipe through MDM.

**Prevent →** Short-lived credentials only (SSO plus OIDC for kubectl), disk encryption with auto-lock enforced by MDM, and hardware security keys for the IdP.

</details>

### Q196. A pentest finds Grafana reachable from the internet with `admin/admin`, and an exposed Kubernetes Dashboard 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tools were installed with Helm defaults and exposed with a `LoadBalancer` Service or a public Ingress for convenience.

**Diagnose →**
```bash
kubectl get svc -A --field-selector spec.type=LoadBalancer
kubectl get ingress -A
```
Check the Grafana and Dashboard access logs for logins from unknown IPs.

**Fix →** Take them off the public network right away (ClusterIP, internal LB, or behind an auth proxy with SSO). Change the Grafana admin password, enable SSO and disable basic login. Check the Dashboard SA's RBAC and remove the Dashboard if no one needs it. Review what was accessed.

**Prevent →** A policy that blocks `type: LoadBalancer` without an internal annotation in non-ingress namespaces, an external attack-surface scan, and no default credentials in Helm values.

</details>

### Q197. During an incident you find CloudTrail has no events for a 6-hour window, and the trail shows a `StopLogging` call 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The attacker had enough privilege to stop the trail to cover their tracks. Some evidence may also have been deleted from the log bucket.

**Diagnose →**
```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=StopLogging
aws cloudtrail get-trail-status --name main
aws cloudtrail validate-logs --trail-arn <arn> --start-time <t>   # detects missing or modified files
```
Event history (90 days of management events) is separate from the trail, so check it for the gap. Also use other sources: VPC Flow Logs, S3 access logs, GuardDuty and Config.

**Fix →** Restart logging, contain the principal that called `StopLogging`, and reconstruct the window from secondary sources.

**Prevent →** An organization trail owned by the management account (member accounts cannot stop it), an SCP denying `cloudtrail:StopLogging`/`DeleteTrail`, log file validation, a separate log-archive account with S3 Object Lock, and alerts on any CloudTrail configuration change.

</details>

### Q198. A new critical zero-day in a popular Java library is announced. Leadership wants to know within an hour which running services are affected 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Not a failure yet. The difficulty is having no queryable inventory of what runs where.

**Diagnose →** Query what you already have:
```bash
# SBOMs stored per image digest (Dependency-Track, GUAC, or S3 + jq)
grype sbom:./sboms/api.cdx.json | grep -i <library>
# Live cluster inventory
kubectl get pods -A -o jsonpath='{range .items[*]}{.status.containerStatuses[*].imageID}{"\n"}{end}' | sort -u > running.txt
trivy k8s --report summary --scanners vuln
```
Map running digests to SBOMs, and check shaded or fat JARs, which name-based scanning can miss.

**Fix →** Prioritise internet-facing services. Apply mitigations (WAF rules, config flags, removing the vulnerable class), then roll out patched builds.

**Prevent →** Generate SBOMs at build time, store them by digest, and keep a continuously updated runtime inventory. This drill should take minutes, not days.

</details>

### Q199. A client calling an mTLS-protected internal API gets `400 Bad Request: No required SSL certificate was sent` from nginx 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** nginx has `ssl_verify_client on`, and the client did not present a certificate. Either it is not configured with one, or the certificate it has was not issued by a CA in `ssl_client_certificate`, so the client never offers it.

**Diagnose →**
```bash
curl -v --cert client.crt --key client.key --cacert ca.crt https://internal-api.acme.com/health
openssl s_client -connect internal-api.acme.com:443 -servername internal-api.acme.com </dev/null | grep -A5 "Acceptable client certificate CA names"
```

**Fix →** Configure the client with a certificate and key issued by a CA nginx trusts. If the client CA was rotated, add the new CA to the bundle in `ssl_client_certificate`.

**Prevent →** Issue client certificates automatically (cert-manager or a service mesh) with short lifetimes, and monitor expiry on both sides.

</details>

### Q200. The postmortem shows this is the third time this year a cloud credential leaked through a Git commit 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The fixes so far targeted people ("be more careful") rather than the system. Long-lived credentials still exist, scanning is opt-in, and secrets are easy to copy into files.

**Diagnose →** Look at the three incidents together: what kind of credential, why it existed, how it was committed, how it was detected, and how long it was exposed.
```bash
gitleaks detect --source . --log-opts="--all" --report-format json --report-path all-history.json
```

**Fix →** Remove the class of credential, not just the instance. Replace static keys with OIDC and workload identity, enable org-wide push protection and server-side secret scanning, add pre-commit hooks through a managed config, and route leak alerts to an owned on-call rotation with auto-revocation where providers support it.

**Prevent →** Track metrics (number of long-lived credentials, time to revoke) and make each postmortem action item a guardrail with an owner and a due date, not a reminder.

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
