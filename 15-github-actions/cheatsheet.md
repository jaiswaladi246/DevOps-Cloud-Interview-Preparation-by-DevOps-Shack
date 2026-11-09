# GitHub Actions — Cheatsheet

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

> Dense reference for daily use and interview prep. GitHub.com Actions in 2025–2026: artifacts v4, `node24` actions, OIDC to AWS/Azure/GCP, ARC runner scale sets, GitHub CLI 2.x.

[⬅ Back to GitHub Actions](./README.md) · [🏠 Home](../README.md)

## Contents

[Workflow skeleton](#workflow-syntax-skeleton) · [Key reference](#workflow-keys-at-a-glance) · [Triggers](#triggers) · [Contexts](#contexts) · [Expressions](#expressions-and-functions) · [Workflow commands and files](#workflow-commands-and-environment-files) · [Matrix](#matrix) · [Concurrency](#concurrency) · [Permissions](#permissions) · [Secrets, vars, environments](#secrets-variables-and-environments) · [Reusable workflows vs composite actions](#reusable-workflows-vs-composite-actions) · [Caching](#caching) · [Artifacts](#artifacts) · [OIDC AWS](#oidc-to-aws) · [OIDC Azure](#oidc-to-azure) · [OIDC GCP](#oidc-to-gcp) · [Hardening](#security-hardening) · [Runners](#runners) · [gh CLI](#gh-cli) · [Debugging](#debugging) · [Errors](#common-errors) · [Top 15](#top-15-interview-one-liners)

---

## Workflow syntax skeleton

```yaml
name: CI                                         # shown in the Actions tab
run-name: CI ${{ github.ref_name }} by @${{ github.actor }}

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
    paths-ignore: ['docs/**', '**.md']
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, staging, prod]
        default: dev
      dry_run:
        type: boolean
        default: true

permissions:
  contents: read                                 # read-only default for every job

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

env:
  NODE_VERSION: '22'

defaults:
  run:
    shell: bash                                  # bash --noprofile --norc -eo pipefail {0}

jobs:
  test:
    name: test (node ${{ matrix.node }})
    runs-on: ubuntu-latest
    timeout-minutes: 15
    strategy:
      fail-fast: false
      matrix:
        node: [20, 22, 24]
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm
      - run: npm ci
      - run: npm test

  build:
    needs: test
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write                            # OIDC
    outputs:
      image: ${{ steps.meta.outputs.image }}
    steps:
      - uses: actions/checkout@v4
      - id: meta
        run: echo "image=app:sha-${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: prod
      url: https://app.example.com
    concurrency:
      group: deploy-prod
      cancel-in-progress: false
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image }}"
```

---

## Workflow keys at a glance

| Key | Level | Notes |
|---|---|---|
| `on` | workflow | Events and filters (`branches`, `tags`, `paths`, `types`) |
| `permissions` | workflow, job | Unlisted scopes become `none` |
| `env` | workflow, job, step | Most specific wins; not usable in `jobs.<id>.if` (use `vars`) |
| `defaults.run` | workflow, job | `shell`, `working-directory` for `run` steps |
| `concurrency` | workflow, job | One running + one pending per group |
| `runs-on` | job | Label, array of labels, or `{ group, labels }` |
| `needs` | job | Job dependencies; read `needs.<id>.outputs` / `.result` |
| `if` | job, step | Implicit `success() &&` unless a status function is used |
| `strategy` | job | `matrix`, `fail-fast` (default true), `max-parallel` |
| `environment` | job | Protection rules, env secrets and vars, deployment record |
| `outputs` | job | Map from step outputs; strings, 1 MB each |
| `container` / `services` | job | Run steps in a container, sidecars (Linux runners only) |
| `timeout-minutes` | job, step | Job default 360 |
| `continue-on-error` | job, step | Failure does not fail the job/run |
| `uses` (job) | job | Call a reusable workflow; no `steps`/`runs-on` allowed |
| `uses` / `run` | step | Action or shell command |
| `with` / `id` / `shell` / `working-directory` | step | Inputs, reference for outputs, shell, cwd |

---

## Triggers

| Event | Typical use | Watch out |
|---|---|---|
| `push` | CI on branches, releases on tags | Workflow file is read from the pushed commit |
| `pull_request` | PR CI | Runs on the merge commit; fork PRs get no secrets and a read-only token |
| `pull_request_target` | Label/comment on fork PRs | Runs **base** workflow with secrets and write token: never check out and run PR code |
| `workflow_dispatch` | Manual runs with typed inputs | File must be on the default branch to show in UI |
| `schedule` | Nightly jobs | UTC, default branch only, may be delayed, avoid minute 0, disabled after 60 days inactivity (public repos) |
| `workflow_call` | Reusable workflow | Inputs, secrets, outputs declared under it |
| `workflow_run` | Privileged follow-up after an unprivileged run | Runs from default branch with secrets; treat upstream artifacts as untrusted |
| `repository_dispatch` | External systems trigger a run | `POST /repos/{o}/{r}/dispatches` with `event_type` and `client_payload` |
| `merge_group` | Merge queue checks | Required checks must also run on `merge_group` |
| `release` | Publish on release | Use `types: [published]` |
| `issue_comment`, `issues`, `pull_request_review` | ChatOps, triage | Titles and bodies are attacker-controlled |

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]   # default: opened, synchronize, reopened
    branches: [main, 'release/**']
    paths: ['src/**', 'package*.json']
  schedule:
    - cron: '17 2 * * 1-5'            # 02:17 UTC, Mon–Fri
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]
  repository_dispatch:
    types: [deploy-request]
