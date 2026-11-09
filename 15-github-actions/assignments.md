# GitHub Actions — Assignments

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

> 25 practical assignments: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Do them in a real repository (a free public repo gives you unlimited hosted minutes and every feature used here). Reading YAML is not the same as debugging it.

[⬅ Back to GitHub Actions](./README.md) · [🏠 Home](../README.md)

## Index

| # | Level | Assignment |
|---|-------|------------|
| 1 | 🟢 | Your first workflow with branch and path filters |
| 2 | 🟢 | Drive runs from the terminal with `gh` |
| 3 | 🟢 | Node.js CI with the built-in dependency cache |
| 4 | 🟢 | Manual runs with typed `workflow_dispatch` inputs |
| 5 | 🟢 | Secrets vs configuration variables |
| 6 | 🟢 | Hand files between jobs with artifacts v4 |
| 7 | 🟢 | Scheduled health check with a job summary |
| 8 | 🟢 | Passing data: step outputs, job outputs, `GITHUB_ENV`, `GITHUB_PATH` |
| 9 | 🟢 | Conditions, status functions and failure handling |
| 10 | 🟢 | Readable logs and debug logging |
| 11 | 🟡 | Matrix with `include`/`exclude` and a single required gate |
| 12 | 🟡 | Concurrency: cancel stale PR runs, queue deployments |
| 13 | 🟡 | Least-privilege `GITHUB_TOKEN` and a PR comment bot |
| 14 | 🟡 | Environments with reviewers, wait timers and branch policies |
| 15 | 🟡 | Write a composite action |
| 16 | 🟡 | Write a reusable workflow with inputs, secrets and outputs |
| 17 | 🟡 | Caching done right: keys, restore-keys and BuildKit `type=gha` |
| 18 | 🟡 | OIDC to AWS: push to ECR with zero stored keys |
| 19 | 🟡 | Monorepo: dynamic matrix from changed paths |
| 20 | 🟡 | Build and publish images to GHCR with proper tags |
| 21 | 🔴 | Exploit and fix script injection and a `pull_request_target` pwn request |
| 22 | 🔴 | Build, test and release a JavaScript action |
| 23 | 🔴 | Self-hosted runners on Kubernetes with ARC scale sets |
| 24 | 🔴 | OIDC to Azure and GCP, and a custom `sub` claim |
| 25 | 🔴 | Supply-chain hardened release pipeline with org-level guardrails |

---

## 🟢 Beginner

### Assignment 1. Your first workflow with branch and path filters 🟢

**Objective:** Understand how an event is matched to a workflow file and when a workflow does not start.

**Tasks**
1. Create a repo with a `src/` folder, a `docs/` folder and `.github/workflows/ci.yml`.
2. Trigger on `push` to `main` and on `pull_request` targeting `main`.
3. Add `paths-ignore: ['docs/**', '**.md']` to both triggers.
4. Add one job on `ubuntu-latest` that checks out the code and prints `github.event_name`, `github.ref`, `github.sha` and `github.actor`.
5. Push a change to `docs/README.md` and confirm no run starts. Push a change to `src/` and confirm it does.
6. Open a PR from a feature branch and compare `github.ref` on the PR run with the push run.

**Expected outcome / acceptance criteria**
- A docs-only push creates no run; a code push creates one.
- You can explain why the PR run shows `refs/pull/<n>/merge` and why `github.sha` on a PR is the test merge commit, not your branch head.

<details>
<summary>Hints</summary>

```yaml
on:
  push:
    branches: [main]
    paths-ignore: ['docs/**', '**.md']
  pull_request:
    branches: [main]
    paths-ignore: ['docs/**', '**.md']
```

`branches` and `paths` are ANDed: both must match. The PR head commit is `github.event.pull_request.head.sha`.

</details>

**Bonus:** Make the job required in branch protection, then open a docs-only PR. Note that the PR is blocked ("Expected — Waiting for status to be reported") and write down two ways to fix that.

### Assignment 2. Drive runs from the terminal with `gh` 🟢

**Objective:** Inspect, re-run and download from workflow runs without the web UI.

**Tasks**
1. Install the GitHub CLI and run `gh auth login`, then `gh auth status` to confirm the `workflow` scope.
2. List the last 10 runs of `ci.yml` with `gh run list`.
3. Break a test on purpose and push. Use `gh run watch` to follow the run live.
4. Show only the failed step logs with `gh run view <id> --log-failed`.
5. Fix the test, then re-run just the failed jobs of the old run with `gh run rerun <id> --failed`.
6. Disable and re-enable the workflow with `gh workflow disable` / `gh workflow enable`.

**Expected outcome / acceptance criteria**
- A text file containing the exact commands you used and the run IDs.
- You can explain the difference between re-running failed jobs (same commit, same workflow file version) and pushing a new commit.

<details>
<summary>Hints</summary>

`gh run list --workflow ci.yml -L 10`, `gh run view <id> --json conclusion,jobs --jq '.jobs[] | {name, conclusion}'`, `gh run watch <id> --exit-status`. Re-runs use the workflow file and `GITHUB_SHA` of the original run.

