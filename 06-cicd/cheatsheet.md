# CI/CD — Cheatsheet

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

> Dense reference for GitHub Actions, Jenkins, GitLab CI, Docker builds, scanning, ECR/OIDC and Kubernetes deploys.

[⬅ Back to CI/CD](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [GitHub Actions: syntax](#github-actions-syntax)
- [GitHub Actions: expressions and contexts](#github-actions-expressions-and-contexts)
- [GitHub Actions: recipes](#github-actions-recipes)
- [GitHub CLI for pipelines](#github-cli-for-pipelines)
- [Jenkins](#jenkins)
- [GitLab CI](#gitlab-ci)
- [Docker build and cache](#docker-build-and-cache)
- [Scanning and quality gates](#scanning-and-quality-gates)
- [AWS: ECR and OIDC](#aws-ecr-and-oidc)
- [Kubernetes deploy and rollback](#kubernetes-deploy-and-rollback)
- [Deployment strategies](#deployment-strategies)
- [Tool comparison](#tool-comparison)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## GitHub Actions: syntax

| Key | Purpose | Example |
|---|---|---|
| `on` | Triggers | `on: [push, pull_request]` |
| `on.push.branches` / `tags` / `paths` | Filter | `branches: [main]`, `tags: ['v*']`, `paths: ['src/**']` |
| `on.workflow_dispatch.inputs` | Manual run with inputs | `inputs: { env: { type: choice, options: [dev, prod] } }` |
| `on.schedule` | Cron (UTC, default branch only) | `- cron: '0 2 * * 1-5'` |
| `on.workflow_call` | Reusable workflow | `inputs:`, `secrets:`, `outputs:` |
| `permissions` | GITHUB_TOKEN scopes | `contents: read`, `id-token: write`, `packages: write` |
| `concurrency` | Serialize or cancel | `group: ${{ github.workflow }}-${{ github.ref }}` |
| `jobs.<id>.needs` | Dependencies | `needs: [build, test]` |
| `jobs.<id>.if` | Conditional | `if: github.ref == 'refs/heads/main'` |
| `jobs.<id>.environment` | Protection rules and env secrets | `environment: prod` |
| `jobs.<id>.strategy.matrix` | Fan-out | `matrix: { node: [20, 22] }` |
| `jobs.<id>.outputs` | Pass data to later jobs | `outputs: { tag: ${{ steps.v.outputs.tag }} }` |
| `jobs.<id>.timeout-minutes` | Kill hung jobs | `timeout-minutes: 15` |
| `jobs.<id>.container` / `services` | Run job in container, sidecars | `services: { db: { image: postgres:16 } }` |
| `defaults.run.shell` | Strict shell | `shell: bash` (adds `-eo pipefail`) |

Set outputs and env between steps:

```bash
echo "tag=${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"   # step output
echo "VERSION=1.2.3"        >> "$GITHUB_ENV"      # env for later steps
echo "$HOME/bin"            >> "$GITHUB_PATH"     # PATH addition
echo "### Summary"          >> "$GITHUB_STEP_SUMMARY"
```

## GitHub Actions: expressions and contexts

| Expression | Meaning |
|---|---|
| `${{ github.sha }}` / `github.ref` / `github.ref_name` | Commit SHA, full ref, short ref |
| `${{ github.event_name }}` | `push`, `pull_request`, ... |
| `${{ github.actor }}` / `github.repository` | User, `owner/repo` |
| `${{ secrets.NAME }}` / `vars.NAME` | Secret, configuration variable |
| `${{ needs.job.outputs.x }}` | Output from earlier job |
| `${{ matrix.node }}` | Current matrix value |
| `${{ runner.os }}` | `Linux`, `Windows`, `macOS` |
| `${{ hashFiles('**/package-lock.json') }}` | Cache key from lockfile |
| `contains()`, `startsWith()`, `format()`, `toJSON()` | Built-in functions |
| `success()`, `failure()`, `always()`, `cancelled()` | Status check functions for `if:` |

Security reminder: never interpolate untrusted values (`github.event.pull_request.title`, branch names) directly into `run:`. Pass them through `env:` and quote the shell variable.

## GitHub Actions: recipes

Cache and Node setup:

```yaml
- uses: actions/setup-node@v4
  with: { node-version: 22, cache: npm }
```

OIDC to AWS:

```yaml
permissions: { id-token: write, contents: read }
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/gha-ecr-push
      aws-region: ap-south-1
```

Build with cache and push:

```yaml
- uses: docker/setup-buildx-action@v3
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ${{ env.IMAGE }}:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

Only run when a path changed, cancel old PR runs, and gate with `if`:

```yaml
on:
  pull_request:
    paths: ['services/api/**']
concurrency:
  group: api-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

Reusable workflow call:

```yaml
jobs:
  test:
    uses: org/ci/.github/workflows/test.yml@v1
    with: { node-version: 22 }
    secrets: inherit
```

Pin actions by SHA (add the tag as a comment, let Dependabot update it):

```yaml
- uses: actions/checkout@<full-40-char-commit-sha> # v4.2.2
```

## GitHub CLI for pipelines

```bash
gh workflow list
gh workflow run ci-cd.yml -f env=dev        # dispatch with input
gh run list --workflow ci-cd.yml --limit 5
gh run watch                                # follow the latest run
gh run view <id> --log-failed               # only failing step logs
gh run rerun <id> --failed                  # re-run failed jobs
gh run cancel <id>
gh secret set NAME < file                   # set a secret
gh variable set NAME --body value
gh pr checks --watch                        # PR checks live
gh api repos/ORG/REPO/actions/caches        # inspect caches
```

## Jenkins

Declarative skeleton:

```groovy
pipeline {
  agent { docker { image 'node:22-alpine' } }
  options { timeout(time: 30, unit: 'MINUTES'); disableConcurrentBuilds(abortPrevious: true) }
  environment { REGISTRY = 'registry.example.com' }
  stages {
    stage('Test') {
      parallel {
        stage('Unit') { steps { sh 'npm test' } }
        stage('Lint') { steps { sh 'npm run lint' } }
      }
    }
    stage('Deploy') {
      when { branch 'main' }
      steps {
        input message: 'Deploy to prod?', submitter: 'release-managers'
        withCredentials([string(credentialsId: 'api-token', variable: 'TOKEN')]) {
          sh './deploy.sh'
        }
      }
    }
  }
  post {
    always  { junit 'reports/*.xml' }
    failure { echo 'notify the team' }
  }
}
```

| Item | Notes |
|---|---|
| `agent any` / `label 'x'` / `docker` / `kubernetes` | Where to run |
| `when { branch 'main'; changeset 'src/**'; expression { ... } }` | Conditional stages |
| `post { always, success, failure, unstable, cleanup }` | Post conditions |
| `stash` / `unstash` | Move files between agents |
| `archiveArtifacts`, `junit` | Keep artifacts, publish tests |
| `@Library('shared@v1') _` | Load a shared library |
| `withCredentials([usernamePassword(...), string(...), file(...)])` | Credentials, masked in logs |
| `waitForQualityGate abortPipeline: true` | SonarQube gate |
| `Replay` button | Edit and re-run a pipeline without a commit |

Useful URLs and CLI:

```bash
# Syntax check without running
curl -X POST -u user:token -F "jenkinsfile=<Jenkinsfile" http://jenkins:8080/pipeline-model-converter/validate
java -jar jenkins-cli.jar -s http://jenkins:8080 -auth user:token list-jobs
java -jar jenkins-cli.jar -s http://jenkins:8080 -auth user:token build my-job -s -v
# Configuration as Code reload: Manage Jenkins -> Configuration as Code
```

## GitLab CI

```yaml
stages: [lint, test, build, deploy]

default:
  image: node:22-alpine
  cache:
    key:
      files: [package-lock.json]
    paths: [node_modules/]

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

lint:
  stage: lint
  script: [npm ci, npm run lint]

test:
  stage: test
  script: [npm ci, npm test]
  coverage: '/All files.*?\|\s*([\d.]+)/'
  artifacts:
    when: always
    reports:
      junit: junit.xml

build:
  stage: build
  image: docker:27
  services: [docker:27-dind]
  needs: [test]
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    - docker build -t "$IMAGE" .
    - docker push "$IMAGE"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-prod:
  stage: deploy
  environment: { name: production, url: https://app.example.com }
  when: manual
  needs: [build]
  script: ./deploy.sh "$IMAGE"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

| Keyword | Purpose |
|---|---|
| `rules: if / changes / exists / when` | Conditional jobs (replaces `only/except`) |
| `needs: [job]` | DAG, start without waiting for stage |
| `extends: .template` | Inherit hidden job config |
| `include: local / project / template / component` | Reuse config |
| `artifacts: paths, expire_in, reports` | Pass files, show test reports |
| `cache: key, paths, policy` | Speed up installs |
| `environment: name, url, action: stop` | Deployments and review apps |
| `id_tokens: { AWS_TOKEN: { aud: sts.amazonaws.com } }` | OIDC JWT for cloud auth |
| `parallel: matrix:` | Fan-out |
| `resource_group: prod` | Serialize deploys |
| `trigger:` | Child and multi-project pipelines |
| `retry`, `timeout`, `allow_failure`, `interruptible` | Resilience and cost |

Predefined variables: `CI_COMMIT_SHA`, `CI_COMMIT_SHORT_SHA`, `CI_COMMIT_REF_NAME`, `CI_DEFAULT_BRANCH`, `CI_PIPELINE_SOURCE`, `CI_REGISTRY_IMAGE`, `CI_MERGE_REQUEST_IID`, `CI_JOB_TOKEN`.

## Docker build and cache

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t app:1.0 --push .
docker buildx build --cache-from type=registry,ref=reg/app:cache \
                    --cache-to   type=registry,ref=reg/app:cache,mode=max -t app:sha .
docker buildx build --cache-to type=gha,mode=max --cache-from type=gha .   # GitHub Actions
docker buildx build --sbom=true --provenance=true --push -t reg/app:sha .
DOCKER_BUILDKIT=1 docker build --secret id=npmrc,src=$HOME/.npmrc .         # RUN --mount=type=secret
```

| Dockerfile rule | Why |
|---|---|
| Copy lockfile and install before copying source | Dependency layer stays cached |
| Multi-stage build, copy only runtime files | Smaller image, fewer CVEs |
| Pin base image tag (or digest) | Reproducible builds |
| `USER node` / non-root user | Reduce blast radius |
| `.dockerignore` for `.git`, `node_modules` | Smaller context, no secret leaks |
| No secrets in `ARG`/`ENV` | They persist in image history |

## Scanning and quality gates

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 app:sha
trivy image --format sarif -o trivy.sarif app:sha
trivy fs --scanners vuln,secret,misconfig .
trivy config k8s/                        # Kubernetes and Terraform misconfig
trivy image --input image.tar            # scan a tarball (no registry needed)
syft app:sha -o spdx-json > sbom.json    # SBOM
grype sbom:sbom.json --fail-on high
cosign sign --yes reg/app@sha256:<digest>
cosign verify reg/app@sha256:<digest> \
  --certificate-identity-regexp 'https://github.com/ORG/REPO/.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
hadolint Dockerfile
gitleaks detect --source .
```

SonarQube properties that matter:

```properties
sonar.projectKey=my-app
sonar.sources=src
sonar.tests=test
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.python.coverage.reportPaths=coverage.xml
sonar.qualitygate.wait=true
```

| Gate type | Typical rule |
|---|---|
| Tests | All pass, flaky tests quarantined with a ticket |
| Coverage on new code | At least 80 percent |
| Static analysis | No new bugs or vulnerabilities |
| Image scan | No fixed HIGH or CRITICAL |
| IaC and policy | `trivy config`, OPA or Kyverno checks pass |
| Secrets | `gitleaks` finds nothing |

## AWS: ECR and OIDC

```bash
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
aws ecr create-repository --repository-name app --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true
aws ecr describe-images --repository-name app --query 'sort_by(imageDetails,&imagePushedAt)[-5:].imageTags'
aws ecr put-lifecycle-policy --repository-name app --lifecycle-policy-text file://lifecycle.json
aws ecr batch-delete-image --repository-name app --image-ids imageTag=old
aws eks update-kubeconfig --name my-cluster --region ap-south-1
aws sts get-caller-identity                     # who am I right now?
```

OIDC trust policy conditions:

| Token claim `sub` format | When |
|---|---|
| `repo:ORG/REPO:ref:refs/heads/main` | Job on `main`, no environment |
| `repo:ORG/REPO:environment:prod` | Job that declares `environment: prod` |
| `repo:ORG/REPO:pull_request` | Pull request event |
| `repo:ORG/REPO:ref:refs/tags/v1.0.0` | Tag push |

Use `StringEquals` for exact matches and `StringLike` with wildcards sparingly. Always check `aud` equals `sts.amazonaws.com`.

EKS access entries (instead of editing `aws-auth`):

```bash
aws eks create-access-entry --cluster-name c --principal-arn <role-arn> --type STANDARD
aws eks associate-access-policy --cluster-name c --principal-arn <role-arn> \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=dev
```

## Kubernetes deploy and rollback

```bash
kubectl apply -f k8s/ -n dev
kubectl set image deployment/app app=reg/app:<sha> -n dev
kubectl rollout status deployment/app -n dev --timeout=120s
kubectl rollout history deployment/app -n dev
kubectl rollout undo deployment/app -n dev                  # previous revision
kubectl rollout undo deployment/app -n dev --to-revision=3
kubectl rollout pause|resume deployment/app -n dev
kubectl diff -f k8s/                                        # preview changes
kubectl apply --dry-run=server -f k8s/                      # server-side validation
kubectl auth can-i patch deployments -n prod --as system:serviceaccount:ci:deployer
helm upgrade --install app ./chart -n prod --atomic --timeout 5m --set image.tag=<sha>
helm rollback app 2 -n prod
helm diff upgrade app ./chart -n prod                       # helm-diff plugin
kubectl argo rollouts get rollout app --watch               # Argo Rollouts
```

| Failure pattern | First command |
|---|---|
| Rollout stuck | `kubectl rollout status` then `kubectl describe pod` |
| `ImagePullBackOff` | `kubectl describe pod` (auth, tag typo, node role) |
| `CrashLoopBackOff` | `kubectl logs --previous` |
| Readiness never passes | Check probe path and port, `kubectl get endpoints` |
| `Forbidden` from CI | `kubectl auth can-i`, review Role and access entry |

## Deployment strategies

| Strategy | How | Rollback speed | Cost | Use when |
|---|---|---|---|---|
| Recreate | Stop old, start new | Redeploy | Low, has downtime | Dev, singleton apps |
| Rolling | Replace pods gradually (`maxSurge`, `maxUnavailable`) | Minutes | Low | Default for stateless services |
| Blue/green | Two full stacks, switch traffic | Instant (flip back) | 2x capacity | Need instant rollback |
| Canary | Small percent first, analyze metrics | Fast (abort) | Low to medium | Risky changes, large user base |
| Feature flags | Deploy dark, enable per cohort | Instant (toggle) | Low | Decouple deploy from release |
| Shadow | Mirror traffic to new version | n/a | Medium | Validate without user impact |

Database changes: use expand and contract migrations so old and new code both work during the rollout.

## Tool comparison

| | GitHub Actions | Jenkins | GitLab CI |
|---|---|---|---|
| Config | YAML in `.github/workflows` | Groovy `Jenkinsfile` | YAML `.gitlab-ci.yml` |
| Runner | Hosted or self-hosted runner | Agent (static, Docker, K8s pod) | Runner (shell, docker, k8s) |
| Reuse | Reusable workflows, composite actions | Shared libraries | `include`, `extends`, components |
| Secrets | Repo, env, org secrets, OIDC | Credentials store, plugins | CI/CD variables (masked, protected), OIDC |
| Approval | Environment reviewers | `input` step | Protected environments, manual jobs |
| Ecosystem | Marketplace actions | 1800+ plugins | Built-in DevSecOps features |
| Weak spot | Third-party action risk | Plugin and upgrade burden | Runner management at scale |

## Top 15 interview one-liners

1. **CI vs CD:** CI integrates and verifies every change automatically; continuous delivery keeps `main` always releasable behind a manual gate; continuous deployment removes the gate.
2. **Build once, deploy many:** one immutable image promoted through environments, with only config differing.
3. **Why tag with the Git SHA:** it ties a running container to an exact commit, so audits and rollbacks are exact; `latest` is mutable and ambiguous.
4. **OIDC beats stored keys:** the runner exchanges a short-lived signed token for temporary credentials, so there is nothing long-lived to leak or rotate.
5. **OIDC `sub` claim gotcha:** jobs with an `environment` get `repo:ORG/REPO:environment:NAME`, not the branch form, so trust policies must match what the job really presents.
6. **Least-privilege GITHUB_TOKEN:** set `permissions: contents: read` at the top and grant extra scopes per job.
7. **Pin third-party actions by commit SHA:** tags can be moved; a SHA cannot, and Dependabot can keep it current.
8. **Quality gate:** an automated pass or fail on tests, coverage, static analysis and scans that blocks promotion, enforced by branch protection so it cannot be skipped.
9. **Speeding up pipelines:** cache dependencies, order Dockerfile layers, use BuildKit cache, parallelize independent jobs, path-filter monorepos, and cancel superseded runs.
10. **Scan before push:** a vulnerable image should never reach the registry; fail on fixed HIGH and CRITICAL and track exceptions with expiry dates.
11. **Rollback:** redeploy the previous known-good SHA; `kubectl rollout undo` or `helm rollback` for emergencies; in GitOps, revert the commit.
12. **Blue/green vs canary:** blue/green gives instant rollback at double capacity; canary limits blast radius with real traffic and metric analysis.
13. **Jenkins vs GitHub Actions:** Jenkins is self-hosted and infinitely flexible but you operate controller, agents and plugins; Actions is managed and integrated but ties you to GitHub.
14. **Secrets in pipelines:** inject at runtime from a secret store or OIDC, scope per environment, never bake into images, and remember masking is not a security boundary.
15. **Flaky tests:** quarantine with a tracked ticket, fix root causes (timing, shared state, network), and never fix flakiness by blanket retries.

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