```

```bash
# fire repository_dispatch
gh api repos/OWNER/REPO/dispatches -f event_type=deploy-request -F 'client_payload[env]=staging'
```

Skip a push/PR run: put `[skip ci]`, `[ci skip]`, `[no ci]`, `[skip actions]` or `[actions skip]` in the commit message.

---

## Contexts

| Context | Common fields |
|---|---|
| `github` | `event_name`, `ref`, `ref_name`, `ref_type`, `sha`, `head_ref`, `base_ref`, `repository`, `repository_owner`, `actor`, `triggering_actor`, `run_id`, `run_number`, `run_attempt`, `workflow`, `job`, `server_url`, `event.*`, `token` |
| `env` | Values from `env:` at workflow, job, step level |
| `vars` | Configuration variables (org, repo, environment) |
| `secrets` | Secrets, plus `GITHUB_TOKEN` |
| `inputs` | `workflow_dispatch` / `workflow_call` inputs, typed |
| `needs` | `needs.<job>.result`, `needs.<job>.outputs.<name>` |
| `steps` | `steps.<id>.outputs.<name>`, `.outcome`, `.conclusion` |
| `matrix` / `strategy` | Current leg values; `strategy.job-index`, `strategy.job-total` |
| `runner` | `os`, `arch`, `name`, `temp`, `tool_cache`, `environment` (`github-hosted` / `self-hosted`) |
| `job` | `status`, `container.id`, `services.<id>.ports` |

Where contexts are available (common traps):

| Location | Allowed contexts |
|---|---|
| `jobs.<id>.if` | `github`, `needs`, `vars`, `inputs` (no `env`, no `secrets`) |
| `jobs.<id>.runs-on`, `concurrency`, `environment` | `github`, `needs`, `strategy`, `matrix`, `vars`, `inputs` |
| `steps.if` | above plus `steps`, `job`, `runner`, `env` (no `secrets`: map to `env` first) |
| `on.workflow_call.outputs.*.value` | `jobs` |

Default environment variables on every runner: `GITHUB_SHA`, `GITHUB_REF`, `GITHUB_REF_NAME`, `GITHUB_REPOSITORY`, `GITHUB_RUN_ID`, `GITHUB_WORKSPACE`, `GITHUB_EVENT_PATH`, `GITHUB_ACTOR`, `RUNNER_OS`, `RUNNER_TEMP`, `CI=true`.

---

## Expressions and functions

| Expression | Result |
|---|---|
| `${{ github.ref == 'refs/heads/main' }}` | String comparisons are case-insensitive |
| `${{ startsWith(github.ref, 'refs/tags/v') }}` | Tag check |
| `${{ contains(github.event.pull_request.labels.*.name, 'deploy') }}` | Object filter `.*` + `contains` |
| `${{ format('{0}-{1}', runner.os, matrix.node) }}` | String formatting |
| `${{ join(matrix.*, '-') }}` | Join array |
| `${{ toJSON(github.event) }}` | Dump context (great for debugging) |
| `${{ fromJSON(needs.plan.outputs.matrix) }}` | Parse JSON (dynamic matrix, booleans, numbers) |
| `${{ hashFiles('**/package-lock.json') }}` | SHA-256 of matching files, empty string if none |
| `${{ inputs.env \|\| 'dev' }}` | Default value (`\|\|` returns first truthy operand) |
| `${{ github.event_name == 'push' && 'prod' \|\| 'dev' }}` | Ternary idiom (breaks if the middle value is falsy) |
| `success()` / `failure()` / `cancelled()` / `always()` | Status checks; `!cancelled()` is the safe "run even on failure" |

Falsy values: `false`, `0`, `-0`, `""`, `null`. `if:` may omit `${{ }}`, but an expression that starts with `!` must be wrapped or quoted.

```yaml
- if: failure() && steps.test.outcome == 'failure'
  run: ./collect-logs.sh