</details>

**Bonus:** Write a one-liner that cancels every in-progress run of a workflow on a given branch.

### Assignment 3. Node.js CI with the built-in dependency cache 🟢

**Objective:** Build a fast, deterministic CI job for a real project.

**Tasks**
1. Add a small Node.js app with `package.json`, `package-lock.json`, an ESLint config and at least three tests (`node --test` or Jest).
2. Write a job with `actions/checkout@v4` and `actions/setup-node@v4` using `node-version: 22` and `cache: npm`.
3. Install with `npm ci`, then run `npm run lint` and `npm test`.
4. Run it twice and compare the "Post Run actions/setup-node" logs and install time between the two runs.
5. Add `timeout-minutes: 10` to the job.

**Expected outcome / acceptance criteria**
- The second run logs a cache hit and installs noticeably faster.
- You can explain why `npm ci` (not `npm install`) and why the cache key is derived from the lockfile.

<details>
<summary>Hints</summary>

The setup-node cache stores the npm cache directory (`~/.npm`), not `node_modules`, keyed on a hash of `package-lock.json`. Without a lockfile, `cache: npm` fails with "Dependencies lock file is not found".

</details>

**Bonus:** Read the Node version from `.nvmrc` with `node-version-file: .nvmrc` so local and CI versions cannot drift.

### Assignment 4. Manual runs with typed `workflow_dispatch` inputs 🟢

**Objective:** Build a safe, parameterized manual workflow.

**Tasks**
1. Create `ops.yml` with `workflow_dispatch` inputs: `environment` (choice: dev, staging, prod), `dry_run` (boolean, default true), `replicas` (number, default 2) and `reason` (string, required).
2. Print all inputs in the job, and print `DRY RUN` when `dry_run` is true.
3. Trigger it from the Actions tab.
4. Trigger it from the terminal with `gh workflow run ops.yml -f environment=staging -f dry_run=false -f replicas=3 -f reason="load test"`.
5. Try to trigger it on a branch where the file does not exist and record the error.

**Expected outcome / acceptance criteria**
- Both UI and CLI runs succeed with correct values.
- You can explain that the workflow must exist on the default branch to appear in the UI, and that `--ref` selects which branch's version runs.

<details>
<summary>Hints</summary>

Boolean inputs arrive in `inputs.dry_run` as a real boolean, so `if: inputs.dry_run` works; in `github.event.inputs` they are strings. Never interpolate `inputs.reason` directly into `run:`; pass it through `env:`.

</details>

**Bonus:** Add `run-name: Ops ${{ inputs.environment }} by @${{ github.actor }}` so the runs list is readable.

### Assignment 5. Secrets vs configuration variables 🟢

**Objective:** Store config at the right level and understand masking.

**Tasks**
1. Create a repo secret `API_TOKEN` and a repo variable `API_URL` with `gh secret set` and `gh variable set`.
2. Use them in a step through `env:` and call `curl -H "Authorization: Bearer $API_TOKEN" "$API_URL"` (use `https://httpbin.org/bearer`).
3. `echo "$API_TOKEN"` and confirm it shows as `***` in the log.
4. Print a base64-encoded version of the secret and observe that it is **not** masked. Then delete that run's logs.
5. Open a PR from a fork (or ask a friend) and observe that `API_TOKEN` is empty there.

**Expected outcome / acceptance criteria**
- Variable used for non-sensitive config, secret for credentials, and you can name the precedence order (environment > repository > organization).
- You can explain why derived values must be masked with `::add-mask::`.

<details>
<summary>Hints</summary>

`gh secret set API_TOKEN --body "abc123"`, `gh variable set API_URL --body https://httpbin.org/bearer`. Secrets are referenced as `${{ secrets.API_TOKEN }}`, variables as `${{ vars.API_URL }}`. Delete run logs from the run's "..." menu or `gh api -X DELETE repos/OWNER/REPO/actions/runs/<id>/logs`.

</details>

**Bonus:** Mask a value produced at runtime (for example a token returned by an API) with `echo "::add-mask::$TOKEN"` before anything else prints it.

### Assignment 6. Hand files between jobs with artifacts v4 🟢

**Objective:** Pass build output from one job to another and understand artifact scope.

**Tasks**
1. Job `build`: create a `dist/` folder with a file containing the commit SHA, then upload it with `actions/upload-artifact@v4` as `app-dist`, `retention-days: 3`.
2. Job `verify` with `needs: build`: download `app-dist` with `actions/download-artifact@v4` and check the file contents equal `github.sha`.
3. Add a matrix of 3 values to the build job that each uploads `report-<value>`, then download all of them into one folder in a later job.
4. Download the artifact locally with `gh run download`.

**Expected outcome / acceptance criteria**
- `verify` passes; the merged download contains three report files.
- You can explain why two matrix legs uploading the same artifact name fail in v4.

<details>
<summary>Hints</summary>

```yaml
- uses: actions/download-artifact@v4
  with:
    pattern: report-*
    merge-multiple: true
    path: reports
```

