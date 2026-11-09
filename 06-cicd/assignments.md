# CI/CD — Assignments

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

> 25 practical assignments across GitHub Actions, Jenkins and GitLab CI · 🟢 10 Beginner · 🟡 10 Intermediate · 🔴 5 Advanced

[⬅ Back to CI/CD](./README.md) · [🏠 Home](../README.md)

## How to use these

Do them in order inside each level. Work in your own repository, commit each assignment in its own folder or branch, and keep the pipeline run URL as evidence. If you get stuck, open the hint; try for at least 20 minutes first. Assignment 21 builds on the [hands-on lab](./hands-on-lab.md).

## Table of contents

- [🟢 Level 1: Beginner (Assignments 1-10)](#-level-1-beginner-assignments-1-10)
- [🟡 Level 2: Intermediate (Assignments 11-20)](#-level-2-intermediate-assignments-11-20)
- [🔴 Level 3: Advanced (Assignments 21-25)](#-level-3-advanced-assignments-21-25)

---

## 🟢 Level 1: Beginner (Assignments 1-10)

### Assignment 1. Your first workflow 🟢

**Objective:** Run a GitHub Actions workflow on every push.

**Tasks**

1. Create a repo with a `README.md` and a `hello.sh` script that prints the current date.
2. Add `.github/workflows/hello.yml` triggered on `push` and `workflow_dispatch`.
3. Make one step run `./hello.sh` and another print `${{ github.sha }}` and `${{ github.actor }}`.
4. Trigger the workflow from the UI and from a push.

**Expected outcome / acceptance criteria**

- The Actions tab shows a green run for both triggers.
- The log shows the commit SHA and your username.

<details>
<summary><b>Hints</b></summary>

Use `runs-on: ubuntu-latest`, `actions/checkout@v4` and remember `chmod +x hello.sh` before committing (or run `bash hello.sh`).

</details>

**Bonus:** Add a `schedule` trigger with a cron expression and note that scheduled runs only fire on the default branch.

### Assignment 2. Lint and test a small app 🟢

**Objective:** Fail the pipeline when lint or tests fail.

**Tasks**

1. Create a small Python or Node app with two unit tests and a linter (`ruff` or `eslint`).
2. Write a workflow that installs dependencies, lints, then tests.
3. Push a commit with a lint error and confirm the run fails.
4. Fix it and confirm the run turns green.

**Expected outcome / acceptance criteria**

- Failing lint stops the job before tests run.
- The failing step name is visible in the run summary.

<details>
<summary><b>Hints</b></summary>

Each `run` step fails the job on a non-zero exit code. Order cheap checks first so failures are fast.

</details>

**Bonus:** Upload the test report with `actions/upload-artifact@v4` even on failure using `if: always()`.

### Assignment 3. Dependency caching 🟢

**Objective:** Cut install time with a cache.

**Tasks**

1. Take the app from the previous assignment.
2. Enable caching using `actions/setup-node@v4` (`cache: npm`) or `actions/setup-python@v5` (`cache: pip`).
3. Run the workflow twice and compare install step durations.
4. Write the two timings in your repo README.

**Expected outcome / acceptance criteria**

- The second run logs a cache hit.
- Install time is visibly lower.

<details>
<summary><b>Hints</b></summary>

The cache key is derived from the lockfile hash, so you must commit `package-lock.json` or `requirements.txt` pinned versions.

</details>

**Bonus:** Replace the built-in cache with an explicit `actions/cache@v4` step and a custom key with `restore-keys`.

### Assignment 4. Secrets and variables 🟢

**Objective:** Use secrets and variables correctly and prove they are masked.

**Tasks**

1. Create a repository secret `API_TOKEN` and a repository variable `APP_ENV=staging`.
2. Write a workflow that prints `APP_ENV` and uses `API_TOKEN` in an environment variable of a step.
3. Try to `echo` the secret and observe the log.
4. Explain in a short note why masking is not a security boundary.

**Expected outcome / acceptance criteria**

- The log shows `***` instead of the secret value.
- The variable value prints in clear text.

<details>
<summary><b>Hints</b></summary>

Reference with `${{ secrets.API_TOKEN }}` and `${{ vars.APP_ENV }}`. Pass secrets via `env:`, not by interpolating into the script body.

</details>

**Bonus:** Show how a base64-encoded secret printed in two halves leaks despite masking, then explain why OIDC removes the problem.

### Assignment 5. Build a Docker image in CI 🟢

**Objective:** Build and tag an image in a workflow.

**Tasks**

1. Write a Dockerfile for the app.
2. Add a workflow job that runs `docker build` with tag `app:${{ github.sha }}`.
3. Run the image with `docker run -d -p 8080:8080` in the job and `curl` the health endpoint.
4. Stop the container at the end.

**Expected outcome / acceptance criteria**

- The job fails if `curl -f` receives a non-2xx response.
- The image tag equals the commit SHA.

<details>
<summary><b>Hints</b></summary>

Use `curl -fsS --retry 5 --retry-connection-refused localhost:8080/healthz` to give the app time to start.

</details>

**Bonus:** Switch to `docker/build-push-action@v6` with `load: true`.

### Assignment 6. Push to GitHub Container Registry 🟢

**Objective:** Publish an image to ghcr.io with the built-in token.

**Tasks**

1. Add `permissions: packages: write` to the job.
2. Log in with `docker/login-action@v3` using `registry: ghcr.io`, `username: ${{ github.actor }}`, `password: ${{ secrets.GITHUB_TOKEN }}`.
3. Push `ghcr.io/<owner>/<repo>:<sha>` and `:latest`.
4. Pull the image locally and run it.

**Expected outcome / acceptance criteria**

- The package appears under the repository's Packages.
- `docker run` of the pulled image works.

<details>
<summary><b>Hints</b></summary>

The image name must be lowercase. Use `${{ github.repository }}` and lowercase it in a shell step if needed.

</details>

**Bonus:** Use `docker/metadata-action@v5` to generate tags automatically (branch, sha, semver).

### Assignment 7. Matrix build 🟢

**Objective:** Test across several runtime versions in parallel.

**Tasks**

1. Add `strategy.matrix` with three Node (or Python) versions.
2. Pass the version into `setup-node`.
3. Make one version intentionally fail and observe the others.
4. Set `fail-fast: false` and compare behavior.

**Expected outcome / acceptance criteria**

- Three parallel jobs appear in the run graph.
- With `fail-fast: false` the other versions complete even if one fails.

<details>
<summary><b>Hints</b></summary>

Reference the value as `${{ matrix.node }}`. The job name can include it for readable logs.

</details>

**Bonus:** Add an `include` entry that tests an experimental version with `continue-on-error: true`.

### Assignment 8. Hello Jenkins pipeline 🟢

**Objective:** Run a declarative pipeline in Jenkins.

**Tasks**

1. Start Jenkins with `docker run -p 8080:8080 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts-jdk17`.
2. Complete the setup wizard and install the suggested plugins.
3. Create a Pipeline job that reads a `Jenkinsfile` from your Git repository.
4. Write stages `Build`, `Test`, `Package` that echo messages and archive a text file.

**Expected outcome / acceptance criteria**

- The Stage View shows three green stages.
- The archived file is downloadable from the build page.

<details>
<summary><b>Hints</b></summary>

Initial admin password: `docker exec <container> cat /var/jenkins_home/secrets/initialAdminPassword`. Use `pipeline { agent any ... }`.

</details>

**Bonus:** Add a `post { always { cleanWs() } }` block (Workspace Cleanup plugin).

### Assignment 9. Hello GitLab CI 🟢

**Objective:** Run a pipeline with stages in GitLab.

**Tasks**

1. Create a project on gitlab.com.
2. Write `.gitlab-ci.yml` with stages `build`, `test`, `deploy`.
3. Make `test` run two jobs in parallel.
4. Add an artifact produced in `build` and consumed in `test`.

**Expected outcome / acceptance criteria**

- Pipeline graph shows build then two parallel test jobs then deploy.
- The test job can read the build artifact.

<details>
<summary><b>Hints</b></summary>

Artifacts from earlier stages are downloaded automatically by later stages. Use `artifacts: paths: [dist/]` and `expire_in: 1 week`.

</details>

**Bonus:** Restrict `deploy` with `rules: - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH`.

### Assignment 10. Status badge and branch protection 🟢

**Objective:** Make CI results enforceable.

**Tasks**

1. Add the workflow status badge to your README.
2. Enable branch protection on `main`: require a pull request and the lint-test check.
3. Open a PR with a failing test and try to merge it.
4. Fix the test and merge.

**Expected outcome / acceptance criteria**

- The merge button is blocked while the check fails.
- The README badge turns green after the merge.

<details>
<summary><b>Hints</b></summary>

The required check name is the job name (or `workflow / job` for reusable workflows). A check must have run at least once to be selectable.

</details>

**Bonus:** Enable 'Require branches to be up to date' and observe how a second PR must rebase.

---

## 🟡 Level 2: Intermediate (Assignments 11-20)

### Assignment 11. Reusable workflow 🟡

**Objective:** Remove duplication using `workflow_call`.

**Tasks**

1. Create `.github/workflows/test.yml` with `on: workflow_call` and inputs `node-version` and `working-directory`.
2. Call it from a second workflow for two directories of a monorepo.
3. Return an output (for example the coverage percent) and print it in the caller.
4. Document inputs in a comment header.

**Expected outcome / acceptance criteria**

- Both callers reuse the same logic.
- The caller prints the output value.

<details>
<summary><b>Hints</b></summary>

Call with `uses: ./.github/workflows/test.yml` and `with:`. Pass secrets with `secrets: inherit` or explicitly.

</details>

**Bonus:** Move it to a central repository and call it as `org/ci-workflows/.github/workflows/test.yml@v1`.

### Assignment 12. Concurrency and path filters 🟡

**Objective:** Avoid wasted runs in a monorepo.

**Tasks**

1. Create `services/api` and `services/web` directories.
2. Use `on.push.paths` so each workflow runs only for its folder.
3. Add `concurrency` with `cancel-in-progress` for pull requests.
4. Push rapid commits and show cancelled runs.

**Expected outcome / acceptance criteria**

- Changing only `services/web` does not trigger the API workflow.
- Superseded PR runs show 'Cancelled'.

<details>
<summary><b>Hints</b></summary>

Do not cancel runs on `main` that might be mid-deploy: `cancel-in-progress: ${{ github.event_name == 'pull_request' }}`. Remember path filters and required checks interact: a skipped workflow leaves a required check pending.

</details>

**Bonus:** Use `dorny/paths-filter` to compute changed folders and a final `ci-ok` job that required checks can point to.

### Assignment 13. Docker layer cache in CI 🟡

**Objective:** Speed up image builds with BuildKit cache.

**Tasks**

1. Write a Dockerfile that copies the dependency manifest before the source.
2. Use `docker/setup-buildx-action@v3` and `docker/build-push-action@v6` with `cache-from: type=gha` and `cache-to: type=gha,mode=max`.
3. Run twice with a source-only change and note cached layers in the log.
4. Record build durations before and after.

**Expected outcome / acceptance criteria**

- Log lines show `CACHED` for dependency layers.
- A source-only change skips the dependency install.

<details>
<summary><b>Hints</b></summary>

Cache hit requires identical layer inputs. Look for `CACHED [stage N/M]` in the build output.

</details>

**Bonus:** Switch to a registry cache (`type=registry,ref=...:buildcache`) and compare.

### Assignment 14. Trivy gate 🟡

**Objective:** Block vulnerable images and handle exceptions.

**Tasks**

1. Add a Trivy step scanning the built image with `severity: HIGH,CRITICAL` and `exit-code: 1`.
2. Use an outdated base image and confirm the failure.
3. Update the base image and confirm success.
4. Add a `.trivyignore` entry for one CVE with a comment and expiry date, and explain when this is acceptable.

**Expected outcome / acceptance criteria**

- Old base image fails the job with a CVE table.
- The ignored CVE no longer fails the build.

<details>
<summary><b>Hints</b></summary>

Use `ignore-unfixed: true` for actionable results. Scan the final image, not only the source tree. Also try `trivy fs .` and `trivy config .` for IaC.

</details>

**Bonus:** Upload results as SARIF (`format: sarif`) to the Security tab with `github/codeql-action/upload-sarif@v3`.

### Assignment 15. CodeQL and Dependabot 🟡

**Objective:** Add code scanning and automated dependency updates.

**Tasks**

1. Enable CodeQL with a workflow for your language.
2. Introduce an obvious SQL injection or `eval` of user input and confirm an alert.
3. Add `.github/dependabot.yml` for the package ecosystem, `docker` and `github-actions`, weekly.
4. Merge one Dependabot PR.

**Expected outcome / acceptance criteria**

- The alert appears under Security, Code scanning.
- Dependabot opens PRs including one for an action version.

<details>
<summary><b>Hints</b></summary>

CodeQL needs `security-events: write`. For compiled languages set a build mode. Dependabot config is YAML with `version: 2` and `updates:` entries.

</details>

**Bonus:** Group Dependabot updates and auto-merge patch bumps only when checks pass.

### Assignment 16. AWS OIDC from GitHub 🟡

**Objective:** Authenticate to AWS without stored keys.

**Tasks**

1. Create the IAM OIDC provider for `token.actions.githubusercontent.com`.
2. Create a role whose trust policy matches `repo:ORG/REPO:ref:refs/heads/main` with `aud` `sts.amazonaws.com`.
3. Use `aws-actions/configure-aws-credentials@v4` and run `aws sts get-caller-identity`.
4. Run from a feature branch and capture the failure message.

**Expected outcome / acceptance criteria**

- Main branch run prints the assumed role ARN.
- Feature branch run fails with `Not authorized to perform sts:AssumeRoleWithWebIdentity`.

<details>
<summary><b>Hints</b></summary>

The job needs `permissions: id-token: write`. If it declares an `environment`, the `sub` becomes `repo:ORG/REPO:environment:NAME`.

</details>

**Bonus:** Restrict by `job_workflow_ref` so only a specific reusable workflow can assume the role.

### Assignment 17. Environments and approvals 🟡

**Objective:** Gate production behind a human.

**Tasks**

1. Create `staging` and `production` environments.
2. Add required reviewers and a deployment branch restriction (`main`) to `production`.
3. Add environment-specific secrets with the same name in each.
4. Write a workflow whose second job targets `production` and prints which secret scope it sees.

**Expected outcome / acceptance criteria**

- The production job waits for approval.
- Each job reads its own environment's secret value.

<details>
<summary><b>Hints</b></summary>

Use `environment: production` at job level. Wait timers and required reviewers are configured per environment. Private repos need a paid plan for reviewers.

</details>

**Bonus:** Add a wait timer and use the deployment URL in the environment declaration.

### Assignment 18. Declarative Jenkinsfile with parallel stages and credentials 🟡

**Objective:** Build a realistic Jenkins pipeline.

**Tasks**

1. Write a Jenkinsfile with `options { timeout, disableConcurrentBuilds }`.
2. Run unit tests and lint in a `parallel` block.
3. Store a token as a Jenkins credential and use `withCredentials([string(...)])`.
4. Verify the token is masked in the console log.

**Expected outcome / acceptance criteria**

- Stage View shows parallel branches.
- Console output shows `****` for the token.

<details>
<summary><b>Hints</b></summary>

Credential IDs are referenced by string. Use `parallel { stage('Unit') {...} stage('Lint') {...} }` inside a stage.

</details>

**Bonus:** Run the build on a Docker agent: `agent { docker { image 'node:22-alpine' } }`.

### Assignment 19. Jenkins shared library 🟡

**Objective:** Reuse pipeline code across jobs.

**Tasks**

1. Create a Git repo with `vars/buildApp.groovy` defining `def call(Map cfg)`.
2. Register it in Manage Jenkins, System, Global Trusted Pipeline Libraries.
3. Load it with `@Library('shared@main') _` and call `buildApp(name: 'demo')` from two Jenkinsfiles.
4. Change the library and see both pipelines change behavior.

**Expected outcome / acceptance criteria**

- Two different repos use the same library function.
- A library change alters both builds without touching their Jenkinsfiles.

<details>
<summary><b>Hints</b></summary>

Files in `vars/` become global steps named after the file. Pin to a tag or branch you control to avoid surprise breakage.

</details>

**Bonus:** Add `resources/` with a shell script loaded through `libraryResource`.

### Assignment 20. GitLab CI rules, needs and environments 🟡

**Objective:** Model a branching pipeline in GitLab.

**Tasks**

1. Create jobs `lint`, `test`, `build`, `deploy-staging`, `deploy-prod`.
2. Use `needs:` so `build` starts as soon as `test` passes (DAG).
3. Use `rules:` so deploy jobs run only on the default branch, and `deploy-prod` is `when: manual`.
4. Define `environment:` with a name and URL for both deploys.

**Expected outcome / acceptance criteria**

- The pipeline graph shows DAG dependencies.
- Production deploy waits for a manual click; Environments page lists both.

<details>
<summary><b>Hints</b></summary>

`rules` replace `only/except`. A manual job on a protected environment can also require approvals. Merge request pipelines need `if: $CI_PIPELINE_SOURCE == "merge_request_event"`.

</details>

**Bonus:** Add `include: template: Security/SAST.gitlab-ci.yml` and review findings.

---

## 🔴 Level 3: Advanced (Assignments 21-25)

### Assignment 21. Full pipeline to EKS 🔴

**Objective:** Deliver the complete lab as your own project.

**Tasks**

1. Complete the [hands-on lab](./hands-on-lab.md) with your own application.
2. Add a Helm chart or Kustomize overlays instead of `envsubst`.
3. Make prod deploy use the image digest instead of the tag (`image@sha256:...`).
4. Document the architecture with a Mermaid diagram in your repo.

**Expected outcome / acceptance criteria**

- Dev deploys automatically; prod needs approval and uses a digest.
- No long-lived cloud credentials exist anywhere in the repo settings.

<details>
<summary><b>Hints</b></summary>

Get the digest from `docker/build-push-action` output `digest`. Pin by digest to guarantee prod runs exactly what was scanned.

</details>

**Bonus:** Add Argo CD: the pipeline commits the new image reference to a config repo and Argo syncs it.

### Assignment 22. Canary with automated rollback 🔴

**Objective:** Deliver progressively and roll back on bad metrics.

**Tasks**

1. Install Argo Rollouts and Prometheus on a test cluster.
2. Convert the Deployment to a `Rollout` with steps `setWeight: 10`, `pause`, `setWeight: 50`, `pause`.
3. Add an `AnalysisTemplate` that queries the HTTP 5xx ratio from Prometheus.
4. Deploy a version that returns 500 errors and show the rollout aborting automatically.

**Expected outcome / acceptance criteria**

- Healthy versions progress to 100 percent.
- The faulty version is aborted and traffic returns to the stable ReplicaSet.

<details>
<summary><b>Hints</b></summary>

Use `kubectl argo rollouts get rollout app --watch`. A `successCondition` such as `result[0] < 0.05` drives pass or fail. Without a traffic router, weights are approximated by pod counts.

</details>

**Bonus:** Add header-based routing for internal testers through an ingress or service mesh.

### Assignment 23. Supply chain hardening 🔴

**Objective:** Make the pipeline resistant to tampering.

**Tasks**

1. Pin every third-party action to a full commit SHA and add Dependabot to update them.
2. Set top-level `permissions: {}` and grant per-job minimum scopes.
3. Generate an SBOM with Syft or `docker buildx --sbom=true`, and sign the image with Cosign keyless.
4. Verify the signature with `cosign verify` constrained by certificate identity and OIDC issuer.

**Expected outcome / acceptance criteria**

- `cosign verify` succeeds only for images built by your workflow.
- A scan of the workflows shows no unpinned third-party actions.

<details>
<summary><b>Hints</b></summary>

Keyless signing needs `id-token: write`. Verify with `--certificate-identity-regexp` and `--certificate-oidc-issuer https://token.actions.githubusercontent.com`.

</details>

**Bonus:** Enforce signatures at admission with Kyverno `verifyImages` or Sigstore policy-controller.

### Assignment 24. Self-hosted autoscaling runners 🔴

**Objective:** Run CI on your own Kubernetes capacity.

**Tasks**

1. Install Actions Runner Controller (gha-runner-scale-set) with Helm on a test cluster.
2. Register a scale set against your repository using a GitHub App or PAT.
3. Point a workflow at it with `runs-on: <scale-set-name>` and show runner pods appearing and disappearing.
4. Document how you would isolate untrusted pull requests.

**Expected outcome / acceptance criteria**

- Runner pods scale from zero on demand and terminate after the job.
- Your isolation notes cover fork PRs, network egress and ephemeral runners.

<details>
<summary><b>Hints</b></summary>

Scale sets use `minRunners` and `maxRunners`. Never run fork PRs on persistent self-hosted runners; use ephemeral runners with no cloud credentials.

</details>

**Bonus:** Mount a BuildKit cache volume or run a registry cache to recover build speed.

### Assignment 25. Pipeline observability and DORA metrics 🔴

**Objective:** Measure the delivery system itself.

**Tasks**

1. Export workflow run data with the GitHub API (`gh api repos/ORG/REPO/actions/runs`) into a CSV or a time-series store.
2. Compute deployment frequency, lead time for changes, change failure rate, and mean time to restore for the last 30 days.
3. Build a dashboard showing p50 and p90 pipeline duration and queue time.
4. Identify the slowest job and cut its duration by at least 30 percent, documenting what you changed.

**Expected outcome / acceptance criteria**

- The dashboard shows all four DORA metrics.
- A before and after comparison proves the speed-up.

<details>
<summary><b>Hints</b></summary>

Lead time is first commit to production deploy. Count a failure as a deployment followed by a rollback or hotfix. Queue time is `started_at - created_at` on the job.

</details>

**Bonus:** Alert when pipeline p90 duration regresses by 20 percent week over week.

[⬅ Back to CI/CD](./README.md) · [🏠 Home](../README.md)

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