- if: ${{ !cancelled() }}
  uses: actions/upload-artifact@v4
  with: { name: test-results, path: reports/ }
```

---

## Workflow commands and environment files

| Purpose | Command |
|---|---|
| Step output | `echo "version=1.4.0" >> "$GITHUB_OUTPUT"` |
| Env var for later steps | `echo "BUILD_DATE=$(date -u +%F)" >> "$GITHUB_ENV"` |
| Add to PATH for later steps | `echo "$HOME/.local/bin" >> "$GITHUB_PATH"` |
| Markdown run summary | `echo "### Deployed :rocket:" >> "$GITHUB_STEP_SUMMARY"` |
| Multiline value | `{ echo "notes<<GHA_EOF"; cat notes.md; echo "GHA_EOF"; } >> "$GITHUB_OUTPUT"` |
| Mask a runtime value | `echo "::add-mask::$TOKEN"` |
| Fold log lines | `echo "::group::Install"` … `echo "::endgroup::"` |
| Annotations | `echo "::error file=app.js,line=10,col=5,title=Lint::Unexpected var"` (`::warning`, `::notice`) |
| Debug line (shown with debug on) | `echo "::debug::cache key is $KEY"` |
| Stop processing commands | `echo "::stop-commands::$(uuidgen)"` … `echo "::<same-token>::"` |

`::set-output` and `::set-env` are deprecated/disabled: use the environment files.

---

## Matrix

```yaml
strategy:
  fail-fast: false
  max-parallel: 4
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [20, 22, 24]
    exclude:
      - os: macos-latest
        node: 20
    include:
      - os: ubuntu-latest           # adds a key to an existing leg
        node: 24
        coverage: true
      - os: ubuntu-24.04-arm        # adds a brand new leg
        node: 24