v4 artifacts are immutable: a second upload with the same name in a run errors with "an artifact with this name already exists" unless you use `overwrite: true`.

</details>

**Bonus:** Use the `artifact-id` output of upload-artifact and the REST API to fetch the artifact's size and expiry date.

### Assignment 7. Scheduled health check with a job summary 🟢

**Objective:** Use cron triggers and write a human-readable run summary.

**Tasks**
1. Create `health.yml` triggered by `schedule` (every 30 minutes, at minute 7 and 37) and by `workflow_dispatch`.
2. Curl three public URLs and record status code and response time with `curl -o /dev/null -s -w '%{http_code} %{time_total}'`.
3. Write a markdown table of results to `$GITHUB_STEP_SUMMARY`.
4. Fail the job if any URL does not return 200.
5. Check the run times in the runs list and note the delay vs the cron time.

**Expected outcome / acceptance criteria**
- A run summary page with a results table.
- You can explain that cron is UTC, runs from the default branch only, can be delayed under load, and is disabled after 60 days of no repo activity on public repos.

<details>
<summary>Hints</summary>

```yaml
on:
  schedule:
    - cron: '7,37 * * * *'
```

Avoid minute 0: the top of the hour is the busiest time for the scheduler.

</details>

**Bonus:** Open (or update) a GitHub issue automatically when the check fails, using `gh issue create` with `GH_TOKEN: ${{ github.token }}` and `issues: write`.

### Assignment 8. Passing data: step outputs, job outputs, `GITHUB_ENV`, `GITHUB_PATH` 🟢

**Objective:** Use every in-run data channel correctly.

**Tasks**
1. Step `meta` (with `id: meta`): compute a short SHA and a version from `package.json`, write both to `$GITHUB_OUTPUT`.
2. Use them in a later step as `steps.meta.outputs.short_sha`.
3. Write `BUILD_DATE` to `$GITHUB_ENV` and read it in a later step as `$BUILD_DATE`.
4. Create `./bin/hello` (an executable script), add `./bin` to `$GITHUB_PATH` and call `hello` in the next step.
5. Expose `version` as a job output and print it in a second job through `needs.<job>.outputs.version`.
6. Write a multiline value (the last 5 commit subjects) to `$GITHUB_OUTPUT` using a delimiter.

**Expected outcome / acceptance criteria**
- All values appear in later steps and in the second job.
- You can explain why `$GITHUB_ENV` changes are invisible to the step that writes them.

<details>
<summary>Hints</summary>

```bash
echo "short_sha=${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
{
  echo "log<<GHA_DELIM"
  git log -5 --format=%s
  echo "GHA_DELIM"
} >> "$GITHUB_OUTPUT"
```

Fetch history with `fetch-depth: 0` (or at least 5) or `git log -5` shows one commit.

</details>

**Bonus:** Try to output a secret as a job output and observe that GitHub skips it with a warning. Explain why.

### Assignment 9. Conditions, status functions and failure handling 🟢

**Objective:** Control what runs after a failure, a cancel or a skip.

**Tasks**
1. Create jobs `test` → `build` → `deploy` with `needs`.
2. Make `deploy` run only on pushes to `main`: `if: github.event_name == 'push' && github.ref == 'refs/heads/main'`.
3. Add a step at the end of `test` that runs `if: failure()` and prints diagnostic info, and one with `if: always()` that cleans up.
4. Add a `notify` job that `needs: [test, build, deploy]`, runs `if: always()`, and prints each needs result.
5. Make a flaky lint step `continue-on-error: true` and observe how the job conclusion and `steps.<id>.outcome` vs `conclusion` differ.
6. Cancel a run mid-step and see which steps still run.

**Expected outcome / acceptance criteria**
- A table of scenarios (pass, fail, cancel) and which jobs and steps ran in each.
- You can explain why `!cancelled()` is usually better than `always()` for post-processing.

<details>
<summary>Hints</summary>

`needs.test.result` is one of `success`, `failure`, `cancelled`, `skipped`. A job with `if` that contains no status function implicitly gets `success() &&` prepended.

</details>

**Bonus:** Add `timeout-minutes: 1` to a step running `sleep 120` and record what conclusion it produces.

### Assignment 10. Readable logs and debug logging 🟢

**Objective:** Make failures easy to diagnose.

**Tasks**
1. Use `::group::Install` / `::endgroup::` to fold noisy output.
2. Emit `::notice::`, `::warning file=src/app.js,line=3::` and `::error::` annotations and find them on the run summary and on the PR "Files changed" tab.
3. Re-run a job with "Enable debug logging" ticked and find the `##[debug]` lines.
4. Set a repo variable `ACTIONS_STEP_DEBUG=true` and confirm debug output on the next run, then remove it.
5. Use `set -euxo pipefail` in a multi-line `run` and compare the output with the default `bash -e {0}` shell.

**Expected outcome / acceptance criteria**
- A screenshot or notes showing grouped logs, annotations on a line of code, and debug output.
- You can name the difference between step debug (`ACTIONS_STEP_DEBUG`) and runner diagnostic logs (`ACTIONS_RUNNER_DEBUG`).

<details>
<summary>Hints</summary>

The default shell on Linux runners is `bash --noprofile --norc -eo pipefail {0}` only when you set `shell: bash` explicitly; the implicit default is `bash -e {0}`, which does not catch failures inside pipes. Set `defaults.run.shell: bash` at workflow level.

</details>

**Bonus:** Install `actionlint` locally and fix every issue it reports in your workflows.

---

## 🟡 Intermediate

### Assignment 11. Matrix with `include`/`exclude` and a single required gate 🟡

**Objective:** Test across versions and operating systems without breaking branch protection every time the matrix changes.

**Tasks**
1. Build a matrix: `os: [ubuntu-latest, windows-latest, macos-latest]`, `node: [20, 22, 24]`.
2. `exclude` Node 20 on macOS, and `include` one extra leg `{os: ubuntu-latest, node: 24, experimental: true}`.
3. Make the experimental leg non-blocking with `continue-on-error: ${{ matrix.experimental == true }}`.
4. Set `fail-fast: false` and `max-parallel: 4`.
5. Add a `ci-ok` job with `needs: [test]`, `if: always()`, that fails unless `needs.test.result == 'success'`.
6. Make only `ci-ok` a required status check, then add Node 26 to the matrix and confirm protection still works.

**Expected outcome / acceptance criteria**
- 9 legs run (8 from the product minus exclude, plus the include), visible with readable names.
- A failing leg makes `ci-ok` red and blocks merge; a skipped `test` job does not sneak through as green.

<details>
<summary>Hints</summary>

```yaml
ci-ok:
  needs: [test]
  if: always()
  runs-on: ubuntu-latest
  steps:
    - run: |
        [ "${{ needs.test.result }}" = "success" ] || { echo "test: ${{ needs.test.result }}"; exit 1; }
```

Without `if: always()`, a failed `test` makes `ci-ok` **skipped**, and a skipped required check counts as passing.

</details>

**Bonus:** Name each leg with `name: test (${{ matrix.os }}, node ${{ matrix.node }})` and compute the monthly minute cost difference between the macOS legs and the Linux legs in a private repo.

### Assignment 12. Concurrency: cancel stale PR runs, queue deployments 🟡

**Objective:** Save minutes on PRs and never run two deploys to the same environment at once.

**Tasks**
1. In `ci.yml`, add `concurrency: { group: ${{ github.workflow }}-${{ github.ref }}, cancel-in-progress: true }`.
2. Push three commits to a PR within 30 seconds and confirm only the last run completes.
3. Push two commits to `main` and confirm the main-branch runs are **not** cancelled by PR runs (different `github.ref`).
4. In a `deploy` job, set `concurrency: { group: deploy-${{ inputs.environment }}, cancel-in-progress: false }` and add a 60 second sleep.
5. Trigger the deploy four times quickly and record what happens to runs 2, 3 and 4.

**Expected outcome / acceptance criteria**
- PR runs cancel each other; deploys queue.
- You can explain that a group holds at most one running and one pending entry, so run 2 is replaced by run 3 and then by run 4 while run 1 is still going.

<details>
<summary>Hints</summary>

To cancel only on PRs but never on `main`: `cancel-in-progress: ${{ github.event_name == 'pull_request' }}`. Workflow-level and job-level concurrency can be combined.

</details>

**Bonus:** Use `github.head_ref || github.run_id` in the group so pushes to `main` each get a unique group and never cancel anything.

### Assignment 13. Least-privilege `GITHUB_TOKEN` and a PR comment bot 🟡

**Objective:** Run every workflow with the minimum token scope.

**Tasks**
1. Set the repo default workflow permissions to read-only (Settings > Actions > General, or `gh api -X PUT repos/OWNER/REPO/actions/permissions/workflow -f default_workflow_permissions=read`).
2. Add `permissions: contents: read` at the top of every workflow.
3. Add a `coverage-comment` job with `permissions: { contents: read, pull-requests: write }` that posts or updates a single PR comment with test coverage using `gh pr comment --edit-last` or `gh api`.
4. Remove `pull-requests: write` and record the exact error.
5. Show that a tag pushed by a step using `GITHUB_TOKEN` does **not** trigger your `on: push: tags` workflow.

**Expected outcome / acceptance criteria**
- Comment job works with exactly two permissions; the error text without them is `Resource not accessible by integration` (HTTP 403).
- You can explain why GitHub suppresses workflow runs for events created with `GITHUB_TOKEN`, and what to use instead (a GitHub App token via `actions/create-github-app-token`).

<details>
<summary>Hints</summary>

`gh` reads `GH_TOKEN`: set `env: GH_TOKEN: ${{ github.token }}`. Any permission not listed in a `permissions` block becomes `none`. On fork PRs the token is read-only regardless of what you request.

</details>

**Bonus:** Use `gh api repos/OWNER/REPO/actions/runs/<id>/jobs` and the "Set up job" log section to show the effective token permissions of a job.

### Assignment 14. Environments with reviewers, wait timers and branch policies 🟡