continue-on-error: ${{ matrix.experimental == true }}
```

Dynamic matrix from a previous job:

```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.s.outputs.services }}
    steps:
      - id: s
        run: echo 'services=["api","web"]' >> "$GITHUB_OUTPUT"
  build:
    needs: plan
    if: needs.plan.outputs.services != '[]'
    strategy:
      matrix:
        service: ${{ fromJSON(needs.plan.outputs.services) }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "building ${{ matrix.service }}"
```

Limits: 256 jobs per matrix per run. Fan-in gate for branch protection:

```yaml
ci-ok:
  needs: [lint, test, build]
  if: always()
  runs-on: ubuntu-latest
  steps:
    - env:
        NEEDS: ${{ toJSON(needs) }}
      run: echo "$NEEDS" | jq -e 'all(.[]; .result == "success" or .result == "skipped")'
```

---

## Concurrency

| Goal | Config |
|---|---|
| Cancel superseded PR runs | `group: ${{ github.workflow }}-${{ github.ref }}`, `cancel-in-progress: true` |
| Cancel on PRs only, never on main | `cancel-in-progress: ${{ github.event_name == 'pull_request' }}` |
| Unique group per main push | `group: ${{ github.workflow }}-${{ github.head_ref \|\| github.run_id }}` |
| Serialize deploys per environment | Job-level `group: deploy-${{ inputs.environment }}`, `cancel-in-progress: false` |
| Cross-repo lock | Same literal group name is **not** shared across repos; use an external lock or one deploy repo |

Rule: per group, one running and at most one pending. A newer pending run replaces the older pending one (it is cancelled).

---

## Permissions

```yaml
permissions:                 # workflow default
  contents: read
jobs:
  release:
    permissions:             # job override replaces, does not merge
      contents: write        # create release, push tags
      id-token: write        # OIDC
      packages: write        # GHCR
      attestations: write    # artifact attestations
      pull-requests: write   # PR comments
```

| Scope | Needed for |
|---|---|
| `contents` | checkout (read), push, tags, releases (write) |
| `pull-requests` | comment, label, review on PRs |
| `issues` | create/comment issues |
| `packages` | push to GHCR |
| `id-token` | request OIDC JWT (`write`) |
| `actions` | cancel runs, delete caches |
| `checks` / `statuses` | create check runs or commit statuses |
| `security-events` | upload SARIF to code scanning |
| `deployments` | create deployment records |
| `attestations` | `actions/attest-build-provenance` |

Shortcuts: `permissions: {}` (nothing), `read-all`, `write-all` (avoid). Fork PRs always get a read-only token. Events created by `GITHUB_TOKEN` do not trigger new runs (except `workflow_dispatch`, `repository_dispatch`); use a GitHub App token when they must.

---

## Secrets, variables and environments

| Task | Command |
|---|---|
| Repo secret | `gh secret set API_TOKEN --body "$VALUE"` |
| Secret from file | `gh secret set KUBECONFIG_B64 < kubeconfig.b64` |
| Environment secret | `gh secret set DB_PASSWORD --env prod --body "$VALUE"` |
| Org secret for selected repos | `gh secret set NPM_TOKEN --org ORG --repos "api,web"` |
| Variable | `gh variable set AWS_REGION --body ap-south-1` (`--env prod` for environment) |
| List | `gh secret list --env prod`, `gh variable list` |
| Create environment | `gh api -X PUT repos/OWNER/REPO/environments/prod` |
| Add wait timer | `gh api -X PUT repos/OWNER/REPO/environments/staging -F wait_timer=10` |

Precedence: environment > repository > organization. Secrets: 48 KB max each, masked in logs, not passed to fork PRs, cannot be used directly in `if:`. Environment protection: required reviewers (up to 6), prevent self-review, wait timer (up to 43,200 minutes), deployment branch and tag policies, custom protection rules.

---

## Reusable workflows vs composite actions

| | Reusable workflow | Composite action |
|---|---|---|
| Defined by | `on: workflow_call` in `.github/workflows/*.yml` | `action.yml` with `runs.using: composite` |
| Called at | Job level: `jobs.x.uses:` | Step level: `steps[*].uses:` |
| Unit of reuse | One or more whole jobs | A sequence of steps inside the caller's job |
| Own runner, `environment`, `permissions` | Yes | No (shares caller's runner, workspace, env) |
| Secrets | `secrets:` mapping or `secrets: inherit` | Only via inputs |
| Logs | Each job and step shown separately | Collapsed under one step |
| Inputs | Typed (`string`, `boolean`, `number`) | Always strings |
| Limits | 10 nested levels, 50 unique workflows per run | Composite can nest other actions |
| Use for | Standard pipelines, deploy with approvals | Shared setup, small building blocks |

```yaml
# .github/workflows/deploy.yml (callee)
on:
  workflow_call:
    inputs:
      environment: { type: string, required: true }
      image:       { type: string, required: true }
    secrets:
      SLACK_WEBHOOK: { required: false }
    outputs:
      url:
        value: ${{ jobs.deploy.outputs.url }}
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    permissions: { contents: read, id-token: write }
    outputs:
      url: ${{ steps.out.outputs.url }}
    steps:
      - id: out
        run: echo "url=https://${{ inputs.environment }}.example.com" >> "$GITHUB_OUTPUT"
```

```yaml
# caller
jobs:
  deploy-dev:
    uses: ./.github/workflows/deploy.yml              # or ORG/platform/.github/workflows/deploy.yml@<sha>
    permissions: { contents: read, id-token: write }  # callee can only reduce these
    with:
      environment: dev
      image: ${{ needs.build.outputs.image }}
    secrets: inherit
```

```yaml
# .github/actions/setup-app/action.yml (composite)
name: setup-app
inputs:
  node-version: { default: '22' }
outputs:
  cache-hit: { value: '${{ steps.node.outputs.cache-hit }}' }
runs:
  using: composite
  steps:
    - id: node
      uses: actions/setup-node@v4
      with: { node-version: '${{ inputs.node-version }}', cache: npm }
    - run: npm ci
      shell: bash                                      # required in composite run steps
```

Other action types: JavaScript (`runs.using: node24`, `main: dist/index.js`, bundle with `ncc`) and Docker (`runs.using: docker`, `image: Dockerfile`, Linux runners only).

---

## Caching

```yaml
- uses: actions/cache@v4
  id: cache
  with:
    path: |
      ~/.cache/pip
      ~/.cache/pre-commit
    key: ${{ runner.os }}-py-${{ hashFiles('**/requirements*.txt', '.pre-commit-config.yaml') }}
    restore-keys: |
      ${{ runner.os }}-py-
- if: steps.cache.outputs.cache-hit != 'true'
  run: pip install -r requirements.txt
```

| Tool | Built-in cache |
|---|---|
| Node | `actions/setup-node` `cache: npm \| yarn \| pnpm` |
| Python | `actions/setup-python` `cache: pip \| pipenv \| poetry` |
| Go | `actions/setup-go` (on by default, keyed on `go.sum`) |
| Java | `actions/setup-java` `cache: maven \| gradle \| sbt` |
| Docker | `cache-from: type=gha` / `cache-to: type=gha,mode=max` in `docker/build-push-action` |

| Rule | Detail |
|---|---|
| Save | Only when the exact `key` missed, at end of a successful job (`actions/cache/save` to control it) |
| Scope | Current branch, then PR base branch, then default branch; PR caches invisible to `main` |
| Quota | 10 GB per repo by default, LRU eviction, 7 days unused = evicted |
| Immutable | A key cannot be overwritten; bump the key (e.g. `-v2-`) to force refresh |
| Manage | `gh cache list`, `gh cache delete <key\|id>`, `gh cache delete --all` |

---

## Artifacts

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: report-${{ matrix.node }}       # names must be unique per run
    path: |
      reports/
      !reports/**/*.tmp
    retention-days: 7
    if-no-files-found: error
- uses: actions/download-artifact@v4
  with:
    pattern: report-*
    merge-multiple: true
    path: all-reports
```

| Fact | Detail |
|---|---|
| Purpose | Outputs of a run (binaries, reports, SBOMs), not dependency caches |
| v4 semantics | Immutable, available immediately after upload, unique names, no mixing with v3 |
| Retention | Default 90 days (configurable per repo/org, `retention-days` per upload) |
| Cross-run download | `actions/download-artifact@v4` with `run-id` and `github-token` |
| CLI | `gh run download <run-id> -n report-22` |

---

## OIDC to AWS

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/gha-deploy-prod
      aws-region: ap-south-1
      role-session-name: gha-${{ github.run_id }}
  - run: aws sts get-caller-identity
  - uses: aws-actions/amazon-ecr-login@v2
```

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:environment:prod"
      }
    }
  }]
}
```

| Job context | `sub` claim |
|---|---|
| Branch push / dispatch | `repo:ORG/REPO:ref:refs/heads/main` |
| Tag | `repo:ORG/REPO:ref:refs/tags/v1.2.0` (use `StringLike` with `refs/tags/v*`) |
| Pull request | `repo:ORG/REPO:pull_request` |
| Job with `environment` | `repo:ORG/REPO:environment:prod` |
| Custom template | e.g. `repo:ORG/REPO:environment:prod:job_workflow_ref:ORG/platform/.github/workflows/deploy.yml@refs/heads/main` |

Never trust only `aud`, and never `repo:ORG/*` for a prod role.

---

## OIDC to Azure

```bash
az ad app federated-credential create --id <APP_OBJECT_ID> --parameters '{
  "name": "gha-prod",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:ORG/REPO:environment:prod",
  "audiences": ["api://AzureADTokenExchange"]
}'
az role assignment create --assignee <CLIENT_ID> --role Contributor \
  --scope /subscriptions/<SUB_ID>/resourceGroups/rg-app-prod
```

```yaml
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    environment: prod
    runs-on: ubuntu-latest
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: az group show -n rg-app-prod -o table
```

Azure matches `subject` exactly (one federated credential per branch/environment; up to 20 per identity). IDs are not secrets, so `vars` is fine.

---

## OIDC to GCP

```bash
gcloud iam workload-identity-pools create github --location=global
gcloud iam workload-identity-pools providers create-oidc gha --location=global \
  --workload-identity-pool=github \
  --issuer-uri=https://token.actions.githubusercontent.com \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository_owner == 'ORG'"
gcloud iam service-accounts add-iam-policy-binding deployer@PROJECT.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github/attribute.repository/ORG/REPO"
```

```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github/providers/gha
    service_account: deployer@PROJECT.iam.gserviceaccount.com
```

---

## Security hardening

| Control | How |
|---|---|
| Least-privilege token | Repo default "read", `permissions: contents: read` at top, grant per job |
| Pin third-party actions | `uses: owner/action@<40-char-sha> # v1.2.3`; Dependabot `package-ecosystem: github-actions` |
| Restrict actions | Org/repo setting: GitHub-owned + verified + allow-list; optionally require SHA pinning |
| No script injection | Pass `github.event.*` (titles, bodies, branch names) via `env:` and quote `"$VAR"` |
| No pwn requests | Never check out PR head in `pull_request_target`; split into `pull_request` + `workflow_run` |
| Don't leave the token on disk | `actions/checkout` with `persist-credentials: false` |
| Keyless cloud access | OIDC with tight `sub`; per-environment roles |
| Gate secrets | Environment secrets + required reviewers + branch/tag policies |
| Protect workflow files | CODEOWNERS on `.github/`, rulesets requiring code-owner review |
| Self-hosted runners | Ephemeral/JIT only, never on public repos, runner groups restricted to repos |
| Fork PRs | "Require approval for first-time / all outside collaborators" |
| Provenance | `actions/attest-build-provenance`, verify with `gh attestation verify` |
| Lint and audit | `actionlint`, `zizmor .github/workflows` in CI |

```yaml
# injection-safe
- env:
    TITLE: ${{ github.event.pull_request.title }}
  run: echo "PR title: $TITLE"
```

---

## Runners

| Runner | Label examples | Notes |
|---|---|---|
| Hosted Linux | `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-22.04` | 1x minute multiplier, fresh VM per job |
| Hosted Linux arm64 | `ubuntu-24.04-arm` | Free for public repos |
| Hosted Windows / macOS | `windows-latest`, `macos-latest`, `macos-15` | 2x / 10x multipliers in private repos |
| Larger runners | Custom names, `runs-on: { group: big, labels: ubuntu-16core }` | Paid, static IPs, more CPU/RAM |
| Self-hosted | `[self-hosted, linux, x64, gpu]` | All labels must match |
| ARC scale set | `runs-on: arc-linux` (Helm release name) | Ephemeral pods, `minRunners`/`maxRunners` |

```bash
# self-hosted (ephemeral)
./config.sh --url https://github.com/ORG/REPO --token <REG_TOKEN> --labels gpu --ephemeral --unattended
gh api -X POST repos/ORG/REPO/actions/runners/registration-token --jq .token

# ARC
helm install arc -n arc-systems --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
helm install arc-linux -n arc-runners --create-namespace \
  --set githubConfigUrl=https://github.com/ORG --set githubConfigSecret=arc-github-app \
  --set minRunners=0 --set maxRunners=20 \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

---

## gh CLI

| Task | Command |
|---|---|
| List workflows | `gh workflow list` |
| Show a workflow's recent runs | `gh workflow view ci.yml` |
| Trigger dispatch | `gh workflow run deploy.yml --ref main -f environment=staging -f dry_run=false` |
| Inputs from JSON | `echo '{"environment":"prod"}' \| gh workflow run deploy.yml --json` |
| Enable / disable | `gh workflow enable ci.yml`, `gh workflow disable ci.yml` |
| List runs | `gh run list -w ci.yml -b main -L 20` (`--status failure`, `-e pull_request`) |
| Watch live | `gh run watch <id> --exit-status` |
| View summary / jobs | `gh run view <id>`, `gh run view <id> --verbose` |
| Failed logs only | `gh run view <id> --log-failed` |
| Full log of one job | `gh run view --job <job-id> --log` |
| Re-run | `gh run rerun <id>` / `--failed` / `--job <job-id>` / `--debug` |
| Cancel | `gh run cancel <id>` |
| Delete a run | `gh run delete <id>` |
| Download artifacts | `gh run download <id>` / `-n name` / `-p 'report-*'` |
| Caches | `gh cache list --sort size_in_bytes`, `gh cache delete --all` |
| Secrets / variables | `gh secret set NAME --env prod`, `gh variable list` |
| Verify attestation | `gh attestation verify oci://ghcr.io/org/app@sha256:... --owner org` |
| Raw API | `gh api repos/ORG/REPO/actions/runs --jq '.workflow_runs[0].status'` |
| Local extension | `gh extension install nektos/gh-act` then `gh act -l` |

```bash
# latest run on this branch, wait for it, fail the shell if it fails
gh run watch "$(gh run list -b "$(git branch --show-current)" -L1 --json databaseId --jq '.[0].databaseId')" --exit-status

# cancel all in-progress runs of a workflow
gh run list -w ci.yml -s in_progress --json databaseId --jq '.[].databaseId' | xargs -n1 gh run cancel

# conclusions of every job in a run
gh run view <id> --json jobs --jq '.jobs[] | "\(.name)\t\(.conclusion)"'
```

---

## Debugging

| Need | How |
|---|---|
| Verbose step logs | Re-run with "Enable debug logging", `gh run rerun <id> --debug`, or set secret/variable `ACTIONS_STEP_DEBUG=true` |
| Runner diagnostic logs | `ACTIONS_RUNNER_DEBUG=true` (adds runner/worker logs to the log archive) |
| See the event payload | `- run: cat "$GITHUB_EVENT_PATH" \| jq .` or `echo '${{ toJSON(github) }}'` via `env` |
| Why a step was skipped | Debug logs show `Evaluating condition for step` with the expanded expression |
| Workflow never started | File on wrong branch, filters (`branches` AND `paths`), YAML error (red banner in Actions tab), disabled workflow, `[skip ci]`, created by `GITHUB_TOKEN` |
| Stuck "Waiting for a runner" | Label mismatch, runner group not allowed for repo, concurrency limit reached, ARC listener down |
| Lint before pushing | `actionlint`, `zizmor .github/workflows` |
| Run locally | `act -j test` (nektos/act, approximation of hosted runners) |
| Interactive shell | `mxschmitt/action-tmate` (avoid on public repos; set `limit-access-to-actor: true`) |

---

## Common errors

| Error | Fix |
|---|---|
| `Resource not accessible by integration` | Add the missing `permissions` scope; fork PRs are read-only |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Trust policy `sub` does not match (environment vs ref form, casing, tag vs branch) |
| `Unable to get ACTIONS_ID_TOKEN_REQUEST_URL env variable` | Missing `id-token: write` (on the calling job for reusable workflows) |
| `AADSTS700213: No matching federated identity record found` | Azure federated credential subject does not match the token's `sub` |
| `Dependencies lock file is not found` | Commit the lockfile or drop `cache:` in setup-*; check `cache-dependency-path` |
| `Matrix vector 'x' does not contain any values` | Dynamic matrix got `[]`; guard the job with `if` |
| `Conflict: an artifact with this name already exists on the workflow run` | Unique artifact names per matrix leg, or `overwrite: true` |
| `The workflow is not valid. ... Unexpected value 'x'` | YAML key typo/indentation; run `actionlint` |
| `Branch "x" is not allowed to deploy to prod due to environment protection rules` | Add the branch/tag to the environment's deployment policy |
| `Skip output 'x' since it may contain secret` | Job outputs containing secrets are dropped; pass a reference, not the secret |
| `This request was automatically failed because there were no enabled runners online` | No matching self-hosted runner, or runner group excludes the repo |
| `Error: Process completed with exit code 1.` | Read the lines above it; add `set -x`; check the step's `shell` |
| Required check stuck "Expected — Waiting for status to be reported" | Workflow skipped by path filter or renamed job; use an always-running gate job |
| `refusing to allow a GitHub App to create or update workflow ... without workflows permission` | Token needs `workflows` scope (PAT) or `workflows: write` (App) to push `.github/workflows` changes |

---

## Top 15 interview one-liners

1. **Workflow, job, step, action, runner:** an event starts a workflow run; jobs run in parallel on separate runners unless `needs` orders them; steps run sequentially in one job and share the workspace; an action is a reusable step.
2. **`pull_request` vs `pull_request_target`:** `pull_request` runs the PR's code without secrets for forks; `pull_request_target` runs the base branch's workflow with secrets and a write token, so it must never execute PR code.
3. **`GITHUB_TOKEN`:** a per-job installation token that expires with the job, scoped by `permissions`; set `contents: read` by default and grant per job.
4. **Why a tag pushed by the workflow didn't trigger release:** events from `GITHUB_TOKEN` don't start new runs (loop prevention); use a GitHub App token.
5. **OIDC:** the job requests a signed JWT (`id-token: write`), the cloud validates issuer, audience and `sub`, and returns credentials for about an hour; no static keys in GitHub.
6. **The `sub` claim:** `repo:ORG/REPO:ref:refs/heads/main`, `...:environment:prod` or `...:pull_request`; the trust policy must pin it, never only `aud`.
7. **Script injection:** `${{ }}` is substituted into the script before the shell runs, so untrusted event data in `run:` is code execution; pass it via `env:`.
8. **Pin actions by SHA:** tags are mutable (tj-actions/changed-files, March 2025, showed it); a full commit SHA is immutable, and Dependabot keeps pins current.
9. **Cache vs artifact:** cache speeds up future runs (keyed, evictable, best effort); artifacts store a run's outputs (immutable in v4, 90-day default retention).
10. **Reusable workflow vs composite action:** reusable workflows share whole jobs with their own runners, environments and secrets; composite actions share steps inside the caller's job.
11. **Concurrency:** one running and one pending per group; cancel-in-progress for PRs, queue (no cancel) for deployments.
12. **Required checks with a matrix:** make a single `if: always()` gate job the required check, because skipped jobs count as passing and matrix names change.
13. **Environments:** they gate a job with reviewers, wait timers and branch policies, release environment secrets only after approval, and change the OIDC `sub`.
14. **Self-hosted runners:** treat them as untrusted, make them ephemeral (ARC scale sets or `--ephemeral`/JIT), restrict with runner groups, and never attach them to public repos.
15. **Debugging:** `gh run view --log-failed`, re-run with debug logging (`ACTIONS_STEP_DEBUG`), dump `toJSON(github)` through `env`, and lint with `actionlint` before pushing.

---

[⬅ Back to GitHub Actions](./README.md) · [🏠 Home](../README.md)

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