**Objective:** Gate deployments with GitHub environments, not with YAML tricks.

**Tasks**
1. Create environments `dev`, `staging`, `prod` (UI or `gh api -X PUT repos/OWNER/REPO/environments/prod`).
2. `staging`: 2 minute wait timer. `prod`: one required reviewer, "prevent self-review" on, deployment branches restricted to `main` and tags `v*`.
3. Add an environment secret `DB_PASSWORD` with different values per environment and an environment variable `APP_URL`.
4. Write a deploy workflow with three jobs chained by `needs`, each with `environment: { name: <env>, url: ${{ vars.APP_URL }} }`.
5. Try deploying prod from a feature branch with `workflow_dispatch` and record the error.
6. Approve the prod job from the UI and confirm the URL appears on the run graph and in the repo's Deployments page.

**Expected outcome / acceptance criteria**
- Each job sees only its own environment's secret.
- Prod cannot run from a non-main branch and cannot be self-approved.

<details>
<summary>Hints</summary>

The error is `Branch "feature/x" is not allowed to deploy to prod due to environment protection rules.` Environment secrets are only released to the job after all protection rules pass. Required reviewers on private repos need GitHub Pro/Team/Enterprise.

</details>

**Bonus:** Add a `deployment_status` or `gh api repos/OWNER/REPO/deployments` query that lists the last 5 prod deployments with their SHA and status.

### Assignment 15. Write a composite action 🟡

**Objective:** Remove copy-pasted setup steps from many workflows.

**Tasks**
1. Create `.github/actions/setup-app/action.yml` with `runs.using: composite`.
2. Inputs: `node-version` (default `22`), `install` (default `'true'`). Output: `cache-hit`.
3. Steps: `actions/setup-node@v4` with npm cache, then `npm ci` only when `install == 'true'`.
4. Every `run` step in the action must declare `shell: bash`.
5. Use it from two workflows with `uses: ./.github/actions/setup-app` after `actions/checkout`.
6. Move the action to a separate public repo, tag `v1.0.0` and `v1`, and consume it as `OWNER/setup-app@v1`.

**Expected outcome / acceptance criteria**
- Both workflows shrink and behave the same; the action's output is readable by the caller.
- You can explain why a local action needs checkout first, and why composite steps share the caller's workspace and env.

<details>
<summary>Hints</summary>

```yaml
outputs:
  cache-hit:
    value: ${{ steps.node.outputs.cache-hit }}
runs:
  using: composite
  steps:
    - id: node
      uses: actions/setup-node@v4
      with: { node-version: '${{ inputs.node-version }}', cache: npm }
    - if: inputs.install == 'true'
      run: npm ci
      shell: bash
```

Composite inputs are always strings.

</details>

**Bonus:** Add a `branding` block and an `author`, and write a README with a usage example, as you would for the Marketplace.

### Assignment 16. Write a reusable workflow with inputs, secrets and outputs 🟡

**Objective:** Share whole jobs (runner, environment, permissions) across repositories.

**Tasks**
1. Create `.github/workflows/build.yml` with `on: workflow_call`, inputs `node-version` (string) and `run-e2e` (boolean), secret `NPM_TOKEN` (required: false), output `artifact-name`.
2. Inside, run lint, test, build, upload an artifact, and set the output from a job output.
3. Call it from `ci.yml` at **job level**: `jobs.build.uses: ./.github/workflows/build.yml` with `with:` and `secrets:`.
4. Read `needs.build.outputs.artifact-name` in a following job and download the artifact.
5. Call the same workflow from a second repository as `OWNER/REPO/.github/workflows/build.yml@v1`.
6. Switch to `secrets: inherit` and explain the trade-off.

**Expected outcome / acceptance criteria**
- Two callers use one implementation; outputs flow back to the caller.
- You can state the limits: reusable workflows nest up to 10 levels total, a run can call at most 50 unique reusable workflows, and `env` from the caller is not propagated.

<details>
<summary>Hints</summary>

Outputs are mapped twice: `on.workflow_call.outputs.artifact-name.value: ${{ jobs.build.outputs.name }}`. A job that calls a reusable workflow cannot have `steps` or `runs-on`. Permissions in the called workflow can only reduce what the caller grants.

</details>

**Bonus:** Use a matrix in the caller to call the reusable workflow three times with different inputs.

### Assignment 17. Caching done right: keys, restore-keys and BuildKit `type=gha` 🟡

**Objective:** Get real cache hits, and measure them.

**Tasks**
1. Cache `~/.cache/pip` (or Go's module cache) with `actions/cache@v4`, key `${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}` and a `restore-keys` prefix.
2. Change one dependency and confirm a partial restore followed by a save under the new key.
3. Build a Docker image with `docker/setup-buildx-action@v3` and `docker/build-push-action@v6` using `cache-from: type=gha` and `cache-to: type=gha,mode=max`.
4. Record build time cold, warm, and after changing only application code.
5. List and delete caches with `gh cache list` and `gh cache delete`.
6. Explain from a PR run why it can read `main`'s caches but `main` cannot read the PR's.

**Expected outcome / acceptance criteria**
- A table of timings before and after caching.
- Correct explanation of cache scope (branch, then base branch, then default branch) and eviction (7 days unused, 10 GB per repo by default).

<details>
<summary>Hints</summary>

`actions/cache` saves only when the primary key missed. Use `actions/cache/restore` and `actions/cache/save` separately when you need to save even on failure. In a Dockerfile, copy lockfiles and install dependencies before `COPY . .` or every code change busts the layer cache.

</details>

**Bonus:** Write a workflow that deletes caches belonging to a PR branch when the PR is closed (`on: pull_request: types: [closed]`, `actions: write`).

### Assignment 18. OIDC to AWS: push to ECR with zero stored keys 🟡

**Objective:** Replace `AWS_ACCESS_KEY_ID` secrets with federated, short-lived credentials.

**Tasks**
1. Create the IAM OIDC provider for `https://token.actions.githubusercontent.com` with audience `sts.amazonaws.com`.
2. Create a role `gha-ecr-push` whose trust policy requires `aud = sts.amazonaws.com` and `sub = repo:OWNER/REPO:ref:refs/heads/main`.
3. Attach a policy allowing ECR push to one repository only, plus `ecr:GetAuthorizationToken`.
4. In the workflow grant `permissions: { id-token: write, contents: read }`, use `aws-actions/configure-aws-credentials@v4` with `role-to-assume`, then `aws-actions/amazon-ecr-login@v2`, then build and push.
5. Run `aws sts get-caller-identity` in the job and confirm the assumed-role ARN.
6. Run the same workflow from a feature branch and record the error.

**Expected outcome / acceptance criteria**
- Image pushed from `main`; feature branch denied with `Not authorized to perform sts:AssumeRoleWithWebIdentity`.
- No AWS secrets exist in the repo (`gh secret list` shows none).

<details>
<summary>Hints</summary>

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:ref:refs/heads/main"
  }
}
```

A job with `environment:` sends `sub = repo:OWNER/REPO:environment:<name>` instead of the ref form.

</details>

**Bonus:** Add a step that decodes the OIDC token's claims (request it from `$ACTIONS_ID_TOKEN_REQUEST_URL` with `$ACTIONS_ID_TOKEN_REQUEST_TOKEN`) and prints `sub`, `aud`, `ref` and `job_workflow_ref`, never the raw token.

### Assignment 19. Monorepo: dynamic matrix from changed paths 🟡

**Objective:** Build and test only the services that changed, with one required check.

**Tasks**
1. Create `services/api`, `services/web`, `services/worker`, each with its own tests.
2. Job `changes`: compute the list of changed service folders vs the PR base (`git diff --name-only` with `fetch-depth: 0`) and output a JSON array.
3. Job `test` with `needs: changes`, `if: needs.changes.outputs.services != '[]'`, and `strategy.matrix.service: ${{ fromJSON(needs.changes.outputs.services) }}`.
4. Run the tests inside `services/${{ matrix.service }}` using `defaults.run.working-directory`.
5. Add a `ci-ok` gate (as in Assignment 11) that also passes when `test` was skipped because nothing changed.
6. Open PRs touching zero, one and all three services.

**Expected outcome / acceptance criteria**
- PRs run exactly the matrix legs for changed services; a docs-only PR is green through `ci-ok`.
- A change to a shared folder (`libs/`) triggers all services.

<details>
<summary>Hints</summary>

```bash
base=${{ github.event.pull_request.base.sha }}
svcs=$(git diff --name-only "$base" HEAD | awk -F/ '$1=="services"{print $2}' | sort -u | jq -R . | jq -cs .)
echo "services=$svcs" >> "$GITHUB_OUTPUT"
```

An empty array in `fromJSON` for a matrix fails with "Matrix vector does not contain any values", which is why the `if` guard is needed. `dorny/paths-filter` is a popular alternative.

</details>

**Bonus:** Make the `ci-ok` gate accept `success` or `skipped` for `test` but nothing else.

### Assignment 20. Build and publish images to GHCR with proper tags 🟡

**Objective:** Produce traceable images with branch, SHA and semver tags.

**Tasks**
1. Grant `packages: write` and log in to `ghcr.io` with `docker/login-action@v3` using `${{ github.actor }}` and `${{ secrets.GITHUB_TOKEN }}`.
2. Use `docker/metadata-action@v5` to generate tags: `type=sha`, `type=ref,event=branch`, `type=semver,pattern={{version}}`, `type=semver,pattern={{major}}.{{minor}}`, and OCI labels.
3. Build multi-arch (`linux/amd64,linux/arm64`) with `docker/setup-qemu-action@v3` and buildx, and push only on `main` and tags (`push: ${{ github.event_name != 'pull_request' }}`).
4. Tag `v1.2.0` and confirm images `1.2.0`, `1.2` and `latest` exist and share one digest.
5. Pull the image and check `org.opencontainers.image.revision` in `docker inspect`.

**Expected outcome / acceptance criteria**
- Package visible under the repo with linked source; PRs build but never push.
- You can explain why deployments should reference `@sha256:` digests, not `latest`.

<details>
<summary>Hints</summary>

The `GITHUB_TOKEN` can push to GHCR packages linked to the repo. If the package already exists and was created elsewhere, grant the repo access in the package settings, or you get `denied: permission_denied: write_package`.

</details>

**Bonus:** Replace QEMU emulation with native `ubuntu-24.04-arm` runners in a matrix and merge the per-arch images into one manifest with `docker buildx imagetools create`.

---

## 🔴 Advanced

### Assignment 21. Exploit and fix script injection and a `pull_request_target` pwn request 🔴

**Objective:** Prove, in a throwaway repo, why two common patterns leak secrets, then fix them.

> Do this only in a disposable public repository you own, with a fake secret (`FAKE_TOKEN=not-a-real-secret`). Never test against other people's repositories.

**Tasks**
1. **Injection:** write a workflow on `issues: [opened]` with `run: echo "New issue: ${{ github.event.issue.title }}"`.
2. Open an issue titled `x"; echo "INJECTED $FAKE_TOKEN" | base64; echo "` with `FAKE_TOKEN` set in the job `env` from a secret. Show that your command runs and the base64 value is printed unmasked.
3. Fix it by passing the title through `env: TITLE: ${{ github.event.issue.title }}` and using `"$TITLE"`. Re-run the attack and show it is inert.
4. **Pwn request:** write a workflow on `pull_request_target` that checks out `github.event.pull_request.head.sha` and runs `npm test` with a secret in `env`.
5. From a second account (or a fork) open a PR that changes the `test` script to print the secret base64-encoded. Show the leak.
6. Fix it: split into an unprivileged `pull_request` workflow that runs the code and uploads results as an artifact, and a `workflow_run` workflow that only reads that artifact as data (never executes it) and posts a comment.
7. Run `zizmor` and `actionlint` on the before and after versions.

**Expected outcome / acceptance criteria**
- A short write-up with the four YAML files (vulnerable and fixed) and the scanner output.
- You can explain why `pull_request_target` runs the **base** branch's workflow with secrets and a write token, and why checking out the head turns that into code execution.

<details>
<summary>Hints</summary>

Expressions are substituted into the script text before bash ever sees it, so quoting in YAML does not help. In the `workflow_run` consumer, treat the artifact like hostile input: validate it is a number or a fixed format, never `source` it or interpolate it into `run:`. Also set `persist-credentials: false` on checkout so the token is not left in `.git/config`.

</details>

**Bonus:** Add a "Require approval for all outside collaborators" setting for fork PR workflows, and explain why it does not protect `pull_request_target`.

### Assignment 22. Build, test and release a JavaScript action 🔴

**Objective:** Ship a production-quality action that others consume by tag and SHA.

**Tasks**
1. Create a repo with `action.yml` (`runs.using: node24`, `main: dist/index.js`), inputs `token` and `label`, output `pr-number`.
2. Implement it in `src/index.js` with `@actions/core` and `@actions/github`: find the PR for the current commit and add the label.
3. Unit test it with Jest or `node --test`, mocking `@actions/github`.
4. Bundle with `@vercel/ncc build src/index.js -o dist --license licenses.txt` and commit `dist/`.
5. Add a CI job that rebuilds `dist/` and fails if `git diff --exit-code dist/` shows changes.
6. Add an integration test job that uses the action from the same repo with `uses: ./`.
7. Release `v1.0.0`, move the major tag `v1` to it, and consume it from another repo both as `@v1` and pinned to the full commit SHA with a `# v1.0.0` comment.

**Expected outcome / acceptance criteria**
- Tests, dist check and integration test all green; the action labels a real PR.
- You can explain why `node_modules` is not committed, why `dist/` is, and why consumers should pin by SHA instead of `v1`.

<details>
<summary>Hints</summary>

`core.getInput('token', { required: true })`, `core.setOutput('pr-number', n)`, `core.setFailed(err.message)`. Use `github.getOctokit(token).rest.repos.listPullRequestsAssociatedWithCommit`. Move the major tag with `git tag -fa v1 -m v1.0.0 && git push origin v1 --force`.

</details>

**Bonus:** Convert it to TypeScript, add a Dependabot config for npm and GitHub Actions, and publish it to the Marketplace.

### Assignment 23. Self-hosted runners on Kubernetes with ARC scale sets 🔴

**Objective:** Run ephemeral, autoscaling runners in your own cluster.

**Tasks**
1. Create a kind or EKS cluster and a GitHub App (or a fine-grained PAT) for runner registration.
2. Install the controller: `helm install arc --namespace arc-systems --create-namespace oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller`.
3. Store the App credentials in a Kubernetes secret in `arc-runners`.
4. Install a scale set named `arc-linux` with `githubConfigUrl` pointing at your repo or org, `minRunners: 0`, `maxRunners: 5`.
5. Run a workflow with `runs-on: arc-linux` and a 10-leg matrix; watch pods with `kubectl get pods -n arc-runners -w`.
6. Configure `containerMode.type: dind` (or `kubernetes` with a work volume) so jobs can build containers, and set CPU/memory requests in the runner pod template.
7. Restrict the scale set to one runner group that only trusted repos can use.

**Expected outcome / acceptance criteria**
- Pods scale from 0 to 5 under load, each runner takes one job then is deleted, and scale back to 0.
- A short note on why ephemeral runners matter for security and why public repos should never use self-hosted runners.

<details>
<summary>Hints</summary>

```bash
helm install arc-linux --namespace arc-runners --create-namespace \
  --set githubConfigUrl="https://github.com/OWNER/REPO" \
  --set githubConfigSecret=arc-github-app \
  --set minRunners=0 --set maxRunners=5 \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

The Helm release name becomes the `runs-on` label. Check `kubectl logs -n arc-systems deploy/arc-gha-rs-controller` and the listener pod when jobs stay queued.

</details>

**Bonus:** Give the runner pods an IAM role through EKS Pod Identity or IRSA and compare that approach with OIDC from the workflow.

### Assignment 24. OIDC to Azure and GCP, and a custom `sub` claim 🔴

**Objective:** Use keyless federation on two more clouds and tighten what the token proves.

**Tasks**
1. **Azure:** create an app registration (or a user-assigned managed identity), add a federated credential with issuer `https://token.actions.githubusercontent.com`, subject `repo:OWNER/REPO:environment:prod`, audience `api://AzureADTokenExchange`, and grant it a role on one resource group.
2. In the workflow use `azure/login@v2` with `client-id`, `tenant-id`, `subscription-id` (no `creds` secret) and run `az group show`.
3. **GCP:** create a Workload Identity Pool and an OIDC provider with an `attribute-condition` that restricts `assertion.repository_owner` and `assertion.repository`.
4. Bind a service account with `roles/iam.workloadIdentityUser` to the `principalSet` for your repository, then use `google-github-actions/auth@v2` with `workload_identity_provider` and `service_account` and run `gcloud storage ls`.
5. Customize the repo's OIDC `sub` claim template to include `job_workflow_ref`, then update the AWS or Azure trust to require that only your central `deploy.yml` reusable workflow can assume the role.
6. Prove a different workflow in the same repo is now denied.

**Expected outcome / acceptance criteria**
- Both clouds authenticated without any long-lived secret.
- A table of the `sub` value per trigger (branch push, tag, PR, environment, custom template) and which trust policy accepts it.

<details>
<summary>Hints</summary>

```bash
gh api -X PUT repos/OWNER/REPO/actions/oidc/customization/sub \
  --input - <<<'{"use_default":false,"include_claim_keys":["repo","context","job_workflow_ref"]}'
```

Azure federated credentials match the subject exactly (no wildcards), so create one per environment or branch, or use flexible federated identity credentials. Azure needs `id-token: write` too.

</details>

**Bonus:** Write Terraform for all three cloud trusts and run it from a GitHub Actions workflow that itself uses OIDC.

### Assignment 25. Supply-chain hardened release pipeline with org-level guardrails 🔴

**Objective:** Combine everything into a release process an auditor would sign off.

**Tasks**
1. Pin every third-party action to a full 40-character commit SHA with a version comment; add `.github/dependabot.yml` with `package-ecosystem: github-actions` (weekly, grouped).
2. Add `CODEOWNERS` for `.github/` and require code owner review in a ruleset on `main`.
3. Build the image once on `main`, generate an SBOM (`anchore/sbom-action` or `trivy image --format cyclonedx`), and create provenance with `actions/attest-build-provenance@v2` (`id-token: write`, `attestations: write`).
4. Release with release-please (or semantic-release) using a GitHub App token, so the tag it creates **does** trigger the release workflow.
5. The release workflow promotes the existing image by digest (no rebuild), publishes a GitHub Release with the SBOM attached, and deploys to a `prod` environment with a required reviewer.
6. Verify the attestation from your laptop with `gh attestation verify oci://IMAGE@sha256:... --owner OWNER`.
7. At org or repo level: allow only GitHub-owned, verified-creator and listed actions; turn on "Require actions to be pinned to a full-length commit SHA" if available to you; protect `v*` tags with a tag ruleset.

**Expected outcome / acceptance criteria**
- A diagram (Mermaid) of the pipeline and a list of each control and the threat it addresses.
- `gh attestation verify` succeeds for the released digest and fails for an image built on your laptop.
- `zizmor` reports no high-severity findings.

<details>
<summary>Hints</summary>

```yaml
- uses: actions/create-github-app-token@v1
  id: app
  with:
    app-id: ${{ vars.RELEASE_APP_ID }}
    private-key: ${{ secrets.RELEASE_APP_KEY }}
```

Pass `${{ steps.app.outputs.token }}` to release-please. Tags created with `GITHUB_TOKEN` would not start the release workflow. For in-cluster enforcement, Kyverno `verifyImages` or Sigstore policy-controller can require the attestation.

</details>

**Bonus:** Make the whole CI workflow a required workflow through an org ruleset so every repository runs it, and measure the change in median PR-to-merge time.

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
